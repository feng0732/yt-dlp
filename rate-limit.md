# yt-dlp 下载限速机制深度分析

## 概述

yt-dlp 的下载限速控制点分散在多个模块中，核心是两个阈值和一套反馈系统的协作：

1. **ratelimit（上限阈值）**：主动限速，通过 sleep 控制单连接不超过该速度
2. **throttledratelimit（下限阈值）**：被动检测，速度持续低于该值视为被服务器限流
3. **进度反馈系统**：计算并展示速度/ETA

**关键原则**：进度反馈系统（ProgressCalculator、report_progress）**只负责展示**，真正参与控制决策（限速和节流检测）的速度值全部在下载循环内部即时计算，不依赖展示层的数据。

---

## 一、两个阈值的语义与代码定位

### 1.1 语义对照表

| 参数 | 语义 | 方向 | 触发动作 |
|------|------|------|----------|
| `ratelimit` | 最大允许速度（**上限**） | `speed > ratelimit` → sleep | 主动减速，保护带宽 |
| `throttledratelimit` | 判定限流的最小速度（**下限**） | `speed < throttledratelimit` 持续 3s → 重定向 | 被动检测，切换 CDN |

### 1.2 代码定位

**ratelimit 参数定义：**
- CLI：[options.py#L1016-L1018](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/options.py#L1016-L1018) → `-r` / `--limit-rate`
- 文档：[common.py#L51](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L51)
- 实现：[common.py#L201-L215](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L201-L215) → `slow_down()`
- 调用：[http.py#L281](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L281)

**throttledratelimit 参数定义：**
- CLI：[options.py#L1020-L1022](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/options.py#L1020-L1022) → `--throttled-rate`
- 文档：[common.py#L52](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L52)
- 异常：[_utils.py#L1130-L1135](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/_utils.py#L1130-L1135) → `ThrottledDownload`
- 检测：[http.py#L315-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L315-L325)

---

## 二、控制决策用的速度值 vs 展示用的速度值

### 2.1 核心结论：完全分离的两套速度计算

这是理解整个系统最关键的一点：

```
下载循环内部（控制决策层）          进度反馈系统（展示层）
═══════════════════════════        ════════════════════════
                                        ┌──────────────────┐
┌──────────────────────────┐            │ FragmentFD 专用:  │
│  1. slow_down() 内部:    │            │ ProgressCalculator│
│     speed_local = bytes /│            │   (滑动窗口 3s)   │
│     elapsed              │            │   + SmoothValue   │
│     ↓                    │            │   (指数平滑)      │
│  是否需要 sleep?         │            └────────┬─────────┘
└──────────────────────────┘                     │
                        │                        │
┌──────────────────────────┐                     ▼
│  2. calc_speed()         │            _hook_progress()
│     speed_ctrl = bytes /│             │
│     elapsed              │             ├─→ report_progress() 格式化显示
│     ↓                    │             │     (纯展示，无决策)
│  传给 throttledratelimit │             │
│  检测 + _hook_progress   │             └─→ 用户自定义 progress hooks
└──────────────────────────┘                    (观察者模式)
```

### 2.2 控制决策层：三处速度计算

下载循环位置：[http.py#L248-L326](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L248-L326)

**控制决策使用的 3 个速度值：**

#### （1）slow_down 内部计算 — 用于限速决策

位置：[common.py#L201-L215](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L201-L215)

```python
def slow_down(self, start_time, now, byte_counter):
    rate_limit = self.params.get('ratelimit')
    if rate_limit is None or byte_counter == 0:
        return
    if now is None:
        now = time.time()          # now=None 时自己取时间
    elapsed = now - start_time
    if elapsed <= 0.0:
        return
    speed = float(byte_counter) / elapsed   # ← 第 1 套独立计算
    if speed > rate_limit:
        sleep_time = float(byte_counter) / rate_limit - elapsed
        if sleep_time > 0:
            time.sleep(sleep_time)
```

**关键特征：**
- **独立计算**，不依赖任何外部变量
- 输入参数和 `calc_speed()` 相同，但实现位置不同
- **第一次循环**时 `now=None`，内部会自己调用 `time.time()`

#### （2）calc_speed() 返回值 — 用于节流检测和展示

位置：[common.py#L160-L165](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L160-L165)

```python
@staticmethod
def calc_speed(start, now, bytes):
    dif = now - start
    if bytes == 0 or dif < 0.001:
        return None
    return float(bytes) / dif
```

调用位置：[http.py#L294](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L294)
```python
speed = self.calc_speed(start, now, byte_counter - ctx.resume_len)
```

**这个 `speed` 变量被两处使用：**
1. **节流检测（控制决策）**：[http.py#L315](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L315) → `if speed and speed < throttledratelimit`
2. **进度钩子（展示）**：[http.py#L307](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L307) → 放入 `_hook_progress` 的 status 字典

#### （3）best_block_size 内部计算 — 用于调整读取块大小

位置：[common.py#L181-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L181-L192)

```python
@staticmethod
def best_block_size(elapsed_time, bytes):
    # ...
    rate = bytes / elapsed_time   # ← 第 3 套独立计算
    # 用 rate 决定下一次 read 的块大小
```

**这个 `rate` 只用于动态调整块大小，不参与限速或节流决策。**

### 2.3 展示层：纯观察者模式

#### report_progress() — 只格式化，不做决策

位置：[common.py#L342-L404](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L342-L404)

```python
def report_progress(self, s):
    # finished 状态时，重新计算一个速度用于展示
    if s['status'] == 'finished':
        speed = try_call(lambda: s['total_bytes'] / s['elapsed'])  # ← 仅展示
        s.update({
            'speed': speed,
            '_speed_str': self.format_speed(speed).strip(),
            ...
        })
        self._report_progress_status(s, ...)   # 格式化+打印

    if s['status'] != 'downloading':
        return

    # progress_delta 只限制输出频率，不影响速度计算
    if update_delta := self.params.get('progress_delta'):
        with self._progress_delta_lock:
            if time.monotonic() < self._progress_delta_time:
                return              # 跳过多余的展示
            self._progress_delta_time += update_delta

    # 格式化 percent、speed_str、eta_str → 纯字符串操作
    s.update({
        '_eta_str': self.format_eta(s.get('eta')).strip(),
        '_speed_str': self.format_speed(s.get('speed')),
        '_percent_str': self.format_percent(progress),
        ...
    })
    self._report_progress_status(s, msg_template)  # 最终输出
```

**关键证据：report_progress 中所有计算只修改 `s['_xxx_str']` 等格式化字段，从不反向影响循环内的控制变量。**

#### 进度钩子链路

位置：[YoutubeDL.py#L3304-L3318](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/YoutubeDL.py#L3304-L3318)

```python
fd = get_suitable_downloader(info, params, ...)(self, params)
if not test:
    for ph in self._progress_hooks:     # 用户通过 API 注册的钩子
        fd.add_progress_hook(ph)

# FileDownloader.__init__ 中自带的钩子：
self.add_progress_hook(self.report_progress)  # 内置，第 1 个调用
```

钩子调用位置：[common.py#L488-L500](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L488-L500)
```python
def _hook_progress(self, status, info_dict):
    status['info_dict'] = info_dict
    for ph in self._progress_hooks:
        ph(status)              # 按注册顺序依次调用
```

**结论：所有 progress hooks 都是观察者，无法修改循环内的 `speed`、`byte_counter` 等变量，只能读取 status 字典。**

### 2.4 FragmentFD 特殊情况：两套速度并行

在分片下载场景中，控制决策和展示使用的速度来源**完全不同**：

```
每个分片内部 (HttpQuietDownloader = HttpFD)
═══════════════════════════════════════════
  子下载循环中的 calc_speed() 结果
    ├──→ 子下载器的 throttledratelimit 检测  ← 控制决策（每个分片独立）
    ├──→ 子下载器的 slow_down() 限速         ← 控制决策（每个分片独立）
    └──→ frag_progress_hook() 接收
              │
              ▼
FragmentFD 聚合层 (展示专用)
═════════════════════════════
  ProgressCalculator.update(子下载器的 downloaded_bytes)
    │
    ├─→ 3s 滑动窗口计算速度
    ├─→ SmoothValue 指数平滑
    │
    └─→ state['speed'] = progress.speed.smooth   ← 仅展示，不参与控制
        state['eta'] = progress.eta.smooth       ← 仅展示，不参与控制
        │
        └─→ self._hook_progress(state, ...)
              ├─→ report_progress() 显示         ← 仅展示
              └─→ 用户自定义钩子                  ← 仅观察
```

**代码证据：FragmentFD 中没有 throttledratelimit 检测逻辑**

在 [downloader/fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py) 中搜索 `throttledratelimit` 或 `ThrottledDownload`，结果为 **0 匹配**。这证明：

- 节流检测完全发生在子下载器（HttpQuietDownloader）内部
- FragmentFD 层的 ProgressCalculator 计算出来的平滑速度**不参与任何控制决策**
- 分片下载的限速也完全在子下载器内部独立执行

### 2.5 速度值分类总表

| 速度值 | 计算位置 | 算法 | 用途 | 是否参与控制决策 |
|--------|----------|------|------|------------------|
| slow_down 内部 speed | [common.py#L211](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L211) | 本次连接全程平均 | 限速 sleep | **是** |
| calc_speed() 返回值 | [common.py#L165](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L165) | 本次连接全程平均 | 节流检测 + 展示 | **是（仅HttpFD）** |
| best_block_size 内部 rate | [common.py#L187](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L187) | 单次 read 平均 | 调整块大小 | 间接影响 |
| ProgressCalculator.speed.value | [progress.py#L89](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L89) | 3s 滑动窗口平均 | 平滑前原始值 | **否（仅展示）** |
| ProgressCalculator.speed.smooth | [progress.py#L106](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L106) | 指数平滑后 | FragmentFD 展示用 | **否（仅展示）** |
| report_progress finished 速度 | [common.py#L354](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L354) | 总大小 / 总耗时 | 完成时展示 | **否（仅展示）** |

---

## 三、ratelimit 与 throttledratelimit 的阈值组合关系

### 3.1 四种组合情况

```
速度轴 (bytes/s)
──────────────────────────────────────────────→
              │                │
  throttledratelimit      ratelimit
    (下限，检测节流)     (上限，主动限速)
```

#### 组合 1：ratelimit > throttledratelimit（正常配置）

```
示例：ratelimit = 2M (2097152), throttledratelimit = 100K (102400)

         100K              2M
───────────┼─────────────────┼───────────→
           │   正常工作区    │
           │                 │
      速度在此区间内可能触发    限速生效的稳定区
      throttledratelimit 检测
```

**行为分析：**
- 正常下载时，ratelimit 将速度稳定在 2MB/s 附近
- 2MB/s 远高于 100KB/s，节流检测不会被触发
- 只有当服务器真正限流时（速度从 2M 掉落到 100K 以下并持续 3s），才会触发重定向
- **这是预期的工作模式**

#### 组合 2：ratelimit < throttledratelimit（冲突配置 — 必触发）

```
示例：ratelimit = 50K (51200), throttledratelimit = 100K (102400)

          50K         100K
───────────┼────────────┼──────────────→
           │            │
    ratelimit 生效后    throttledratelimit 触发线
    速度被限制在 50K    但 50K < 100K！
    ←──────── 速度永远在这一侧
```

**这是最危险的配置，详细分析见下一节。**

#### 组合 3：ratelimit = throttledratelimit（临界配置 — 大概率误判）

```
示例：ratelimit = 100K, throttledratelimit = 100K

          100K
───────────┼──────────────────────────────→
           │
      两个阈值重合
```

**行为分析：**
- 限速系统的 sleep 精度有限（`time.sleep()` 精度 + 调度抖动），实际速度会在 ratelimit 附近上下波动
- 波动的下探部分会进入 `< throttledratelimit` 区域
- 一旦下探持续 3 秒以上，就会误判
- **这种配置几乎一定会触发误判**

#### 组合 4：未设置 throttledratelimit（默认值 0）

```python
# http.py#L315
if speed and speed < (self.params.get('throttledratelimit') or 0):
```

- `throttledratelimit or 0` → 未设置时使用 0
- `speed < 0` 在正常下载中永远不可能成立（speed ≥ 0）
- **节流检测完全不生效**

### 3.2 组合 2 详细分析：低限速遇高节流阈值

#### 场景

```bash
# 用户同时设置了两个参数，但设置反了
yt-dlp -r 50K --throttled-rate 100K URL
# ratelimit (50K) < throttledratelimit (100K)
```

#### 逐步执行过程

下载循环：[http.py#L248-L326](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L248-L326)

```
第 1 阶段：T=0s ~ T=0.5s
  刚开始下载，速度逐步上升
  speed 从 0 开始增长
  speed < 50K，slow_down 不 sleep
  speed < 100K，记录 throttle_start = T=0.0

第 2 阶段：T=0.5s ~ T=1s
  speed 达到并超过 50K
  → slow_down() 开始介入：计算 sleep_time，主动 sleep
  速度被限制在 50K 左右
  但 50K < 100K → throttle_start 没有被重置
  throttle_start 仍然是 T=0.0

第 3 阶段：T=3.0s（关键节点）
  现在的时间是 T=3.0
  now - throttle_start = 3.0 - 0.0 = 3.0s
  且 speed (≈50K) < throttledratelimit (100K)
  → 条件 now - ctx.throttle_start > 3 成立！
  → 抛出 ThrottledDownload 异常 ← 误判！
```

#### 误判的代码路径

```python
# http.py#L315-L325
if speed and speed < (self.params.get('throttledratelimit') or 0):
    # speed ≈ 50K < 100K → True
    if ctx.throttle_start is None:
        ctx.throttle_start = now       # T≈0 时设值
    elif now - ctx.throttle_start > 3:
        # T=3 时判断为 True
        if ctx.stream is not None and ctx.tmpfilename != '-':
            ctx.stream.close()
        raise ThrottledDownload   # ← 这里抛出，误判
elif speed:
    ctx.throttle_start = None   # ← 永远走不到这里，因为 speed 始终 < 100K
```

#### 实际后果

1. 抛出 `ThrottledDownload` → 被识别为 `ReExtractInfo` 类型
2. YoutubeDL 会重新提取视频信息（重新向 YouTube/其他站点请求获取新的下载 URL）
3. 用新 URL 重新开始下载 → 但新 URL 的 ratelimit 仍然是 50K，throttledratelimit 仍然是 100K
4. **3 秒后再次误判**，形成循环：
   ```
   下载 3s → 误判 ThrottledDownload → 重新提取 URL → 再下载 3s → 再次误判 ...
   ```
5. 如果重试次数耗尽，最终下载失败

#### 为什么 FragmentFD 中每个分片独立受影响

在分片下载场景下：
- 每个分片通过 `HttpQuietDownloader`（本质是 HttpFD）独立下载
- 每个分片都有自己的下载循环和节流检测逻辑
- 第 1 个分片下载 3s 后触发误判，抛出 `ThrottledDownload`
- 这个异常会向上冒泡，影响整个分片下载流程
- **不会等到所有分片都失败，第一个分片 3 秒后就会触发**

---

## 四、限速、节流与进度的关联全景

### 4.1 核心协作链路

```
用户参数 (ratelimit, throttledratelimit)
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│           FileDownloader 基类                             │
│  ┌──────────────────────┐   ┌────────────────────────┐   │
│  │ slow_down()          │   │ report_progress()      │   │
│  │  内部计算 speed_local │   │  纯格式化+展示         │   │
│  │  → time.sleep()      │   │  (不修改任何控制变量)   │   │
│  └─────────┬────────────┘   └───────────┬────────────┘   │
└────────────┼────────────────────────────┼────────────────┘
             │                            │
             ▼                            ▼
┌──────────────────────────┐   ┌───────────────────────────┐
│  HttpFD (单文件)          │   │ FragmentFD (分片聚合)     │
│  calc_speed()             │   │ ProgressCalculator       │
│  → 用于 throttledratelimit│   │ + SmoothValue             │
│  → 放入 _hook_progress    │   │ → state['speed'].smooth   │
│  (控制 + 展示共用)        │   │ (仅展示，不参与控制)      │
└────────────┬─────────────┘   └────────────┬──────────────┘
             │                               │
             │                               │ (子下载器内部)
             │                               └─────┐
             ▼                                     ▼
    节流检测决策 (HttpFD 层)            每个分片独立决策
    speed_ctrl < throttledratelimit?    (同上，每个分片走独立循环)
     持续 3s → ThrottledDownload
```

### 4.2 下载循环中的调用时序

位置：[downloader/http.py#L248-L326](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L248-L326)

```
while True:
    # ── 数据层 ──
    1. data_block = ctx.data.read(block_size)
    2. byte_counter += len(data_block)
    3. ctx.stream.write(data_block)

    # ── 控制决策 1：限速 ──
    4. slow_down(start, now, byte_counter - ctx.resume_len)
       │   内部独立计算 speed_local
       │   与 calc_speed 结果可能有微秒级差异
       └─→ sleep (如果需要)

    # ── 时间戳更新 ──
    5. now = time.time()
    6. block_size = best_block_size(after - before, len(data_block))
       │   内部独立计算 rate (单次 read 平均)
       └─→ 决定下一次 read 的块大小

    # ── 计算速度：同一个值服务两个目的 ──
    7. speed = calc_speed(start, now, byte_counter - ctx.resume_len)
       │
       ├─── 控制决策 2：节流检测
       │    10. if speed < throttledratelimit:
       │            持续 3s → 抛 ThrottledDownload
       │
       └─── 展示层：进度钩子
            8. eta = calc_eta(...)
            9. _hook_progress({status, speed, eta, ...})
                 │
                 ├── report_progress() → 格式化输出
                 └── 用户自定义 hooks → 纯观察
```

### 4.3 限速生效后的稳定状态分析

当 ratelimit 正常工作且 `ratelimit > throttledratelimit` 时：

```
时间轴: T0────T1────T2────T3────T4────T5──→
速度:
     ▲   ↗ 快速爬升
     │  ↗
 ratelimit━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ (稳定上限)
     │╲  ╱╲  ╱╲  ╱╲  ╱╲  ╱       ← 实际速度在 ratelimit 附近抖动
     │ ╲╱  ╲╱  ╲╱  ╲╱  ╲╱
     │
throttledratelimit ─────────────────────── (安全下限)
     │
     0

关键点:
• ratelimit 通过 sleep 把平均速度拉平到上限值
• 波动范围通常在 ratelimit 的 ±10% 以内
• 只要 throttledratelimit 明显低于波动下限，就不会误判
• 经验建议：throttledratelimit ≤ ratelimit × 0.3 （留 70% 安全裕度）
```

---

## 五、误判控制点汇总

除了之前的 8 个控制点，补充阈值组合相关的控制点：

### 5.1 控制点 9：阈值相对大小（新增）

**风险等级：最高**

- **情况 A**：`ratelimit < throttledratelimit` → **必误判**，3 秒后必定触发 ThrottledDownload
- **情况 B**：`ratelimit ≈ throttledratelimit` → **大概率误判**，速度波动下探到阈值以下
- **情况 C**：`ratelimit > throttledratelimit × 3` → 安全

### 5.2 控制点 10：第一次循环的速度为 None

位置：[common.py#L163](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L163)

```python
if bytes == 0 or dif < 0.001:  # 1ms 保护
    return None
```

节流检测的判断条件：
```python
if speed and speed < throttledratelimit:  # speed 为 None 时跳过
```

- 第 1ms 内 `speed=None`，`if speed and ...` 短路，不检测
- 1ms 后开始检测，但 TCP 慢启动 + ratelimit sleep，前几百毫秒速度还在爬升
- 所以 `throttle_start` 通常在 T≈0.1s 才会被设置，3 秒阈值实际是 T≈3.1s 才触发

### 5.3 控制点 11：sleep 后的速度计算包含 sleep 时间

```
循环 1: read (实际 1ms, 得 1MB)
         slow_down 计算: 1MB/1ms = 1000MB/s → 远超 ratelimit=1MB/s
         sleep_time = 1MB / 1MB/s - 1ms ≈ 999ms
         → sleep 999ms

循环 2: read (1ms, 得 1MB)
         calc_speed 计算: bytes=2MB, elapsed=1ms+999ms+1ms ≈ 1001ms
         speed = 2MB / 1.001s ≈ 1.998MB/s
```

**关键点：**
- `calc_speed()` 计算的 `elapsed` 包含了 `slow_down()` 内部 sleep 的时间
- 所以限速生效后，`calc_speed()` 返回值才是真实的平均速度（≈ ratelimit）
- 这保证了节流检测看到的是限速后的真实速度，而不是瞬时读取速度

### 5.4 完整误判控制点表

| 编号 | 控制点 | 位置 | 影响 | 风险等级 |
|------|--------|------|------|----------|
| 1 | resume_len 处理 | [http.py#L229](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L229) | 忘记减会导致速度虚高 | 中 |
| 2 | Content-Length 可靠性 | [http.py#L201-L206](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L201-L206) | 只影响 ETA，不影响控制 | 低 |
| 3 | now=None 时序 | [http.py#L234](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L234) | 微秒级差异，影响可忽略 | 低 |
| 4 | 3 秒防抖 | [http.py#L315-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L315-L325) | TCP 慢启动可能误判 | 中 |
| 5 | 分片大小估算 | [fragment.py#L261-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py#L261-L264) | 只影响 ETA 展示 | 低 |
| 6 | 并发分片限速 | [fragment.py#L167-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py#L167-L174) | 总带宽 = ratelimit × N | 中 |
| 7 | 采样率限制 | [progress.py#L70-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L70-L72) | 只影响展示刷新频率 | 低 |
| 8 | 外部下载器 | [external.py#L62-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L62-L76) | throttledratelimit 完全失效 | 高 |
| **9** | **阈值相对大小** | **参数配置层** | **ratelimit < throttledratelimit 必误判** | **最高** |
| 10 | 1ms 内 speed=None | [common.py#L163](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L163) | 跳过检测，实际延迟触发 | 低 |
| 11 | sleep 计入 elapsed | 循环时序 | 保证节流检测看到限速后真实速度 | 有利（降低误判） |

---

## 六、普通下载速度的计算方法

### 6.1 速度计算核心函数

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

### 6.2 ETA 计算核心函数

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

### 6.3 速度计算中的变量说明

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

### 6.4 动态块大小调整（间接影响控制精度）

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
- **对控制精度的影响**：块太小会增加循环次数，限速和节流检测更精确；块太大则反之

---

## 七、分片聚合的展示方式

### 7.1 ProgressCalculator 滑动窗口算法

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

### 7.2 SmoothValue 指数平滑器

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

### 7.3 多线程支持（仅展示层用）

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
- **重要：** 这套机制只为展示层聚合速度用，不参与每个分片内部的控制决策

### 7.4 分片进度聚合逻辑

位置：[downloader/fragment.py#L242-L279](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py#L242-L279)

```python
def frag_progress_hook(s):
    # 只处理 downloading 和 finished 状态
    if s['status'] not in ('downloading', 'finished'):
        return

    # ctx_id 校验：确保是当前上下文的进度事件
    if ctx_id is not None and s.get('ctx_id') != ctx_id:
        return

    # 非直播场景：估算总大小（仅用于 ETA 展示）
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
        progress.thread_reset()  # 重置线程累计值（展示层用）

    # 聚合后的状态 → 全部用于展示
    state['downloaded_bytes'] = ctx['complete_frags_downloaded_bytes'] = progress.downloaded
    state['speed'] = ctx['speed'] = progress.speed.smooth   # 平滑后的值，仅展示
    state['eta'] = progress.eta.smooth                       # 平滑后的值，仅展示

    self._hook_progress(state, info_dict)  # 分发给展示层钩子
```

**展示的数据流向：**
```
子下载器 (HttpQuietDownloader)
    │  (每个分片独立下载，noprogress=True)
    │  内部独立执行 slow_down 和 throttledratelimit 检测
    │  ↓ 触发自己的 _hook_progress (status 含 speed)
    ▼
frag_progress_hook (FragmentFD 的分片聚合钩子)
    │  读取子下载器传来的 downloaded_bytes
    │  ProgressCalculator.update() 汇总到滑动窗口
    │  SmoothValue 指数平滑
    ▼
state['speed'] = progress.speed.smooth   ← 仅展示，不参与控制
state['eta'] = progress.eta.smooth       ← 仅展示，不参与控制
    │
    ▼
self._hook_progress(state, info_dict)
    │
    ├── report_progress() → 进度条显示
    └── 用户自定义 progress hooks → 观察者模式（只读）
```

### 7.5 分片与非分片的对比

| 维度 | HttpFD (非分片) | FragmentFD (分片) |
|------|-----------------|-------------------|
| 控制决策用速度算法 | calc_speed() 全程平均 | **每个分片内部**用 calc_speed() 全程平均（各分片独立） |
| 展示用速度算法 | 同上，与控制层共用 | ProgressCalculator 3s 窗口 + SmoothValue 平滑（与控制层分离） |
| 限速执行位置 | HttpFD 下载循环 | **每个分片的子下载器**独立执行（HttpQuietDownloader 内部） |
| 节流检测位置 | HttpFD 下载循环 | **每个分片的子下载器**独立检测（FragmentFD 层无此逻辑） |
| 数据一致性 | 控制和展示用同一个 speed 变量 | 两套完全独立的计算，可能出现显示值与控制值不一致 |

---

## 八、外部下载器参数的转发机制

### 8.1 类层次结构

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

### 8.2 核心参数转发函数

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

### 8.3 ratelimit 参数转发对照表

| 外部下载器 | 参数选项 | 语义 | 代码位置 |
|-----------|----------|------|----------|
| curl | `--limit-rate` | 单连接限速 | [external.py#L237](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L237) |
| wget | `--limit-rate` | 单连接限速 | [external.py#L287](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L287) |
| aria2c | `--max-overall-download-limit` | **全局总带宽**限速 | [external.py#L322](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L322) |
| axel | 不支持 | - | 未实现 |
| httpie | 不支持 | - | 未实现 |
| ffmpeg | 不支持 | - | 未实现 |

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

**重要差异：aria2c 用 `--max-overall-download-limit`（全局限速）而不是单连接限速。** 这意味着只有 aria2c 外部下载器能实现真正意义上的全局带宽池，其他外部下载器和内置下载器都是单连接限速。

### 8.4 外部下载器自定义参数

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

### 8.5 外部下载器的控制决策全部外包

当使用外部下载器时：

| 功能 | 内置下载器 | 外部下载器 |
|------|-----------|-----------|
| ratelimit 限速 | slow_down() 内部 sleep | 转发给外部工具（aria2c 是全局，其他是单连接） |
| throttledratelimit 检测 | 循环内判断 3s 防抖 | **完全失效**（yt-dlp 不解析外部工具输出） |
| 速度计算 | calc_speed() / ProgressCalculator | **无实时数据**，只在完成时用 `total_bytes / elapsed` 算一次 |
| ETA 计算 | calc_eta() / ProgressCalculator | **无** |
| 进度钩子 | 逐块回调 | 只有 start 和 finished 两次 |
| 块大小调整 | best_block_size() | 外部工具自己处理 |

位置：[downloader/external.py#L62-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py#L62-L76)
```python
if retval == 0:
    status = {
        'filename': filename,
        'status': 'finished',
        'elapsed': time.time() - started,
    }
    if filename != '-':
        fsize = os.path.getsize(tmpfilename)
        self.try_rename(tmpfilename, filename)
        status.update({
            'downloaded_bytes': fsize,
            'total_bytes': fsize,
        })
    self._hook_progress(status, info_dict)  # ← 只有这一次回调
    return True
```

---

## 九、限速与节流检测的完整决策树

```
下载开始
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  下载循环 (每个 read() 一次)                          │
│                                                      │
│  ┌─ slow_down() ─────────────────────────────────┐   │
│  │  ① 内部计算 speed_local = bytes/elapsed        │   │
│  │  ② if speed_local > ratelimit: time.sleep()   │   │
│  │     (使用 本次连接全程平均)                     │   │
│  └───────────────────────────────────────────────┘   │
│                                                      │
│  ┌─ calc_speed() ────────────────────────────────┐   │
│  │  speed_ctrl = bytes/elapsed                    │   │
│  │  (使用 本次连接全程平均，与①算法相同)            │   │
│  │                                               │   │
│  │  用途 1: 放入 _hook_progress → report_progress │   │
│  │          (仅展示，不影响控制)                    │   │
│  │                                               │   │
│  │  用途 2: 节流检测决策 ← 真正的控制              │   │
│  └───────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
┌────────────────────┐   ┌────────────────────────────┐
│ FragmentFD 场景?    │   │ HttpFD 场景?                │
│ (分片下载)          │   │ (单文件下载)                │
└─────────┬──────────┘   └──────────────┬─────────────┘
          │                             │
          │ 每个分片独立走 HttpFD 循环   │ speed_ctrl 参与决策
          │ (内部限速 + 内部节流检测)    │
          │                             │
          ▼                             ▼
   子下载器内判断                if speed_ctrl < throttledratelimit:
   (同上)                    ┌─ 是 → ctx.throttle_start 设值
                             │     持续 3s 以上?
                             │     ├─ 是 → raise ThrottledDownload
                             │     └─ 否 → 等待下一轮
                             └─ 否 (或 speed 回升) → throttle_start 清零
```

---

## 十、相关文件索引

| 文件 | 核心内容 |
|------|----------|
| [downloader/common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py) | `slow_down()`（限速，第1套速度计算）、`calc_speed()`（第2套，控制+展示）、`calc_eta()`、`best_block_size()`（第3套，块大小）、`report_progress()`（纯展示）、进度钩子框架 |
| [downloader/http.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py) | HttpFD 下载循环、限速调用点 `#L281`、节流检测 `#L315-L325`、speed 变量共用控制+展示 |
| [downloader/fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py) | 分片进度聚合 `frag_progress_hook`、分片大小估算、子下载器参数传递、无节流检测逻辑 |
| [downloader/external.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py) | 外部下载器参数转发、各下载器 `_make_cmd()`、ratelimit 映射、节流检测完全失效 |
| [utils/progress.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py) | `ProgressCalculator`（3s 滑动窗口，仅展示层）、`SmoothValue`（指数平滑，仅展示层） |
| [utils/_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/_utils.py) | `ThrottledDownload` 异常定义（`#L1130`）、`cli_option()` 系列（参数转发）、`_configuration_args()`（外部下载器自定义参数） |
| [options.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/options.py) | CLI 选项：`-r/--limit-rate`（`#L1016`）、`--throttled-rate`（`#L1020`） |
| [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/YoutubeDL.py) | 参数汇总、`dl()` 方法 `#L3283` 创建 FD 实例并注册 progress hooks、钩子链路组装 |
