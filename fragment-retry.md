# yt-dlp 分片下载：调度、失败恢复与拼接顺序

本文档基于 yt-dlp 源码，详细解析分片（Fragment）下载的三大核心机制：
并发调度、失败重试恢复、以及分片拼接的顺序保证。

## 1. 代码结构总览

分片下载的相关模块位于 `yt_dlp/downloader/`：

| 文件 | 类 | 职责 |
|------|-----|------|
| [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py) | `FragmentFD` | 分片下载基类，实现通用调度、重试、拼接 |
| [hls.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/hls.py) | `HlsFD` | HLS (m3u8) 分片解析与下载 |
| [dash.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/dash.py) | `DashSegmentsFD` | DASH MPD 分片解析与多轨并发下载 |
| [f4m.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/f4m.py) | `F4mFD` | Adobe HDS (f4m) 分片下载 |
| [ism.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/ism.py) | `IsmFD` | Smooth Streaming (ISM) 分片下载 |
| [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/utils/_utils.py#L5242-L5296) | `RetryManager` | 通用重试管理器 |

所有协议下载器（HlsFD / DashSegmentsFD 等）均继承自 `FragmentFD`，但它们使用通用基类的方式**并不一致**——有的完全走通用调度，有的绕过通用循环直接调用底层原子方法，这是最容易混淆的根源（详见第 7 章）。

---

## 2. 分片调度与并发机制

### 2.1 核心入口

分片下载的主逻辑入口是 [FragmentFD.download_and_append_fragments](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L431-L526)。对于 DASH 这种需要同时下载音频+视频多条轨道的场景，使用 [FragmentFD.download_and_append_fragments_multiple](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L367-L429)。

### 2.2 并发度配置

并发线程数由参数 `concurrent_fragment_downloads` 控制（默认 1），在以下位置计算：

```python
# download_and_append_fragments 中单路并发度
max_workers = math.ceil(
    self.params.get('concurrent_fragment_downloads', 1) / ctx.get('max_progress', 1))
```

多路下载时（如 DASH 音视频双轨），总并发数会被平均分配给各条轨道：

```python
# download_and_append_fragments_multiple 中每条轨道的线程池
tpe = FTPE(math.ceil(max_workers / max_progress))
```

其中 `max_progress` 为轨道数量，`FTPE` 是一个自定义的 `ThreadPoolExecutor`（禁用了默认的 `__exit__` 等待行为）。

### 2.3 单线程 vs 多线程调度

`download_and_append_fragments` 根据 `max_workers` 选择两种执行路径：

**路径一：单线程（max_workers == 1）**

```python
for fragment in fragments:
    if not interrupt_trigger[0]:
        break
    try:
        download_fragment(fragment, ctx)
        result = append_fragment(
            decrypt_fragment(fragment, self._read_fragment(ctx)), fragment['frag_index'], ctx)
    except KeyboardInterrupt:
        if info_dict.get('is_live'):
            break
        raise
    if not result:
        return False
```

严格按 `fragments` 列表顺序串行执行：下载 → 解密 → 写入目标文件。

**路径二：多线程（max_workers > 1）**

```python
def _download_fragment(fragment):
    ctx_copy = ctx.copy()
    download_fragment(fragment, ctx_copy)
    return fragment, fragment['frag_index'], ctx_copy.get('fragment_filename_sanitized')

with tpe or concurrent.futures.ThreadPoolExecutor(max_workers) as pool:
    try:
        for fragment, frag_index, frag_filename in pool.map(_download_fragment, fragments):
            ctx.update({
                'fragment_filename_sanitized': frag_filename,
                'fragment_index': frag_index,
            })
            if not append_fragment(decrypt_fragment(fragment, self._read_fragment(ctx)), frag_index, ctx):
                return False
    except KeyboardInterrupt:
        # ... 中断处理
```

关键点：
1. **下载阶段并发**：`pool.map` 将 `fragments` 提交到线程池，各分片并行下载。
2. **每个线程拥有独立的 `ctx_copy`**：避免多线程竞争同一个上下文。
3. **拼接阶段串行**：`pool.map` 是**按输入迭代顺序返回结果**的迭代器——即使第 5 个分片先下载完成，也会等到第 1~4 个分片的结果 yield 之后才会被处理。这就从机制上保证了拼接顺序。

### 2.4 DASH 多轨调度

DASH 通常分离音频和视频轨道，`DashSegmentsFD.real_download` 会为每条轨道构建独立的 `(ctx, fragments, info_dict)` 参数组，然后交给 `download_and_append_fragments_multiple`：

```python
for fmt in requested_formats or [info_dict]:
    ctx = {
        'filename': fmt.get('filepath') or filename,
        'live': 'is_from_start' if fmt.get('is_from_start') else fmt.get('is_live'),
        'total_frags': fragment_count,
    }
    self._prepare_and_start_frag_download(ctx, fmt)
    fragments_to_download = self._get_fragments(fmt, ctx, extra_query)
    args.append([ctx, fragments_to_download, fmt])

return self.download_and_append_fragments_multiple(*args, is_fatal=lambda idx: idx == 0)
```

每条轨道有自己的线程池、自己的目标文件，互不干扰。`is_fatal=lambda idx: idx == 0` 表示只有第一条轨道（通常是视频）的下载失败才被视为致命错误。

---

## 3. 失败恢复与重试机制

### 3.1 RetryManager 迭代器模式

重试由 [RetryManager](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/utils/_utils.py#L5242-L5296) 实现，它是一个迭代器：

```python
class RetryManager:
    """Usage:
        for retry in RetryManager(...):
            try:
                ...
            except SomeException as err:
                retry.error = err
                continue
    """
```

- `attempt`：当前尝试次数（从 1 开始）
- `retries`：最大重试次数（`fragment_retries` 参数）
- `error_callback`：每次失败时调用，负责报告、睡眠、最终报错

核心循环逻辑：

```python
def __iter__(self):
    while self._should_retry():
        self.error = NO_DEFAULT        # 重置错误标记
        self.attempt += 1
        yield self                      # 交出控制权给业务代码
        if self.error:
            self.error_callback(self.error, self.attempt, self.retries)
```

业务代码在 `yield` 返回后如果捕获到异常，需要把异常赋给 `retry.error`，然后 `continue`。迭代器检测到有错误就调用回调并决定是否继续。

### 3.2 分片级别的重试

在 `download_fragment` 内部：

```python
def error_callback(err, count, retries):
    if fatal and count > retries:
        ctx['dest_stream'].close()
    self.report_retry(err, count, retries, frag_index, fatal)
    ctx['last_error'] = err

for retry in RetryManager(self.params.get('fragment_retries'), error_callback):
    try:
        ctx['fragment_count'] = fragment.get('fragment_count')
        if not self._download_fragment(
                ctx, fragment['url'], info_dict, headers, info_dict.get('request_data')):
            return
    except (HTTPError, IncompleteRead) as err:
        retry.error = err
        continue
    except DownloadError:  # has own retry settings
        if fatal:
            raise
```

- 捕获 `HTTPError` 和 `IncompleteRead` 后，设置 `retry.error` 并进入下一次重试。
- `DownloadError` 是 HTTP 下载器内部已经重试过后仍然失败才抛出的，此时如果是 fatal 分片就直接向上抛出。
- `fatal` 由 `is_fatal` 函数决定：默认非 fatal，但如果设置了 `skip_unavailable_fragments=False` 则全部变为 fatal。

### 3.3 .ytdl 断点续传文件

yt-dlp 用同名 `.ytdl` 文件记录下载进度，格式为 JSON：

```json
{
  "downloader": {
    "current_fragment": { "index": 5 },
    "fragment_count": 100,
    "extra_state": { ... }
  }
}
```

关键方法：
- [_read_ytdl_file](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L82-L93)：恢复 `fragment_index` 和 `extra_state`
- [_write_ytdl_file](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L95-L109)：每次 `_append_fragment` 成功后更新

在 [_prepare_frag_download](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L157-L223) 中有一致性校验逻辑：

```python
ytdl_file_exists = os.path.isfile(self.ytdl_filename(ctx['filename']))
continuedl = self.params.get('continuedl', True)
if continuedl and ytdl_file_exists:
    self._read_ytdl_file(ctx)
    is_corrupt = ctx.get('ytdl_corrupt') is True
    is_inconsistent = ctx['fragment_index'] > 0 and resume_len == 0
    if is_corrupt or is_inconsistent:
        # .ytdl 损坏或与 .part 文件不一致 → 从零开始
        ctx['fragment_index'] = resume_len = 0
        self._write_ytdl_file(ctx)
```

上层的 HlsFD / DashSegmentsFD 在枚举分片时会跳过已下载的部分：

```python
# hls.py 解析 m3u8 时
if frag_index <= ctx['fragment_index']:
    continue
```

### 3.4 分片级别的断点续传（单个分片内部）

不仅整个下载可以续传，单个分片的 HTTP 下载也支持断点续传，在 [_download_fragment](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L111-L130) 中：

```python
fragment_filename = '%s-Frag%d' % (ctx['tmpfilename'], ctx['fragment_index'])
frag_resume_len = 0
if ctx['dl'].params.get('continuedl', True):
    frag_resume_len = self.filesize_or_none(self.temp_name(fragment_filename))
fragment_info_dict['frag_resume_len'] = ctx['frag_resume_len'] = frag_resume_len

success, _ = ctx['dl'].download(fragment_filename, fragment_info_dict)
```

每个分片有独立的临时文件 `<tmpfilename>-Frag<N>`，如果下载中断，下次可以从已有的字节继续下载。

---

## 4. 分片拼接顺序

### 4.1 拼接写入流程

拼接的核心方法是 [_append_fragment](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L146-L155)：

```python
def _append_fragment(self, ctx, frag_content):
    try:
        ctx['dest_stream'].write(frag_content)
        ctx['dest_stream'].flush()
    finally:
        if self.__do_ytdl_file(ctx):
            self._write_ytdl_file(ctx)          # 更新进度
        if not self.params.get('keep_fragments', False):
            self.try_remove(ctx['fragment_filename_sanitized'])  # 删除临时分片文件
        del ctx['fragment_filename_sanitized']
```

每成功写入一个分片，立即：
1. `flush()` 确保数据落盘
2. 更新 `.ytdl` 文件记录当前进度
3. （默认）删除该分片的临时文件

### 4.2 多线程下的顺序保证

如 2.3 节所述，多线程下载使用 `ThreadPoolExecutor.map`，它的关键特性是：
> The returned iterator raises a `concurrent.futures.TimeoutError` if `__next__()` is called and the result isn't available after *timeout* seconds from the original call to `Executor.map()`.
> **Results are returned in the order corresponding to the iterable arguments.**

也就是说，`pool.map(_download_fragment, fragments)` 返回的迭代器严格按照 `fragments` 的输入顺序产出结果。即使第 N 个分片先下载完毕，主线程也会阻塞等待前 N-1 个分片的结果被消费之后，才会处理它。

这是设计上最精妙的一点——**下载并发，但写入串行有序**。

### 4.3 不同协议的拼接差异

**HLS（m3u8）**：
- 普通媒体分片：直接 append 二进制数据即可，MPEG-TS 格式天然支持拼接
- AES-128 加密分片：在 [decrypter](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L341-L365) 中解密后再 append
- WebVTT 字幕：通过 `pack_func` 参数对每个分片进行时间轴对齐、去重后再拼接

**DASH**：
- 每条轨道（音频/视频）各自独立下载到自己的目标文件
- `is_fatal=lambda idx: idx == 0` 保证视频轨道失败才终止，音频等次要轨道缺失可以容忍
- 后续由 FFmpeg 后处理器将多轨文件合并

**F4M（Adobe HDS）**：
- 每个分片是一个 FLV box，需要解析提取 `mdat` box 的 payload 后写入
- 开头需要先写入 FLV header 和 metadata tag

**ISM（Smooth Streaming）**：
- 第一个分片需要提取 `tfhd` box 中的 track_id，生成 PIFF/MP4 header 后写入
- 后续分片直接追加原始内容

### 4.4 拼接前的可选处理钩子

`download_and_append_fragments` 提供了两个可选参数用于定制拼接行为：

```python
def download_and_append_fragments(
        self, ctx, fragments, info_dict, *, is_fatal=(lambda idx: False),
        pack_func=(lambda content, idx: content), finish_func=None,
        tpe=None, interrupt_trigger=(True, )):
```

- `pack_func(content, idx)`：在 `append_fragment` 之前对分片内容做变换（HLS WebVTT 用它做字幕时间轴对齐）
- `finish_func()`：所有分片拼接完毕后调用，写入尾部数据（如 WebVTT 残余的去重窗口内容）

---

## 5. 完整流程图（单路多线程场景）

```
HlsFD.real_download() / DashSegmentsFD.real_download()
  │
  ├─ 解析 manifest → 生成 fragments 列表（每个带 frag_index）
  │
  ├─ _prepare_frag_download(ctx)
  │    ├─ 打开 tmpfilename (.part) 为 dest_stream
  │    ├─ 读取 .ytdl 恢复 fragment_index
  │    └─ 跳过已下载的分片
  │
  └─ download_and_append_fragments(ctx, fragments, info_dict)
       │
       ├─ max_workers > 1 ?
       │    │
       │    ├─ YES: ThreadPoolExecutor
       │    │    ├─ pool.map(_download_fragment, fragments)
       │    │    │    ├─ Worker 1: _download_fragment(frag[3]) → 下载到 .part-Frag3
       │    │    │    ├─ Worker 2: _download_fragment(frag[1]) → 下载到 .part-Frag1
       │    │    │    └─ Worker 3: _download_fragment(frag[2]) → 下载到 .part-Frag2
       │    │    │
       │    │    └─ 主线程按顺序消费:
       │    │         ├─ yield (frag[1]) → decrypt → append → flush → write .ytdl → rm .part-Frag1
       │    │         ├─ yield (frag[2]) → decrypt → append → flush → write .ytdl → rm .part-Frag2
       │    │         └─ yield (frag[3]) → decrypt → append → flush → write .ytdl → rm .part-Frag3
       │    │
       │    └─ NO: 串行 for 循环
       │         └─ 逐个 download → decrypt → append → flush → write .ytdl → rm
       │
       └─ _finish_frag_download()
            ├─ 关闭 dest_stream
            ├─ 删除 .ytdl 文件
            └─ .part → 重命名为最终文件名
```

---

## 6. 关键参数总结

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `fragment_retries` | CLI: 10, API: 0 | 单个分片的 HTTP 错误最大重试次数 |
| `concurrent_fragment_downloads` | 1 | 分片并发下载线程数 |
| `skip_unavailable_fragments` | True | 是否跳过不可用的分片（False 则任一分片失败即终止） |
| `keep_fragments` | False | 是否保留已下载的分片临时文件 |
| `continuedl` | True | 是否启用断点续传（.ytdl + 分片级 .part-FragN） |
| `_no_ytdl_file` | False | 禁用 .ytdl 进度记录（直播等场景） |

---

## 7. 通用路径 vs 协议特定路径：混淆点深度解析

阅读分片代码时最容易产生的误解是"所有下载器都走 `download_and_append_fragments`"。实际情况是：**4 个协议下载器采用了 3 种不同的调度架构**，只有 HLS 和 DASH 完全走通用调度，F4m 和 ISM 则绕过了通用循环。

### 7.1 FragmentFD 的三层能力分层

先看 `FragmentFD` 提供了哪些可复用的能力，按抽象层级分三层：

| 层级 | 方法 | 能力 | 是否可被协议层绕过 |
|------|------|------|-------------------|
| **L3 高层调度** | `download_and_append_fragments` / `download_and_append_fragments_multiple` | 并发线程池 + RetryManager + 解密 + 顺序拼接一体化 | 是（F4m / ISM 绕过） |
| **L2 中层流程控制** | `_prepare_frag_download` / `_start_frag_download` / `_finish_frag_download` | 打开 .part 文件、恢复 .ytdl、进度 hook、关闭/重命名 | 否（全部协议都调用） |
| **L1 底层原子操作** | `_download_fragment` / `_read_fragment` / `_append_fragment` | 单个分片的 HTTP 下载、读回、写入目标文件 | 否（全部协议都调用） |

```
FragmentFD 能力金字塔：

          ┌─────────────────────────────┐
          │  L3: download_and_append_   │
          │      fragments[_multiple]   │  ← HlsFD, DashSegmentsFD 使用
          │  (线程池 + 重试 + 解密 +    │
          │   pack_func + 顺序保证)     │
          ├─────────────────────────────┤
          │  L2: _prepare/_start/_      │
          │      finish_frag_download   │  ← 所有协议必须使用
          │  (.ytdl恢复, dest_stream,   │
          │   进度hook, 重命名)          │
          ├─────────────────────────────┤
          │  L1: _download_fragment /   │
          │      _read_fragment /       │  ← 所有协议必须使用
          │      _append_fragment       │
          │  (单分片HTTP, 读, 写+flush) │
          └─────────────────────────────┘
```

### 7.2 四协议调度架构横向对比

下面用表格 + 调用图逐一说明每个协议下载器对三层能力的使用方式：

| 协议 | real_download 位置 | L3 高层调度 | L2 中层控制 | L1 原子操作 | 自有 RetryManager | 自有循环 |
|------|-------------------|------------|------------|------------|-----------------|---------|
| **HLS** | [hls.py:74](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/hls.py#L74-L409) | ✅ `download_and_append_fragments` | ✅ `_prepare_and_start_frag_download` | ✅ 由 L3 自动调用 | ❌（由 L3 提供） | ❌ |
| **DASH** | [dash.py:17](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/dash.py#L17-L68) | ✅ `download_and_append_fragments_multiple` | ✅ `_prepare_and_start_frag_download` | ✅ 由 L3 自动调用 | ❌（由 L3 提供） | ❌ |
| **F4m** | [f4m.py:309](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/f4m.py#L309-L427) | ❌ **绕过** | ✅ 分别调用 `_prepare_frag_download` 和 `_start_frag_download` | ✅ 循环内直接调用 | ⚠️ **没有分片级 RetryManager**（仅直播 404 特判） | ✅ `while fragments_list` |
| **ISM** | [ism.py:236](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/ism.py#L236-L283) | ❌ **绕过** | ✅ `_prepare_and_start_frag_download` | ✅ 循环内直接调用 | ✅ 循环内自建 `RetryManager` | ✅ `for segment in segments` |

#### HlsFD 的调用路径（最干净的通用路径）

```
HlsFD.real_download (hls.py:74)
  │
  ├─ 【协议特定】下载/解析 m3u8 manifest
  │     ├─ 识别 #EXT-X-KEY, #EXT-X-BYTERANGE, #EXT-X-MAP 等标签
  │     ├─ 跳过广告分片 (ANVATO-SEGMENT-INFO, UPLYNK-SEGMENT)
  │     └─ 构造带字段的 fragments[]: {frag_index, url, decrypt_info, byte_range, media_sequence}
  │
  ├─ 【通用 L2】_prepare_and_start_frag_download(ctx, info_dict)
  │     └─ 打开 .part, 读 .ytdl 恢复进度
  │
  └─ 【通用 L3】download_and_append_fragments(ctx, fragments, info_dict)
        │     ← 或传入 pack_func=pack_fragment (WebVTT场景)
        │
        ├─ 【通用 L3 内部】线程池并发 + RetryManager
        │     ├─ download_fragment() → _download_fragment (L1)
        │     ├─ decrypt_fragment()   ← 【协议特定钩子】AES-128 解密
        │     │                        (decrypt_info 是从 m3u8 解析出来挂在 fragment 上的)
        │     └─ append_fragment() → _append_fragment (L1)
        │
        └─ 【通用 L2 内部】_finish_frag_download
```

**拼接边界**：`decrypt_info` 是 HLS 协议层解析出来挂到 `fragment` dict 上的，解密逻辑由通用 `decrypter()` 统一调用，但只有当 `fragment['decrypt_info']['METHOD'] == 'AES-128'` 时才生效——相当于通用层提供了解密"槽位"，协议层填充配置。

#### DashSegmentsFD 的调用路径（多轨通用路径）

```
DashSegmentsFD.real_download (dash.py:17)
  │
  ├─ 【协议特定】遍历 requested_formats (音视频多轨)
  │     ├─ _get_fragments() 解析 fmt['fragments']
  │     │     └─ 组装 {frag_index, index, url, fragment_count}
  │     └─ 每轨一个 (ctx, fragments, fmt) 三元组
  │
  ├─ 【通用 L2】每条轨道各自调用 _prepare_and_start_frag_download
  │
  └─ 【通用 L3】download_and_append_fragments_multiple(*args, is_fatal=lambda idx: idx == 0)
        │
        ├─ 【通用 L3 内部】每轨道创建 FTPE 线程池
        │     └─ 每轨道进入 download_and_append_fragments → 与 HLS 相同的通用流程
        │
        └─ 【协议特定参数】is_fatal=lambda idx: idx == 0
              ↳ 只有第 0 条轨道（视频）失败才算致命，音频轨道可以容忍缺失
```

**拼接边界**：DASH 的拼接边界比 HLS 更"干净"——它不使用 `decrypt_info`，分片内容本身是完整的 fMP4 segment（包含 moof+mdat），直接按顺序 append 即可。协议差异完全体现在：① 多轨结构上，② `fragments[]` 的字段来源上。

#### F4mFD 的调用路径（绕过 L3，自建循环 + 无分片级重试）

这是最容易踩坑的一个。它**没有调用 `download_and_append_fragments`**：

```
F4mFD.real_download (f4m.py:309)
  │
  ├─ 【协议特定】下载/解析 f4m XML manifest
  │     ├─ FlvReader.read_bootstrap_info() 解析 abst/asrt/afrt box
  │     ├─ build_fragments_list() → [(seg_i, frag_i), ...]
  │     └─ 选择合适的 media (按 bitrate)
  │
  ├─ 【通用 L2】_prepare_frag_download(ctx)  ← 注意：单独调，不是 _prepare_and_start_
  │
  ├─ 【协议特定】写入 FLV header 和 metadata tag
  │     write_flv_header(dest_stream)
  │     write_metadata_tag(dest_stream, metadata)
  │     ← 这就是为什么它必须拆开 _prepare 和 _start —— 中间要插东西！
  │
  ├─ 【通用 L2】_start_frag_download(ctx, info_dict)  ← 启动进度 hook
  │
  └─ 【协议特定自建循环】while fragments_list:
        │
        ├─ 【协议特定】构造 URL: base_url + "Seg%d-Frag%d" % (seg_i, frag_i)
        │
        ├─ 【通用 L1】_download_fragment(ctx, url, info_dict)
        │     ← 注意：外面没有包 RetryManager！HTTPError 会直接抛出
        │
        ├─ 【通用 L1】_read_fragment(ctx) → 读取 .part-FragN 到内存
        │
        ├─ 【协议特定】FlvReader.read_box_info() 循环解析 box
        │     └─ 找到 box_type == b'mdat' → 提取 payload
        │     ← 这是 F4m 特有拼接边界：不是整个分片内容写入，
        │        而是从 FLV box 中提取 mdat 后再写
        │
        ├─ 【通用 L1】_append_fragment(ctx, box_data)  ← 只写 mdat
        │
        ├─ 【协议特定异常分支】直播场景下：
        │     except HTTPError as err:
        │         if live and err.status in (404, 410):
        │             fragments_list = []  # 静默跳过
        │
        └─ 【协议特定】直播刷新分片列表
              _update_live_fragments() → 重新请求 bootstrap info
```

**关键差异——重试**：`F4mFD` 的 `_download_fragment` 调用是"裸"的，外层没有 `for retry in RetryManager(...)` 包裹。这意味着分片下载的重试完全依赖 HTTP 层 `HttpFD` 的 `retries` 参数，而**不是 `fragment_retries` 参数**。直播 404/410 则被特判吞掉，计入 `.ytdl` 进度但不会重试。

**拼接边界**：FLV 格式要求文件头必须放在最前面，但 `_prepare_frag_download` 只是打开了 dest_stream，还没开始写任何内容。F4mFD 抓住这个"窗口期"写入 `FLV header + metadata tag`，然后才调用 `_start_frag_download`。之后每个分片的原始 FLV box 被解包，只提取 `mdat`（媒体数据）payload 通过通用 L1 追加。

#### IsmFD 的调用路径（绕过 L3，自建循环 + 自有 RetryManager）

```
IsmFD.real_download (ism.py:236)
  │
  ├─ 【协议特定】info_dict['fragments'] 已由 extractor 准备好
  │
  ├─ 【通用 L2】_prepare_and_start_frag_download(ctx, info_dict)
  │
  └─ 【协议特定自建循环】for segment in segments:
        │
        ├─ 【协议特定 + 通用】IsmFD 自建 RetryManager:
        │     retry_manager = RetryManager(
        │         self.params.get('fragment_retries'),
        │         self.report_retry,
        │         frag_index=frag_index,
        │         fatal=not skip_unavailable_fragments)
        │
        └─ for retry in retry_manager:
              try:
                  ├─ 【通用 L1】_download_fragment(ctx, url, info_dict)
                  ├─ 【通用 L1】_read_fragment(ctx)
                  │
                  ├─ 【协议特定】首个分片处理：
                  │     if not extra_state['ism_track_written']:
                  │         tfhd_data = extract_box_data(frag_content, [b'moof', b'traf', b'tfhd'])
                  │         track_id = u32.unpack(tfhd_data[4:8])[0]
                  │         write_piff_header(dest_stream, ...)  ← 写 ftyp+moov+mvex
                  │
                  └─ 【通用 L1】_append_fragment(ctx, frag_content)
              except HTTPError as err:
                  retry.error = err
                  continue

   循环结束后：
        └─ 【协议特定】retry_manager.error → 若 skip_unavailable_fragments 则跳过，否则失败
```

**关键差异——重试**：`IsmFD` 自行创建 `RetryManager`，参数与通用 L3 的完全一致（`fragment_retries`、`frag_index`、`fatal`），所以它的重试语义与 HLS/DASH 的 L3 提供的等价。但**并发**功能没有——它是纯串行 for 循环，`concurrent_fragment_downloads` 对 ISM 协议无效。

**拼接边界**：与 F4m 类似，ISM 也需要一个"协议特定文件头"（PIFF/MP4 的 ftyp+moov）。但它的写入时机选择了另一种方式——不拆开 `_prepare_and_start_frag_download`，而是在循环第一次迭代、第一个分片下载完毕后，从第一个分片的内容中提取 `track_id`，动态生成 PIFF header 后直接写入 dest_stream（注意**不是通过 `_append_fragment`**，而是直接写），然后才开始正常的 `_append_fragment` 流程。

```
拼接时机对比：

 F4mFD:  _prepare → 【write FLV header】 → _start → 循环(解box→取mdat→append)

 IsmFD:  _prepare_and_start → 循环开始
             ↳ frag[0] 下载完毕
                 → 【extract tfhd → write PIFF header (直接写dest_stream)】
                 → append frag[0] 原文
             ↳ frag[N] 直接 append
```

### 7.3 拼接边界的四种模式总结

从以上四个协议可以归纳出，"协议特定数据"写入目标文件的位置有四种模式：

| 模式 | 代表协议 | 实现方式 | 是否经过 `_append_fragment` | .ytdl 一致性 |
|------|---------|---------|---------------------------|-------------|
| **A. 前序写入（在 _start 之前）** | F4m | `_prepare_frag_download` → 直接写 dest_stream → `_start_frag_download` | ❌ 直接写 | 写入时 .ytdl 的 fragment_index 仍是 0，一致 |
| **B. 懒写入（首分片下载后）** | ISM | 首分片下载后直接写 dest_stream → 再 `_append_fragment` 首分片原文 | ❌ 头直接写 / ✅ 分片用 append | 头写入时 fragment_index=0，与首分片一起完成 |
| **C. 通过 pack_func 逐分片变换** | HLS WebVTT | 传入 `pack_func` 参数，通用 L3 在 append 前调用 | ✅ 经 pack_func 包装后仍走 `_append_fragment` | 完美一致，`_append_fragment` 内部统一更新 .ytdl |
| **D. 内容原样直接追加** | HLS TS / DASH fMP4 | 通用流程，不做任何变换 | ✅ | 完美一致 |

**关键洞见**：当协议层需要"在第 1 个分片前写入协议特定 header"时，有两条路可选：
- **F4m 路**：拆开 `_prepare_and_start_frag_download`，在中间插入 header 写入（要求 header 内容在下载前已知）
- **Ism 路**：保持组合调用不变，在首分片回调中提取信息后写入（header 内容依赖首分片数据，如 ISM 的 track_id）

**⚠️ 这两种路径的断点恢复安全性截然不同**：F4m 是安全的，ISM 存在窄窗口风险。详见 7.5 节深度分析。

### 7.4 重试路径的三种模式总结

同样，分片级重试的实现也分三种：

| 模式 | 代表协议 | 重试控制者 | 重试参数 | 跳过不可用分片 |
|------|---------|-----------|---------|--------------|
| **I. 通用 L3 内建 RetryManager** | HLS, DASH | `download_and_append_fragments` 内部 | `fragment_retries`（0/10） | ✅ 通过 `is_fatal` 参数控制 |
| **II. 协议层自建 RetryManager** | ISM | `IsmFD.real_download` 循环内 | `fragment_retries`（同参数名） | ✅ `skip_unavailable_fragments` + `retry_manager.error` |
| **III. 裸调用（无分片级 RetryManager）** | F4m | —— | —— （依赖 HTTP 层 retries） | ⚠️ 只有直播 404/410 特判跳过，其他 HTTPError 直接抛出 |

**注意**：模式 III（F4m）是**语义不一致**的——对 F4m 协议设置 `fragment_retries=10` 不会生效，因为它根本没走包含 RetryManager 的代码路径。只有 HTTP 下载器内部的 `retries` 参数（`retries` 不是 `fragment_retries`）才对它生效。这是历史遗留差异，排查重试问题时需要特别注意。

---

## 7.5 协议头写入、进度更新与断点恢复的边界深度分析

这一节从代码时序层面精确梳理：协议头是"裸写" `dest_stream` 的，不走 `_append_fragment`，也不走 `_write_ytdl_file`——这与 `.part` 文件大小、`fragment_index` 内存值、`.ytdl` 磁盘值三者之间存在微妙的时间差，是断点恢复 bug 的温床。

### 7.5.1 前置知识：两套 fragment_index 更新机制

在深入分析之前，必须先区分三个容易混淆的"进度"概念：

| 概念 | 存储位置 | 更新时机 | 更新者 | 作用 |
|------|---------|---------|-------|------|
| `ctx['fragment_index']` | 内存 | 每次分片下载完成（进度 hook `status='finished'` 时） | `frag_progress_hook` [fragment.py:270-L273](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L270-L273) | 构造分片临时文件名、估算总大小 |
| `.ytdl` 中 `current_fragment.index` | 磁盘文件 | 每次 `_append_fragment` 成功后（finally 块中） | `_write_ytdl_file` [fragment.py:151-L152](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L151-L152) | 断点恢复时跳过已完成分片 |
| `complete_frags_downloaded_bytes` | 内存 | 每次分片下载完成（进度 hook 中） | `frag_progress_hook` [fragment.py:275](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L275) | 进度估算 |

> **注意**：`complete_frags_downloaded_bytes` 字面上是"已完成分片的字节数"，但它的初始值是 `resume_len = .part 文件的原始大小`，**包含协议头字节**——即使协议头不属于任何分片。这是后续边界问题的根源之一。

关键时间差：
- **`ctx['fragment_index']`（内存）先更新**：下载完成的瞬间，进度 hook 就加 1
- **`.ytdl`（磁盘）后更新**：要等 `_append_fragment` 完成后，在 finally 中才写入

两者之间存在一个窗口：分片已下载完成，但还没写入目标文件。

### 7.5.2 open_mode 的选择逻辑

`_prepare_frag_download` 中决定文件打开模式 [fragment.py:173-L178](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L173-L178)：

```python
resume_len = self.filesize_or_none(tmpfilename)
if resume_len > 0:
    open_mode = 'ab'   # 追加：文件已有内容，从尾部继续写
else:
    open_mode = 'wb'   # 覆盖：文件为空，从头写
```

判断依据只有一个：`.part` 文件是否有字节。只要有字节（哪怕只是协议头），就是 `'ab'` 追加模式。

### 7.5.3 F4m 的边界分析（前序写入模式）—— 安全

F4m 的写入时序 [f4m.py:361-L372](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/f4m.py#L361-L372)：

```python
self._prepare_frag_download(ctx)      # 打开 .part，open_mode 根据 resume_len 决定
dest_stream = ctx['dest_stream']

# 用 complete_frags_downloaded_bytes == 0 作为 header 写入判断
if ctx['complete_frags_downloaded_bytes'] == 0:
    write_flv_header(dest_stream)     # 直接写 dest_stream，不走 _append_fragment
    write_metadata_tag(dest_stream, metadata)

self._start_frag_download(ctx, info_dict)  # 启动进度 hook
```

**断点恢复场景分析：**

| 中断时机 | .part 内容 | `.ytdl` fragment_index | resume_len | open_mode | `complete_frags_downloaded_bytes == 0`? | 恢复时行为 | 结果 |
|---------|-----------|----------------------|------------|-----------|----------------------------------------|-----------|------|
| **写 header 前** | 空 | 0 | 0 | `'wb'` | ✅ 是 | 重写 header，从 frag 1 开始 | ✅ 正确 |
| **header 写完，frag 1 未 append** | FLV header + metadata | **0**（还没调过 `_append_fragment`） | header_size | `'ab'` | ❌ 否 | **不重写 header**，从 frag 1 开始 | ✅ 正确 |
| **frag 1 已 append** | header + meta + frag1 | 1 | header + meta + frag1 | `'ab'` | ❌ 否 | 跳过 frag 1，从 frag 2 开始 | ✅ 正确 |

**F4m 为什么安全？** 关键在于它用 `complete_frags_downloaded_bytes == 0`（即 `.part` 文件大小）作为 header 写入的判断条件，而不是用 `fragment_index`。只要 `.part` 有任何字节（哪怕只有 header），就不会重写 header。`'ab'` 追加模式也保证了不会覆盖已有内容。

### 7.5.4 ISM 的边界分析（懒写入模式）—— 存在窄窗口风险

ISM 必须从第一个分片的 `tfhd` box 中提取 `track_id` 才能构建 PIFF header，因此不得不推迟到第一个分片下载后再写。这引入了一个未持久化窗口。

让我们逐行跟踪 ISM 第一个分片的完整时序 [ism.py:262-L273](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/ism.py#L262-L273)：

```python
# 初始状态：
#   ctx['fragment_index'] = 0  (内存)
#   .ytdl fragment_index = 0   (磁盘)
#   extra_state.ism_track_written = False  (内存 & 磁盘)
#   .part 文件: 空

try:
    success = self._download_fragment(ctx, segment['url'], info_dict)
    # ┌──────────────────────────────────────────────┐
    # │  _download_fragment 内部:                     │
    # │  1. 文件名 = tmpfilename + "-Frag0"          │
    # │     (用 ctx['fragment_index'] = 0 构造)      │
    # │  2. ctx['dl'].download() → HttpFD 下载        │
    # │  3. 下载完成，进度 hook 触发 status='finished'│
    # │     → ctx['fragment_index'] += 1 → 变成 1   │
    # │     【注意：内存更新了，磁盘还没动】           │
    # │  4. ctx['fragment_filename_sanitized'] = ... │
    # └──────────────────────────────────────────────┘
    # 此时：ctx['fragment_index'] = 1 (内存)
    #       .ytdl fragment_index 仍为 0 (磁盘)

    frag_content = self._read_fragment(ctx)

    if not extra_state['ism_track_written']:   # True (内存中为 False)
        tfhd_data = extract_box_data(frag_content, [b'moof', b'traf', b'tfhd'])
        info_dict['_download_params']['track_id'] = u32.unpack(tfhd_data[4:8])[0]
        write_piff_header(ctx['dest_stream'], info_dict['_download_params'])
        # 直接写 dest_stream → .part 现在有 PIFF header (~1KB)
        extra_state['ism_track_written'] = True
        # 【注意：仅内存，.ytdl 中仍是 False】

    # ▲─────── 窄窗口位置 ───────▲
    # 此时状态快照：
    #   .part 文件: PIFF header
    #   ctx['fragment_index'] = 1      (内存)
    #   .ytdl fragment_index = 0       (磁盘)
    #   extra_state.ism_track_written = True   (内存)
    #   extra_state.ism_track_written = False  (磁盘 .ytdl 中)
    #
    # 如果在此处中断 → 恢复时会重写 PIFF header → 文件损坏

    self._append_fragment(ctx, frag_content)
    # ┌──────────────────────────────────────────────┐
    # │  _append_fragment 内部:                       │
    # │  1. dest_stream.write(frag_content)          │
    # │  2. dest_stream.flush()                      │
    # │  3. finally:                                  │
    # │     _write_ytdl_file(ctx)                     │
    # │       → 写入 ctx['fragment_index'] = 1       │
    # │       → 写入 extra_state.ism_track_written=True │
    # │     【磁盘状态终于追上内存】                   │
    # └──────────────────────────────────────────────┘
```

**断点恢复场景分析（窄窗口中断）：**

| 中断时机 | .part 内容 | `.ytdl` fragment_index | `.ytdl` ism_track_written | resume_len | open_mode | 恢复时行为 | 结果 |
|---------|-----------|----------------------|--------------------------|------------|-----------|-----------|------|
| **写 PIFF header 前** | 空 | 0 | False | 0 | `'wb'` | 写 header，append frag[0] | ✅ 正确 |
| **PIFF header 写完，frag[0] 未 append**（窄窗口） | PIFF header | **0** | **False** | ~1KB | `'ab'` | `ism_track_written=False` → **再写一遍 PIFF header**；然后 append frag[0] | ❌ **文件损坏**：两份 header 堆叠 |
| **frag[0] 已 append** | PIFF header + frag[0] | 1 | True | header + frag0 | `'ab'` | 跳过 frag[0]，从 frag[1] 开始 | ✅ 正确 |

**为什么会损坏？** 三重巧合叠加：
1. **`'ab'` 追加模式**：因为 `.part` 已有 PIFF header 字节，`resume_len > 0` → 用追加模式打开
2. **`ism_track_written` 未持久化**：状态只在内存中，`.ytdl` 里还是 `False`
3. **判断条件是内存状态**：`if not extra_state['ism_track_written']` 使用的是从 `.ytdl` 恢复的值

三者共同作用，导致恢复时用 `'ab'` 模式又追加了一份 PIFF header，MP4 文件结构损坏。

### 7.5.5 一致性检查的盲区

`_prepare_frag_download` 中有一个一致性检查 [fragment.py:204-L211](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py#L204-L211)：

```python
is_inconsistent = ctx['fragment_index'] > 0 and resume_len == 0
# 即：【有分片进度，但文件为空】→ 清零重下
if is_inconsistent:
    ctx['fragment_index'] = resume_len = 0
    self._write_ytdl_file(ctx)
```

这个检查只覆盖了一种不一致。反过来的情况：

> **`fragment_index == 0` 但 `resume_len > 0`**

是**故意不检查**的——因为协议头写入就是在 fragment_index=0 时进行的，这种情况是合法的。

但这也意味着：
- F4m 的 header 写完后中断 → 合法且安全（有 `complete_frags_downloaded_bytes == 0` 保护）
- ISM 窄窗口中断 → 合法但**有语义不一致**（`ism_track_written` 状态没同步）→ 会导致重写 header

### 7.5.6 理论修复方向

如果要修复 ISM 的窄窗口问题，只需在 `write_piff_header` 之后立即持久化一次 `.ytdl`：

```python
write_piff_header(ctx['dest_stream'], info_dict['_download_params'])
extra_state['ism_track_written'] = True
# 立即持久化 ism_track_written 状态，消除窄窗口
if self.__do_ytdl_file(ctx):
    self._write_ytdl_file(ctx)
```

当前代码未这样做。这是一个存在但低概率的 bug——只有在恰好写完 PIFF header 后、`_append_fragment` 之前发生中断才会触发。

### 7.5.7 ISM 拼接边界风险摘要

本摘要提炼 ISM 协议在拼接边界上的核心风险点，便于快速排查。

**一、三层状态与更新时序（首分片场景）**

沿时间轴从左到右，三层状态的更新不同步：

```
时间 →  _download_fragment            write_piff_header    _append_fragment
        (下载 & 进度hook)             (直接写 dest_stream)   (写+flush+.ytdl)
         │                              │                      │
内存     │  ctx['fragment_index']: 0→1  │  ism_track_written   │  不变
ctx      │                              │  : False → True       │
         │                              │                      │
.ytdl    │  不变（仍为 0）              │  不变（仍为 False）  │  追上内存
磁盘     │                              │                      │  fragment_index=1
         │                              │                      │  ism_track_written=True
         │                              │                      │
.part    │  不变（仍为空）              │  写入 PIFF header    │  追加首分片内容
文件     │                              │  （~1KB ftyp+moov）  │
         │                              │                      │
         ◄────── 窄窗口 ────────────────►
         状态不一致区域：文件有 header，但 .ytdl 说没写过
```

**二、窄窗口的精确边界**

窄窗口起点：`write_piff_header()` 调用完成后
窄窗口终点：`_append_fragment()` 内部 `finally: _write_ytdl_file()` 执行后

窗口内状态快照：
| 状态层 | 值 | 含义 |
|-------|----|------|
| `.part` 文件 | 有 PIFF header 字节 | 文件头已写入 |
| `.ytdl` fragment_index | `0` | 磁盘认为 0 个分片完成 |
| `.ytdl` ism_track_written | `false` | 磁盘认为 header 没写过 |
| 内存 ism_track_written | `True` | 内存知道 header 已写 |
| 内存 fragment_index | `1` | 内存知道 1 个分片下载完 |

**三、风险触发三要素**

必须同时满足才会导致文件损坏：
1. **中断时机**：恰好落在窄窗口内（写完 PIFF header 后、`_append_fragment` 完成前）
2. **恢复判断**：`if not extra_state['ism_track_written']` → 从 `.ytdl` 恢复得到 `False` → 判定"还没写 header"
3. **写入模式**：`resume_len > 0` → `open_mode = 'ab'` 追加模式 → 新 header 被追加到文件尾部

结果：文件中有两份 PIFF header 前后堆叠，MP4 解析失败。

**四、与 F4m 的关键差异**

| 对比维度 | F4m | ISM |
|---------|-----|-----|
| Header 写入时机 | `_prepare` 后、`_start` 前 | 首分片下载后 |
| 是否需要分片内容 | 否（manifest 中有所有信息） | 是（需从 `tfhd` box 提取 `track_id`） |
| "是否写过 header" 的判断依据 | `complete_frags_downloaded_bytes == 0`（看文件大小） | `not extra_state['ism_track_written']`（看内存状态） |
| 判断依据的持久化 | 隐式持久化在 `.part` 文件大小里 | 需显式持久化到 `.ytdl` 的 `extra_state` |
| 窄窗口风险 | 无 | 有 |
| 原因 | 文件大小 = header 是否已写，天然一致 | 内存状态需手动刷盘，存在时间差 |

**五、风险等级**

- **发生概率**：低。窄窗口只有几十到几百微秒（`write_piff_header` 写 ~1KB 到内核缓存 + 一次内存赋值），且必须恰好在这个窗口中断。
- **影响程度**：高。一旦触发，输出文件损坏且静默（不会报错，直到播放时才发现）。
- **修复成本**：低。在 `write_piff_header` 后加一次 `_write_ytdl_file` 即可消除窗口。

**六、代码定位速查表**

| 关注点 | 文件 | 行号 |
|-------|------|------|
| ISM 主循环入口 | [ism.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/ism.py) | L236 |
| PIFF header 写入点 | [ism.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/ism.py) | L271 |
| ism_track_written 内存标记 | [ism.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/ism.py) | L272 |
| _append_fragment 调用 | [ism.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/ism.py) | L273 |
| _append_fragment 内部写 .ytdl | [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py) | L151-L152 |
| _write_ytdl_file 写 extra_state | [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py) | L103-L104 |
| _read_ytdl_file 恢复 extra_state | [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/86-yt-dlp/yt_dlp/downloader/fragment.py) | L88-L89 |
