# yt-dlp 下载限速机制深度分析

## 概述

yt-dlp 的下载限速机制控制点分布在多个模块中，三个核心机制紧密协作：

1. **全局限速（ratelimit）**：通过 sleep 控制单连接下载速度
2. **单任务节流检测（throttledratelimit）**：检测服务器端限流，触发重定向
3. **进度反馈系统**：计算速度/ETA，为前两者提供数据支撑，同时向用户展示进度

本文详细梳理三者的关联、具体实现及可能的误判控制点。

---

## 一、限速、节流与进度的关联全景

### 1.1 核心协作链路

```
用户参数 (ratelimit, throttledratelimit)
        │
        ▼
┌─────────────────────────────────────────────────┐
│          FileDownloader 基类                     │
│  ┌─────────────┐   ┌────────────────────────┐  │
│  │ slow_down() │   │ report_progress()      │  │
│  │  (限速)     │   │  (进度展示)            │  │
│  └──────┬──────┘   └───────────┬────────────┘  │
└─────────┼──────────────────────┼───────────────┘
          │                      │
          ▼                      ▼
┌──────────────────┐   ┌───────────────────────┐
│  HttpFD (单文件) │   │ FragmentFD (分片)      │
│  calc_speed()    │   │ ProgressCalculator    │
│  calc_eta()      │   │ SmoothValue           │
│  节流检测逻辑    │   │ 分片进度聚合          │
└─────────┬────────┘   └──────────┬────────────┘
          │                       │
          └───────────┬───────────┘
                      │
                      ▼
              throttledratelimit 检测
              (速度 < 阈值 持续 3s → ThrottledDownload)
```

### 1.2 下载循环中的调用时序

位置：[downloader/http.py#L248-L326](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L248-L326)

```
while True:
    1. data_block = ctx.data.read(block_size)   # 读取数据块
    2. byte_counter += len(data_block)           # 更新计数器
    3. ctx.stream.write(data_block)              # 写入文件
    4. slow_down(start, now, byte_counter - ctx.resume_len)  # 限速
    5. now = time.time(); after = now            # 更新时间戳
    6. block_size = best_block_size(...)         # 动态调整块大小
    7. speed = calc_speed(start, now, ...)       # 计算速度
    8. eta = calc_eta(start, now, ...)           # 计算 ETA
    9. _hook_progress({status, speed, eta, ...}) # 触发进度钩子
    10. if speed < throttledratelimit:           # 节流检测
         - 持续 3s → 抛 ThrottledDownload
```

---

## 二、普通下载速度的计算方法

### 2.1 速度计算核心函数

**calc_speed()** — 简单平均速度
位置：[downloader/common.py#L160-L165](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L160-L165)

```python
@staticmethod
def calc_speed(start, now, bytes):
    dif = now - start
    if bytes == 0 or dif < 0.001:  # One millisecond
        return None
    return float(bytes) / dif
```

**关键点：**
- 计算的是 **从下载开始到当前时刻的全程平均速度**
- 不是瞬时速度，也不是滑动窗口
- `dif < 0.001` 保护：避免开始 1ms 内除零错误
- 返回 `None` 表示速度不可用（刚开始下载时）

**调用位置：** [downloader/http.py#L294](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L294)
```python
speed = self.calc_speed(start, now, byte_counter - ctx.resume_len)
```

### 2.2 ETA 计算核心函数

**calc_eta()** — 基于当前速度的剩余时间估算
位置：[downloader/common.py#L144-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L144-L158)

```python
@classmethod
def calc_eta(cls, start_or_rate, now_or_remaining, total=NO_DEFAULT, current=NO_DEFAULT):
    if total is NO_DEFAULT:
        # 调用方式 1: calc_eta(rate, remaining)
        rate, remaining = start_or_rate, now_or_remaining
        if None in (rate, remaining):
            return None
        return int(float(remaining) / rate)

    # 调用方式 2: calc_eta(start, now, total, current)
    start, now = start_or_rate, now_or_remaining
    if total is None:
        return None
    if now is None:
        now = time.time()
    rate = cls.calc_speed(start, now, current)
    return rate and int((float(total) - float(current)) / rate)
```

**调用位置：** [downloader/http.py#L295-L298](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L295-L298)
```python
if ctx.data_len is None:
    eta = None
else:
    eta = self.calc_eta(start, time.time(), ctx.data_len - ctx.resume_len, byte_counter - ctx.resume_len)
```

### 2.3 速度计算中的变量说明

| 变量 | 含义 | 说明 |
|------|------|------|
| `start` | 当前连接的下载开始时间 | 每次重试会重置 |
| `now` | 当前时间戳 | 每次循环更新 |
| `byte_counter` | 总已下载字节数 | 包含 resume_len |
| `ctx.resume_len` | 断点续传的已下载字节数 | 计算速度时要减去，只算本次连接的下载量 |
| `ctx.data_len` | 文件总大小 | 从 Content-Length 获取，可能为 None |

**公式：**
```
speed = (byte_counter - resume_len) / (now - start)
eta = (data_len - byte_counter) / speed   # 当 data_len 存在时
```

### 2.4 动态块大小调整

**best_block_size()** — 根据当前速度动态调整读取块大小
位置：[downloader/common.py#L181-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L181-L192)

```python
@staticmethod
def best_block_size(elapsed_time, bytes):
    new_min = max(bytes / 2.0, 1.0)
    new_max = min(max(bytes * 2.0, 1.0), 4194304)  # Do not surpass 4 MB
    if elapsed_time < 0.001:
        return int(new_max)
    rate = bytes / elapsed_time
    if rate > new_max:
        return int(new_max)
    if rate < new_min:
        return int(new_min)
    return int(rate)
```

**调整策略：**
- 目标：让每次 `read()` 调用约耗时 1 秒
- 块大小范围：`[max(bytes/2, 1), min(bytes*2, 4MB)]`
- 如果速度极快（elapsed < 1ms），直接用最大值 4MB
- 这样既避免频繁系统调用，又保证限速和进度更新的精度

---

## 三、分片聚合的展示方式

分片下载（HLS/DASH）使用 `ProgressCalculator` 进行更精确的速度计算和进度聚合。

### 3.1 ProgressCalculator 滑动窗口算法

位置：[utils/progress.py#L8-L94](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L8-L94)

**核心参数：**

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `SAMPLING_WINDOW` | 3 秒 | 速度计算窗口，只保留最近 3 秒的采样 |
| `SAMPLING_RATE` | 0.05 秒 | 最快采样间隔，避免更新过于频繁 |
| `GRACE_PERIOD` | 1 秒 | 启动宽限期，1 秒后才显示 ETA |

**滑动窗口实现：** [utils/progress.py#L74-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L74-L93)

```python
self._times.append(current_time)
self._downloaded.append(self.downloaded)

# 清理窗口外的旧数据
offset = bisect.bisect_left(self._times, current_time - self.SAMPLING_WINDOW)
del self._times[:offset]
del self._downloaded[:offset]

# 窗口内数据点不足 2 个时，速度不可用
if len(self._times) < 2:
    self.speed.reset()
    self.eta.reset()
    return

# 用窗口首尾两点计算平均速度
download_time = current_time - self._times[0]
self.speed.set((self.downloaded - self._downloaded[0]) / download_time)
```

**为什么用 3 秒窗口？**
- 太短：速度波动大，ETA 频繁跳动
- 太长：反应迟钝，真实速度变化很久后才体现
- 3 秒是经验值，平衡了稳定性和灵敏度

### 3.2 SmoothValue 指数平滑器

位置：[utils/progress.py#L96-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L96-L109)

```python
class SmoothValue:
    def set(self, value):
        self.value = value
        if self.smooth is None:
            self.smooth = self.value
        else:
            self.smooth = (1 - self._smoothing) * value + self._smoothing * self.smooth
```

**平滑公式：** `平滑值 = (1 - α) × 新值 + α × 旧平滑值`

| 用途 | α 值 | 效果 |
|------|------|------|
| speed | 0.7 | 中等平滑，展示更稳定 |
| eta | 0.9 | 强平滑，避免 ETA 大幅跳动 |

**初始化：** [utils/progress.py#L21-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L21-L22)
```python
self.speed = SmoothValue(0, smoothing=0.7)
self.eta = SmoothValue(None, smoothing=0.9)
```

### 3.3 多线程支持

位置：[utils/progress.py#L46-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L46-L60)

```python
def thread_reset(self):
    current_thread = threading.get_ident()
    with self._lock:
        self._thread_sizes[current_thread] = 0

def update(self, size):
    current_thread = threading.get_ident()
    with self._lock:
        last_size = self._thread_sizes.get(current_thread, 0)
        self._thread_sizes[current_thread] = size
        self._update(size - last_size)  # 只算增量
```

**设计说明：**
- `_thread_sizes` 字典按线程 ID 记录各自的已下载大小
- 每次 `update()` 传入的是该线程的 **累计** 大小，通过差值计算增量
- `thread_reset()` 在分片切换时调用，重置该线程的累计值
- `_lock` 保证多线程并发更新时的数据一致性

### 3.4 分片进度聚合逻辑

位置：[downloader/fragment.py#L242-L279](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py#L242-L279)

```python
def frag_progress_hook(s):
    # 只处理 downloading 和 finished 状态
    if s['status'] not in ('downloading', 'finished'):
        return

    # ctx_id 校验：确保是当前上下文的进度事件
    if ctx_id is not None and s.get('ctx_id') != ctx_id:
        return

    # 非直播场景：估算总大小
    if not ctx['live']:
        # 按当前已完成分片的平均大小估算总大小
        estimated_size = (
            (ctx['complete_frags_downloaded_bytes'] + frag_total_bytes)
            / (state['fragment_index'] + 1) * total_frags)
        progress.total = estimated_size
        progress.update(s.get('downloaded_bytes'))
        state['total_bytes_estimate'] = progress.total
    else:
        progress.update(s.get('downloaded_bytes'))

    # 分片下载完成时
    if s['status'] == 'finished':
        state['fragment_index'] += 1
        progress.thread_reset()  # 重置线程累计值

    # 聚合后的状态
    state['downloaded_bytes'] = progress.downloaded
    state['speed'] = progress.speed.smooth   # 使用平滑后的值
    state['eta'] = progress.eta.smooth       # 使用平滑后的值

    self._hook_progress(state, info_dict)
```

**展示的数据流向：**
```
子下载器 (HttpQuietDownloader)
    │  (每个分片独立下载，noprogress=True)
    ▼
frag_progress_hook (分片聚合钩子)
    │  - 收集子下载器的进度事件
    │  - ProgressCalculator 汇总速度
    │  - 按平均分片大小估算总大小
    ▼
self._hook_progress (对外统一接口)
    │
    ▼
report_progress (进度条显示)
```

### 3.5 分片与非分片的速度计算对比

| 维度 | HttpFD (非分片) | FragmentFD (分片) |
|------|-----------------|-------------------|
| 速度算法 | 全程平均速度 | 3 秒滑动窗口 + 指数平滑 |
| 精度 | 低（随时间推移越来越"滞后"） | 高（反映最近 3 秒的真实速度） |
| ETA 计算 | 直接用平均速度估算 | 用平滑后的速度估算 |
| 多线程支持 | 无（单连接） | 有（_thread_sizes 字典） |
| 代码位置 | [common.py#L160-L165](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L160-L165) | [progress.py#L89](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L89) |

**为什么分片需要更复杂的算法？**
- 分片下载涉及多个 HTTP 连接，每个连接有自己的启动、慢启动过程
- 分片切换时会有短暂停顿，简单平均会被拉低
- 并发分片下载时多个线程同时下载，需要汇总
- 直播场景没有固定总大小，需要动态估算

---

## 四、外部下载器参数的转发机制

当使用 `--external-downloader` 时，yt-dlp 不自己下载，而是将参数转发给外部下载器。

### 4.1 类层次结构

```
FileDownloader
    └── FragmentFD
            └── ExternalFD
                    ├── CurlFD
                    ├── WgetFD
                    ├── Aria2cFD
                    ├── AxelFD
                    ├── HttpieFD
                    └── FFmpegFD
```

**注意：** `ExternalFD` 继承自 `FragmentFD` 而非直接继承 `FileDownloader`，这是为了复用分片处理逻辑。

### 4.2 核心参数转发函数

位置：[utils/_utils.py#L3582-L3596](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/_utils.py#L3582-L3596)

```python
def cli_option(params, command_option, param, separator=None):
    param = params.get(param)
    return ([] if param is None
            else [command_option, str(param)] if separator is None
            else [f'{command_option}{separator}{param}'])

def cli_bool_option(params, command_option, param, true_value='true', false_value='false', separator=None):
    param = params.get(param)
    assert param in (True, False, None)
    return cli_option({True: true_value, False: false_value}, command_option, param, separator)

def cli_valueless_option(params, command_option, param, expected_value=True):
    return [command_option] if params.get(param) == expected_value else []
```

**外部下载器封装：** [downloader/external.py#L118-L130](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L118-L130)

```python
def _option(self, command_option, param):
    return cli_option(self.params, command_option, param)

def _bool_option(self, command_option, param, true_value='true', false_value='false', separator=None):
    return cli_bool_option(self.params, command_option, param, true_value, false_value, separator)

def _valueless_option(self, command_option, param, expected_value=True):
    return cli_valueless_option(self.params, command_option, param, expected_value)
```

### 4.3 ratelimit 参数转发对照表

| 外部下载器 | 参数选项 | 代码位置 |
|-----------|----------|----------|
| curl | `--limit-rate` | [external.py#L237](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L237) |
| wget | `--limit-rate` | [external.py#L287](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L287) |
| aria2c | `--max-overall-download-limit` | [external.py#L322](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L322) |
| axel | 不支持 | 未实现 |
| httpie | 不支持 | 未实现 |
| ffmpeg | 不支持 | 未实现 |

**示例：** [CurlFD._make_cmd()](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L215-L249)
```python
def _make_cmd(self, tmpfilename, info_dict):
    cmd = [self.exe, '--location', '-o', tmpfilename, '--compressed']
    # ... cookies, headers 等 ...
    cmd += self._option('--limit-rate', 'ratelimit')  # ← 限速转发
    cmd += self._option('--retry', 'retries')
    cmd += self._configuration_args()  # ← 外部下载器自定义参数
    cmd += ['--', info_dict['url']]
    return cmd
```

### 4.4 外部下载器自定义参数

**_configuration_args()** — 支持按下载器名称指定额外参数
位置：[utils/_utils.py#L3619-L3629](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/_utils.py#L3619-L3629)

```python
def _configuration_args(main_key, argdict, exe, keys=None, default=[], use_compat=True):
    main_key, exe = main_key.lower(), exe.lower()
    root_key = exe if main_key == exe else f'{main_key}+{exe}'
    keys = [f'{root_key}{k}' for k in (keys or [''])]
    # ... 查找逻辑 ...
    return cli_configuration_args(argdict, keys, default, use_compat)
```

**调用方式：**
```python
cmd += self._configuration_args()
```

**参数查找顺序（以 curl 为例）：**
1. `external_downloader_args['curl']` 或 `external_downloader_args['external+curl']`
2. `external_downloader_args['default']`
3. 兼容模式：如果 `external_downloader_args` 是列表，直接使用整个列表

**用户使用示例：**
```bash
# 给 curl 额外传 --connect-timeout 30
yt-dlp --external-downloader curl \
       --external-downloader-args "curl:--connect-timeout 30" \
       URL
```

### 4.5 外部下载器的进度反馈限制

当使用外部下载器时：
- **yt-dlp 不参与下载过程**，因此没有逐块的进度回调
- 外部下载器自己处理限速（`--limit-rate` 等）
- 进度显示由外部下载器自己控制（curl/wget 会输出进度）
- **throttledratelimit 检测不生效** — 因为 yt-dlp 拿不到实时速度数据
- 下载完成后 yt-dlp 才会收到一次 `finished` 状态的钩子

位置：[downloader/external.py#L62-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L62-L76)
```python
if retval == 0:
    status = {
        'filename': filename,
        'status': 'finished',
        'elapsed': time.time() - started,
    }
    # ... 获取文件大小 ...
    self._hook_progress(status, info_dict)  # 只有这一次回调
    return True
```

---

## 五、可能影响误判的控制点

限速和节流检测依赖速度计算，多个控制点可能影响计算准确性，导致误判。

### 5.1 控制点 1：断点续传的 resume_len 处理

位置：[downloader/http.py#L229](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L229)

```python
byte_counter = 0 + ctx.resume_len
```

**速度计算时必须减去 resume_len：**
```python
# 正确：只计算本次连接下载的字节
speed = self.calc_speed(start, now, byte_counter - ctx.resume_len)
self.slow_down(start, now, byte_counter - ctx.resume_len)
```

**风险点：**
- 如果忘记减 `resume_len`，会把历史下载量算入，导致速度计算偏高
- 限速逻辑会误认为速度很快，不执行 sleep，实际会超速
- 节流检测会误认为速度很快，不会触发重定向

### 5.2 控制点 2：Content-Length 可靠性

位置：[downloader/http.py#L201-L206](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L201-L206)

```python
data_len = ctx.data.headers.get('Content-length')
if ctx.data.headers.get('Content-encoding'):
    # Content-encoding 存在时，Content-length 不可靠（自动解压）
    data_len = None
```

**当 data_len 为 None 时：**
- `eta = None`，不显示剩余时间
- `total_bytes = None`，进度条只显示已下载量，不显示百分比
- **不影响限速**（slow_down 不依赖 total）
- **不影响节流检测**（throttledratelimit 只看 speed）

### 5.3 控制点 3：第一次循环的 now = None

位置：[downloader/http.py#L234](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L234)

```python
now = None  # needed for slow_down() in the first loop run
```

**第一次循环的执行顺序：**
```
1. now = None (初始值)
2. 读取数据块
3. slow_down(start, now, ...) → now 为 None，内部用 time.time()
4. now = time.time()  ← 这里才更新
5. calc_speed(start, now, ...)
```

**slow_down 内部对 now=None 的处理：** [common.py#L206-L207](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L206-L207)
```python
if now is None:
    now = time.time()
```

**风险点：**
- 第一次循环 `slow_down` 和 `calc_speed` 使用的 `now` 有细微差异
- 差异很小（微秒级），实际影响可忽略
- 但如果代码重构时改变顺序，可能引入问题

### 5.4 控制点 4：节流检测的 3 秒防抖

位置：[downloader/http.py#L315-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L315-L325)

```python
if speed and speed < (self.params.get('throttledratelimit') or 0):
    if ctx.throttle_start is None:
        ctx.throttle_start = now       # 第一次低于阈值，记录时间
    elif now - ctx.throttle_start > 3:  # 持续超过 3 秒
        raise ThrottledDownload
elif speed:
    ctx.throttle_start = None           # 速度回升，重置计时器
```

**误判风险：**

| 场景 | 是否误判 | 原因 |
|------|----------|------|
| TCP 慢启动阶段 | 可能 | 刚开始下载速度低，3 秒内未达到全速 |
| 网络瞬时抖动 | 不会 | 3 秒防抖过滤掉 |
| 磁盘写入卡顿 | 可能 | read() 阻塞时间长，速度计算被拉低 |
| 限速生效时 | 不会 | ratelimit 控制的速度是稳定的，不会突然低于 throttledratelimit |
| 服务器限速但偶尔回升 | 不会 | 只要有一次高于阈值就重置计时 |

**TCP 慢启动问题：**
- 典型 TCP 拥塞控制从慢启动开始，窗口逐渐增大
- 如果 `throttledratelimit` 设置较高，前 3 秒可能达不到
- 解决：`throttledratelimit` 应设置为明显低于正常速度的值（如 100KB/s）

### 5.5 控制点 5：分片估算大小的准确性

位置：[downloader/fragment.py#L261-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py#L261-L264)

```python
estimated_size = (
    (ctx['complete_frags_downloaded_bytes'] + frag_total_bytes)
    / (state['fragment_index'] + 1) * total_frags)
```

**估算公式：**
```
预估总大小 = (已下载字节 + 当前分片大小) / 已完成分片数 × 总分片数
```

**风险点：**
- 分片大小不均匀时（如广告分片、片头片尾），估算偏差大
- 刚开始只有少数分片完成时，估算不可靠
- 影响 `eta` 计算，但 **不影响 speed 计算**（speed 只看下载速率）
- 不影响限速和节流检测

### 5.6 控制点 6：并发分片对限速的影响

位置：[downloader/fragment.py#L167-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py#L167-L174)

```python
dl = HttpQuietDownloader(self.ydl, {
    **self.params,  # ratelimit 被传递给每个子下载器
    'noprogress': True,
    ...
})
```

**当 concurrent_fragment_downloads = N 时：**
- 每个分片线程有独立的 `slow_down()`
- 每个线程独立限速 `ratelimit`
- **实际总带宽 = ratelimit × N**
- 这是设计行为，但用户可能误以为是"全局"限速

**节流检测的情况：**
- 每个分片独立检测自己的速度
- 单个分片速度低会触发该分片的重定向
- 但不会影响其他分片

### 5.7 控制点 7：ProgressCalculator 的采样率限制

位置：[utils/progress.py#L70-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L70-L72)

```python
if self._last_update + self.SAMPLING_RATE > current_time:
    return  # 50ms 内不重复采样
self._last_update = current_time
```

**影响：**
- 速度更新频率最高 20 次/秒
- 进度显示不会过于频繁，节省 CPU
- 但节流检测（在 HttpFD 中）不受此限制，每次循环都检测

### 5.8 控制点 8：外部下载器的节流检测失效

如 4.5 节所述，使用外部下载器时：
- `throttledratelimit` 检测 **完全不生效**
- 因为 yt-dlp 拿不到实时速度数据
- 限速由外部下载器自己实现
- 用户需要依赖外部下载器的限流检测机制

---

## 六、限速与节流检测的完整决策树

```
下载开始
    │
    ▼
┌─────────────────────────────────────┐
│  计算速度 speed = bytes / elapsed    │
│  (HttpFD: 全程平均)                 │
│  (FragmentFD: 3s 滑动窗口 + 平滑)   │
└──────────────┬──────────────────────┘
               │
       ┌───────┴───────┐
       │               │
       ▼               ▼
┌─────────────┐  ┌───────────────────┐
│ slow_down() │  │ throttledratelimit │
│ 限速检查    │  │ 节流检测            │
│             │  │  速度 < 阈值?       │
│ speed > 限速?│  │  ├─ 是 → 计时     │
│  ├─ 是 → sleep│  │  │   持续 3s?    │
│  └─ 否 → 继续 │  │  │   ├─ 是 → 抛出 │
│             │  │  │   └─ 否 → 继续  │
└─────────────┘  │  ├─ 否 → 重置计时  │
                 │  └─ speed 为 None → 跳过 │
                 └───────────────────┘
```

---

## 七、相关文件索引

| 文件 | 核心内容 |
|------|----------|
| [downloader/common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py) | `slow_down()`、`calc_speed()`、`calc_eta()`、`best_block_size()`、进度钩子框架 |
| [downloader/http.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py) | HttpFD 下载循环、限速调用点、节流检测逻辑 |
| [downloader/fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py) | 分片进度聚合、`frag_progress_hook`、分片估算 |
| [downloader/external.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py) | 外部下载器参数转发、各下载器 `_make_cmd()` |
| [utils/progress.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py) | `ProgressCalculator` 滑动窗口、`SmoothValue` 指数平滑 |
| [utils/_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/_utils.py) | `ThrottledDownload` 异常、`cli_option()` 系列、`_configuration_args()` |
| [options.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/options.py) | CLI 选项：`-r/--limit-rate`、`--throttled-rate` |
| [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/YoutubeDL.py) | 参数汇总与传递 |
