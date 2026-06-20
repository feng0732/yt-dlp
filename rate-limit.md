# yt-dlp 下载限速机制分析

## 概述

yt-dlp 的下载限速控制点分布在多个模块中，主要包含三个核心机制：

1. **全局限速（ratelimit）**：用户设定的最大下载速度限制
2. **单任务节流检测（throttledratelimit）**：检测下载是否被服务器限流
3. **进度反馈系统**：实时计算下载速度、ETA，配合限速和节流检测

以下是详细分析。

---

## 一、全局限速（ratelimit）

### 1.1 配置入口

命令行选项：`-r` / `--limit-rate` / `--rate-limit`

定义位置：[options.py#L1016-L1018](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/options.py#L1016-L1018)

```python
downloader.add_option(
    '-r', '--limit-rate', '--rate-limit',
    dest='ratelimit', metavar='RATE',
    help='Maximum download rate in bytes per second, e.g. 50K or 4.2M')
```

参数传递路径：CLI → YoutubeDL.params → FileDownloader.params

### 1.2 核心实现：slow_down() 方法

位置：[downloader/common.py#L201-L215](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L201-L215)

```python
def slow_down(self, start_time, now, byte_counter):
    """Sleep if the download speed is over the rate limit."""
    rate_limit = self.params.get('ratelimit')
    if rate_limit is None or byte_counter == 0:
        return
    if now is None:
        now = time.time()
    elapsed = now - start_time
    if elapsed <= 0.0:
        return
    speed = float(byte_counter) / elapsed
    if speed > rate_limit:
        sleep_time = float(byte_counter) / rate_limit - elapsed
        if sleep_time > 0:
            time.sleep(sleep_time)
```

**工作原理：**

- 基于「从下载开始到当前时刻的平均速度
- 如果平均速度超过限速，则计算需要 sleep 的时间
- sleep 时间 = (已下载字节数 / 限速) - 已用时间
- 通过 sleep 来拉低平均速度

**特点：**

- 是「单连接限速，每个下载连接独立控制
- 不是真正的全局限速器（没有全局共享的带宽池
- 并发下载多个分片时，每个分片各自限速独立计算，总带宽可能超限

### 1.3 HTTP 下载器中的调用位置

位置：[downloader/http.py#L280-L281](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L280-L281)

```python
# Apply rate limit
self.slow_down(start, now, byte_counter - ctx.resume_len)
```

在每次读取数据块读取后调用，对本次循环中调用一次。

### 1.4 分片下载中的限速

在 [FragmentFD](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py) 中，每个分片通过内部的 HttpQuietDownloader 下载，每个分片独立应用 ratelimit。

位置：[downloader/fragment.py#L167-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py#L167-L174)

```python
dl = HttpQuietDownloader(self.ydl, {
    **self.params,  # ratelimit 等参数被传递给子下载器
    'noprogress': True,
    ...
})
```

当 `concurrent_fragment_downloads > 1 时，每个并发分片线程各自有自己的 slow_down，总带宽 = 限速 × 并发数。

### 1.5 外部下载器的限速

对于 aria2c、curl、wget 等外部下载器也支持 ratelimit，通过命令行参数传递：

位置：[downloader/external.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/external.py)

- curl/wget: `--limit-rate`
- aria2c: `--max-overall-download-limit`

---

## 二、单任务节流检测（throttledratelimit）

### 2.1 配置入口

命令行选项：`--throttled-rate`

定义位置：[options.py#L1020-L1022](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/options.py#L1020-L1022)

```python
downloader.add_option(
    '--throttled-rate',
    dest='throttledratelimit', metavar='RATE',
    help='Minimum download rate in bytes per second below which throttling is assumed and the video data is re-extracted, e.g. 100K')
```

### 2.2 核心逻辑

位置：[downloader/http.py#L315-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L315-L325)

```python
if speed and speed < (self.params.get('throttledratelimit') or 0):
    # The speed must stay below the limit for 3 seconds
    # This prevents raising error when the speed temporarily goes down
    if ctx.throttle_start is None:
        ctx.throttle_start = now
    elif now - ctx.throttle_start > 3:
        if ctx.stream is not None and ctx.tmpfilename != '-':
            ctx.stream.close()
        raise ThrottledDownload
elif speed:
    ctx.throttle_start = None
```

**工作原理：**

1. 每次循环中计算当前下载速度
2. 如果速度低于 throttledratelimit，记录开始时间 throttle_start
3. 如果速度持续低于阈值超过 3 秒，抛出 ThrottledDownload 异常
4. 如果速度回升到阈值以上，重置 throttle_start

**3 秒防抖机制的作用：** 避免瞬时速度波动导致误判。

### 2.3 ThrottledDownload 异常

位置：[utils/_utils.py#L1130-L1135](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/_utils.py#L1130-L1135)

```python
class ThrottledDownload(ReExtractInfo):
    """ Download speed below --throttled-rate. """
    msg = 'The download speed is below throttle limit'

    def __init__(self):
        super().__init__(self.msg, expected=False)
```

继承关系：`ThrottledDownload → ReExtractInfo → YoutubeDLError

**触发后的行为：**

- 属于 ReExtractInfo 类型，会触发重新提取视频信息（重新获取下载 URL，可能切换到不同的 CDN 节点以解除限流。

---

## 三、进度反馈系统

进度反馈系统负责计算下载速度、ETA，为限速和节流检测提供数据支撑，同时向用户展示进度。

### 3.1 ProgressCalculator 类

位置：[utils/progress.py](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py)

**核心参数：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| SAMPLING_WINDOW | 3 秒 | 速度计算的滑动窗口大小 |
| SAMPLING_RATE | 0.05 秒 | 采样间隔（最快 50ms） |
| GRACE_PERIOD | 1 秒 | 开始显示 ETA 的宽限期 |

**滑动窗口算法：**

```python
# 保留最近 SAMPLING_WINDOW 秒内的采样点
offset = bisect.bisect_left(self._times, current_time - self.SAMPLING_WINDOW)
del self._times[:offset]
del self._downloaded[:offset]

# 用窗口内的首尾计算速度
download_time = current_time - self._times[0]
self.speed.set((self.downloaded - self._downloaded[0]) / download_time)
```

**特点：**

- 只保留最近 3 秒的采样数据
- 计算的是窗口内的平均速度（不是瞬时速度
- 线程安全（使用 threading.Lock 保护
- 支持多线程汇总（_thread_sizes 字典按线程记录大小）

### 3.2 SmoothValue 平滑器

位置：[utils/progress.py#L96-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/utils/progress.py#L96-L109)

```python
class SmoothValue:
    def __init__(self, initial, smoothing):
        self.smooth = ...
        self._smoothing = smoothing

    def set(self, value):
        self.value = value
        if self.smooth is None:
            self.smooth = self.value
        else:
            self.smooth = (1 - self._smoothing) * value + self._smoothing * self.smooth
```

**指数平滑公式：**
- 平滑值 = (1 - α) × 新值 + α × 旧平滑值
- speed.smoothing 越大，变化越慢

**使用场景：**
- speed：smoothing=0.7（变化较慢，展示更稳定
- eta：smoothing=0.9（变化更慢，避免 ETA 不跳动

### 3.3 HttpFD 中的进度计算

位置：[downloader/http.py#L294-L310](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/http.py#L294-L310)

```python
speed = self.calc_speed(start, now, byte_counter - ctx.resume_len)
if ctx.data_len is None:
    eta = None
else:
    eta = self.calc_eta(start, time.time(), ctx.data_len - ctx.resume_len, byte_counter - ctx.resume_len)

self._hook_progress({
    'status': 'downloading',
    'downloaded_bytes': byte_counter,
    'total_bytes': ctx.data_len,
    'eta': eta,
    'speed': speed,
    'elapsed': now - ctx.start_time,
    ...
}, info_dict)
```

HttpFD 直接用简单的平均速度（从下载开始到现在），不用 ProgressCalculator。

### 3.4 FragmentFD 中的进度计算

位置：[downloader/fragment.py#L240-L279](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/fragment.py#L240-L279)

```python
progress = ProgressCalculator(resume_len)

def frag_progress_hook(s):
    ...
    progress.update(s.get('downloaded_bytes'))
    ...
    state['speed'] = progress.speed.smooth
    state['eta'] = progress.eta.smooth
    ...
    self._hook_progress(state, info_dict)
```

FragmentFD 使用 ProgressCalculator，因为分片下载涉及多个分片，需要更精确的速度计算。

### 3.5 进度钩子（Progress Hooks）

位置：[downloader/common.py#L488-L500](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L488-L500)

```python
def _hook_progress(self, status, info_dict):
    status['info_dict'] = info_dict
    for ph in self._progress_hooks:
        ph(status)

def add_progress_hook(self, ph):
    self._progress_hooks.append(ph)
```

**钩子通过 add_progress_hook 注册多个回调函数，下载过程中通过 _hook_progress 统一调用所有钩子，传递 status 字典。

内置的 progress hook 是 report_progress，负责打印进度条。

### 3.6 进度显示控制

**progress_delta 参数：

位置：[downloader/common.py#L373-L377](file:///d:/fz/0601-2/solo-dogfeeding/code/88-yt-dlp/yt_dlp/downloader/common.py#L373-L377)

```python
if update_delta := self.params.get('progress_delta'):
    with self._progress_delta_lock:
        if time.monotonic() < self._progress_delta_time:
            return
        self._progress_delta_time += update_delta
```

限制进度输出的最小时间间隔，避免输出太频繁。

---

## 四、各模块协作关系

### 4.1 数据流图

```
┌─────────────────────────────────────────────────────────────┐
│                   用户配置                          │
│  ratelimit / throttledratelimit / progress_delta   │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│            FileDownloader (基类)                    │
│  ┌──────────────┐  ┌──────────────────────────┐      │
│  │ slow_down()  │  │ report_progress()  │      │
│  │  (限速)      │  │  (进度显示)         │      │
│  └──────────────┘  └──────────────────────────┘      │
│         ▲                    ▲                    │
└─────────┼────────────────────┼────────────────────┘
          │                    │
          │                    │
    ┌─────┴─────┐          ┌────┴─────┐
    │  HttpFD   │          │FragmentFD │
    │  (单文件)│          │(分片)  │
    └─────┬─────┘          └────┬─────┘
          │                    │
          │                    │
          ▼                    ▼
    直接计算速度         ProgressCalculator
    (简单平均)         (滑动窗口+平滑)
          │                    │
          └─────────┬──────────────┘
                    │
                    ▼
          throttledratelimit 检测
          (速度持续过低 → ThrottledDownload)
```

### 4.2 限速与进度的协作时序

```
下载循环中每读取数据块
    │
    ├─→ byte_counter += len(data_block)
    │
    ├─→ slow_down()  ──── 检查是否超速，sleep 限速
    │
    ├─→ calc_speed() ──── 计算当前速度
    │
    ├─→ 节流检测 ────── 速度是否 < throttledratelimit
    │                     持续 3s → 抛 ThrottledDownload
    │
    └─→ _hook_progress() ─→ 调用所有进度钩子
                              ├─ report_progress → 打印进度条
                              └─ 其他自定义钩子
```

---

## 五、关键设计要点

### 5.1 限速的"全局"性

- **不是真正的全局带宽池：
  - 每个下载连接独立计算和控制
  - 并发分片时总带宽 = ratelimit × 并发数
- **设计取舍**：实现简单，不需要全局锁，但多线程场景不会成为性能瓶颈

### 5.2 速度计算的两种方式

| 方式 | 使用场景 | 精度 | 实现复杂度 |
|------|----------|------|------------|
| 简单平均 (HttpFD | 单文件下载 | 低（全程平均） | 低 |
| 滑动窗口 (ProgressCalculator) | 分片下载 | 高（最近 3s 平均） | 高 |

### 5.3 节流检测的防抖设计

3 秒持续低于阈值才触发，避免网络抖动误判。这是因为 CDN 刚开始下载刚开始速度可能因为 TCP 慢启动、瞬时波动等原因。

### 5.4 进度平滑处理
指数平滑让显示更稳定，避免速度/ETA 跳动太快变化太频繁变化。

---

## 六、相关文件索引

| 文件 | 作用 |
|------|------|
| [downloader/common.py | FileDownloader 基类，slow_down、report_progress |
| [downloader/http.py | HttpFD，HTTP 下载限速+节流检测 |
| [downloader/fragment.py | FragmentFD，分片下载进度整合 |
| [downloader/external.py | 外部下载器限速参数传递 |
| [utils/progress.py | ProgressCalculator、SmoothValue |
| [utils/_utils.py | ThrottledDownload 异常定义 |
| [options.py | CLI 选项定义 |
| [YoutubeDL.py | 参数汇总与传递 |
