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

所有协议下载器（HlsFD / DashSegmentsFD 等）均继承自 `FragmentFD`，复用其并发、重试、拼接逻辑。

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
