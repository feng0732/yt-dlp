# yt-dlp 断点续传机制代码分析

## 概述

yt-dlp 的断点续传（Resume Download）分为两种主要实现路径：

1. **普通 HTTP 下载**（`HttpFD`）：基于 Range 请求 + `.part` 临时文件大小
2. **分段下载**（`FragmentFD` 及其子类 `HlsFD`/`DashSegmentsFD`）：基于 `.part` 临时文件 + `.ytdl` 状态元数据文件

判断"是否可恢复"并非单一逻辑，而是通过**文件系统探测**、**HTTP 协议协商**、**状态文件校验**三道关卡综合判定。

---

## 一、临时文件机制

### 1. `.part` 文件——下载中的数据载体

所有下载（除标准输出场景）默认使用 `.part` 后缀的临时文件。相关逻辑位于 [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/common.py#L217-L230)：

```python
def temp_name(self, filename):
    if self.params.get('nopart', False) or filename == '-' or \
            (os.path.exists(filename) and not os.path.isfile(filename)):
        return filename
    return filename + '.part'

def undo_temp_name(self, filename):
    if filename.endswith('.part'):
        return filename[:-len('.part')]
    return filename

def ytdl_filename(self, filename):
    return filename + '.ytdl'
```

核心规则：
- `nopart` 参数可禁用 `.part`，直接写目标文件
- 文件名 `-`（代表 stdout）不生成临时文件
- 若目标路径已存在且不是普通文件，也不追加 `.part`

### 2. `.ytdl` 文件——分段下载的状态账本

仅用于 `FragmentFD`（HLS/DASH/f4m 等协议）。格式为 JSON，定义见 [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/fragment.py#L41-L60)：

```json
{
  "downloader": {
    "current_fragment": {
      "index": 0
    },
    "fragment_count": 100,
    "extra_state": { ... }
  }
}
```

- `current_fragment.index`：**下一个待下载**的片段索引（0-based，详见后文"语义辨析"）
- `fragment_count`：总片段数
- `extra_state`：扩展状态（如 WebVTT 字幕的时间戳校准、去重窗口等）

---

## 二、普通 HTTP 下载的可恢复判定（HttpFD）

核心代码位于 [http.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/http.py#L24-L193) 的 `establish_connection()` 内部。

### 第一步：探测本地已有字节（ctx.resume_len）

```python
# http.py L59-L64
if self.params.get('continuedl', True):
    if os.path.isfile(ctx.tmpfilename):
        ctx.resume_len = os.path.getsize(ctx.tmpfilename)

ctx.is_resume = ctx.resume_len > 0
```

- **前提**：`continuedl` 参数为 `True`（默认开）
- **做法**：直接读取 `.part` 文件在磁盘上的大小
- 若 `resume_len > 0`，标记 `is_resume = True`

### 第二步：构造 Range 请求

```python
# http.py L79-L103
if ctx.resume_len > 0:
    range_start = ctx.resume_len
    if req_start is not None:
        range_start += req_start
    if ctx.is_resume:
        self.report_resuming_byte(ctx.resume_len)
    ctx.open_mode = 'ab'          # 追加模式
elif req_start is not None:
    range_start = req_start
...
if has_range:
    request.headers['Range'] = f'bytes={int(range_start)}-{int_or_none(range_end) or ""}'
```

- 文件打开模式由 `wb`（覆盖写）切换为 `ab`（追加写）
- Range 头格式：`bytes=<start>-` 或 `bytes=<start>-<end>`

### 第三步：验证服务端是否真正支持续传

**这是"是否可恢复"最关键的判定点**。客户端发送 Range 请求后，必须检查响应头：

```python
# http.py L119-L147
ctx.data = self.ydl.urlopen(request)
if has_range:
    content_range = ctx.data.headers.get('Content-Range')
    content_range_start, content_range_end, content_len = parse_http_range(content_range)
    # 校验1：Content-Range 存在且起始位置匹配
    if range_start == content_range_start and (
            not ctx.chunk_size
            or content_range_end == range_end
            or content_len < range_end):
        ctx.content_len = content_len
        return  # ✅ 续传成功
    # 校验2：Content-Range 不存在或位置不匹配 → 服务端不支持 Range
    elif range_start > 0:
        self.report_unable_to_resume()
    ctx.resume_len = 0
    ctx.open_mode = 'wb'          # 退回覆盖写
    ctx.data_len = ctx.content_len = int_or_none(...)
```

`parse_http_range()` 工具函数位于 [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/utils/_utils.py#L4875-L4882)：

```python
def parse_http_range(range):
    if not range:
        return None, None, None
    crg = re.search(r'bytes[ =](\d+)-(\d+)?(?:/(\d+))?', range)
    if not crg:
        return None, None, None
    return int(crg.group(1)), int_or_none(crg.group(2)), int_or_none(crg.group(3))
```

可解析两种格式：
- `Range: bytes=1000-2000` → `(1000, 2000, None)`
- `Content-Range: bytes 1000-2000/5000` → `(1000, 2000, 5000)`

**判定结论**：
| 条件 | 结果 |
|---|---|
| 响应带 `Content-Range` 且 `content_range_start == range_start` | ✅ 可续传，追加写入 |
| 响应无 `Content-Range`（服务端忽略 Range，返回完整文件） | ❌ 不可续传，清零重下 |
| HTTP 416 Range Not Satisfiable | 进入特殊分支 |

### 第四步：HTTP 416 的特殊处理

```python
# http.py L148-L188
except HTTPError as err:
    if err.status == 416:
        try:
            ctx.data = self.ydl.urlopen(Request(url, request_data, headers))
            content_length = ctx.data.headers['Content-Length']
        except HTTPError as err:
            if err.status < 500 or err.status >= 600:
                raise
        else:
            # 误差 100 字节内认为已下载完毕（YouTube 大小微变问题）
            if (content_length is not None
                    and (ctx.resume_len - 100 < int(content_length) < ctx.resume_len + 100)):
                self.report_file_already_downloaded(ctx.filename)
                self.try_rename(ctx.tmpfilename, ctx.filename)
                raise SucceedDownload
            else:
                self.report_unable_to_resume()
                ctx.resume_len = 0
                ctx.open_mode = 'wb'
```

- 416 表示请求的 Range 服务器无法满足
- 此时不带 Range 重新请求获取 `Content-Length`
- 若本地文件大小 ±100 字节匹配远程大小 → 认为下载已完成，直接改名成功
- 否则判定"不可续传"，清零从头下载

### 第五步：下载中断时的重试

```python
# http.py L237-L246
def retry(e):
    close_stream()
    if ctx.tmpfilename == '-':
        ctx.resume_len = byte_counter
    else:
        try:
            ctx.resume_len = os.path.getsize(ctx.tmpfilename)
        except FileNotFoundError:
            ctx.resume_len = 0
    raise RetryDownload(e)
```

每次重试时**重新从磁盘读取文件大小**，而非依赖内存中的 `byte_counter`，确保不丢失已落盘的数据。

---

## 三、分段下载的可恢复判定（FragmentFD）

分段下载（HLS/DASH 等）采用"双重校验"：`.part` 文件大小 + `.ytdl` 元数据文件。

核心逻辑位于 [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/fragment.py#L157-L223) 的 `_prepare_frag_download()`。

### 第一步：.part 文件大小探测

```python
# fragment.py L175-L181
tmpfilename = self.temp_name(ctx['filename'])
open_mode = 'wb'

resume_len = self.filesize_or_none(tmpfilename)
if resume_len > 0:
    open_mode = 'ab'
```

与 HttpFD 一致：文件大小即表示已下载的**已完成片段**累计字节数。

### 第二步：.ytdl 状态文件读取与校验

```python
# fragment.py L189-L213
if self.__do_ytdl_file(ctx):
    ytdl_file_exists = os.path.isfile(self.ytdl_filename(ctx['filename']))
    continuedl = self.params.get('continuedl', True)
    if continuedl and ytdl_file_exists:
        self._read_ytdl_file(ctx)
        is_corrupt = ctx.get('ytdl_corrupt') is True
        is_inconsistent = ctx['fragment_index'] > 0 and resume_len == 0
        if is_corrupt or is_inconsistent:
            message = (
                '.ytdl file is corrupt' if is_corrupt else
                'Inconsistent state of incomplete fragment download')
            self.report_warning(f'{message}. Restarting from the beginning ...')
            ctx['fragment_index'] = resume_len = 0
            ...
            self._write_ytdl_file(ctx)
    else:
        if not continuedl:
            ...
            ctx['fragment_index'] = resume_len = 0
        self._write_ytdl_file(ctx)
        assert ctx['fragment_index'] == 0
```

`__do_ytdl_file()` 启用条件：非直播、非 stdout、未设置 `_no_ytdl_file`。

**状态一致性校验**（不可恢复的两种场景）：

| 场景 | 判定结果 | 处理方式 |
|---|---|---|
| `.ytdl` 文件解析失败（JSON 损坏） | `is_corrupt = True` | 清零重下 |
| `.ytdl` 记录 `fragment_index > 0`，但 `.part` 文件大小为 0 | `is_inconsistent = True` | 清零重下 |

两者任一触发，都会打印警告并**重置状态从头开始**。

### 第三步：跳过已下载片段

在具体下载器中（如 HLS），利用恢复的 `fragment_index` 跳过已完成片段：

```python
# hls.py L212-L214
frag_index += 1
if frag_index <= ctx['fragment_index']:
    continue   # 跳过已下载的片段
```

DASH 下载器同理：

```python
# dash.py L79-L82
frag_index += 1
if frag_index <= ctx['fragment_index']:
    continue
```

### 第四步：状态持久化——每个片段完成后写 .ytdl

```python
# fragment.py L146-L155
def _append_fragment(self, ctx, frag_content):
    try:
        ctx['dest_stream'].write(frag_content)
        ctx['dest_stream'].flush()
    finally:
        if self.__do_ytdl_file(ctx):
            self._write_ytdl_file(ctx)   # ← 关键：写入最新 fragment_index
        if not self.params.get('keep_fragments', False):
            self.try_remove(ctx['fragment_filename_sanitized'])
        del ctx['fragment_filename_sanitized']
```

**写入时机**：片段数据**写入 .part 文件并 flush 之后**，立即更新 `.ytdl`。

```python
# fragment.py L95-L109
def _write_ytdl_file(self, ctx):
    frag_index_stream, _ = self.sanitize_open(self.ytdl_filename(ctx['filename']), 'w')
    try:
        downloader = {
            'current_fragment': {
                'index': ctx['fragment_index'],
            },
        }
        if 'extra_state' in ctx:
            downloader['extra_state'] = ctx['extra_state']
        if ctx.get('fragment_count') is not None:
            downloader['fragment_count'] = ctx['fragment_count']
        frag_index_stream.write(json.dumps({'downloader': downloader}))
    finally:
        frag_index_stream.close()
```

### 第五步：下载完成后清理

```python
# fragment.py L285-L288
def _finish_frag_download(self, ctx, info_dict):
    ctx['dest_stream'].close()
    if self.__do_ytdl_file(ctx):
        self.try_remove(self.ytdl_filename(ctx['filename']))   # 删除 .ytdl
```

下载成功后 `.ytdl` 被删除；`.part` 被重命名为目标文件名。若下次再运行时还能看到 `.ytdl`，说明上次是异常中断。

### 第六步：单个片段内部的续传

`_download_fragment()` 会对每个独立片段文件也做 HttpFD 级别的续传：

```python
# fragment.py L111-L130
def _download_fragment(self, ctx, frag_url, info_dict, headers=None, request_data=None):
    fragment_filename = '%s-Frag%d' % (ctx['tmpfilename'], ctx['fragment_index'])
    ...
    frag_resume_len = 0
    if ctx['dl'].params.get('continuedl', True):
        frag_resume_len = self.filesize_or_none(self.temp_name(fragment_filename))
    fragment_info_dict['frag_resume_len'] = ctx['frag_resume_len'] = frag_resume_len

    success, _ = ctx['dl'].download(fragment_filename, fragment_info_dict)
```

单个片段文件命名为 `<filename>.part-Frag<index>`，其自身也支持 HttpFD 的 Range 续传。

---

## 四、Range 请求头与状态索引持久化的协作关系（深入分析）

分段下载中存在**两层独立但协作的续传机制**：
1. **片段级别**：用 `.ytdl` 中持久化的 `fragment_index` 决定从第几个片段开始
2. **字节级别**：每个片段内部的 HTTP Range 请求决定从该片段的哪个字节开始

两者并非简单并列，而是通过精确的调用时序互相衔接。

### 4.1 第一层：Manifest 中的 byte_range 与 HTTP Range 头

HLS m3u8 中可能包含 `#EXT-X-BYTERANGE` 标签，DASH manifest 中也可能用 `SegmentURL` + `mediaRange` 指定字节范围。这些信息在片段构造时被解析为 `fragment['byte_range']`。

在 [hls.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/hls.py#L286-L292) 中解析 BYTERANGE：

```python
# hls.py L286-L292
elif line.startswith('#EXT-X-BYTERANGE'):
    splitted_byte_range = line[17:].split('@')
    sub_range_start = int(splitted_byte_range[1]) if len(splitted_byte_range) == 2 else byte_range_offset
    byte_range = {
        'start': sub_range_start,
        'end': sub_range_start + int(splitted_byte_range[0]),
    }
```

随后在片段下载时，该 byte_range 被转化为 HTTP 请求头 [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/fragment.py#L439-L448)：

```python
# fragment.py L439-L448
def download_fragment(fragment, ctx):
    ...
    frag_index = ctx['fragment_index'] = fragment['frag_index']
    headers = HTTPHeaderDict(info_dict.get('http_headers'))
    byte_range = fragment.get('byte_range')
    if byte_range:
        headers['Range'] = 'bytes=%d-%d' % (byte_range['start'], byte_range['end'] - 1)
```

此时 `headers['Range']` 表示**该片段在远程媒体文件中的绝对字节区间**。

### 4.2 第二层：单个片段文件内部的续传（HttpFD 级）

`_download_fragment()` 创建片段临时文件 `<filename>.part-Frag<index>`，并将它交给 `HttpQuietDownloader`（即 HttpFD）。HttpFD 会**在传入的 Range 头基础上再叠加本地已下载字节偏移**。

关键叠加逻辑在 [http.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/http.py#L56-L88)：

```python
# http.py L56-L88
# 先解析 FragmentFD 传入的 byte_range（来自 manifest）
req_start, req_end, _ = parse_http_range(headers.get('Range'))

if self.params.get('continuedl', True):
    if os.path.isfile(ctx.tmpfilename):
        ctx.resume_len = os.path.getsize(ctx.tmpfilename)  # 该片段本地已下载字节数
...

if ctx.resume_len > 0:
    range_start = ctx.resume_len
    if req_start is not None:
        # ★ 关键叠加：本地已下载字节 + manifest 指定的起始位置
        range_start += req_start
    ...
elif req_start is not None:
    range_start = req_start
...
request.headers['Range'] = f'bytes={int(range_start)}-{int_or_none(range_end) or ""}'
```

**叠加规则**：最终 HTTP Range = `bytes = (manifest 起始 + 本地已下载字节) - (manifest 结束)`

举例：
- manifest 指定片段为远程文件的 `bytes=5000-5999`（req_start=5000, req_end=5999）
- 本地 `<name>.part-Frag3` 已存在，大小为 300 字节
- 最终发出的请求 Range 为 `bytes=5300-5999`
- 文件以 `ab` 追加模式打开，从第 300 字节继续写

同时，`range_end` 也会被限制不超过 manifest 指定的结束位置：

```python
# http.py L95-L103
if ctx.chunk_size:
    chunk_aware_end = range_start + ctx.chunk_size - 1
    range_end = chunk_aware_end if req_end is None else min(chunk_aware_end, req_end)
elif req_end is not None:
    range_end = req_end
```

### 4.3 fragment_index 的语义辨析："已完成"还是"下一个"？

`.ytdl` 中 `current_fragment.index` 存储的到底是"已完成的最后一个片段索引"，还是"下一个待下载的片段索引"？

答案是：**下一个待下载的片段索引**。证据来自两处：

**证据 1：progress hook 中的自增时机**

在 `_start_frag_download()` 注册的 `frag_progress_hook` 中 [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/fragment.py#L270-L273)：

```python
# fragment.py L270-L273
if s['status'] == 'finished':
    state['fragment_index'] += 1
    ctx['fragment_index'] = state['fragment_index']
    progress.thread_reset()
```

`s['status'] == 'finished'` 是由 HttpFD 在单个片段文件下载完成时触发的（见 http.py L349-L356）。此时**片段文件已从 `.part-FragN` 重命名为正式片段文件名**，但尚未合并到最终 `.part` 文件中。

**证据 2：恢复时的跳过判定使用 `<=`**

HLS 和 DASH 中跳过已下载片段的条件是 `<=` [hls.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/hls.py#L212-L214)：

```python
# hls.py L212-L214
frag_index += 1
if frag_index <= ctx['fragment_index']:
    continue   # 跳过已下载的片段
```

如果 `.ytdl` 中存储的是 `5`，说明片段 `1..5` 都已完成续传，下一次从片段 `6` 开始。

### 4.4 完整调用时序：fragment_index 何时写入 .ytdl？

以单线程（`concurrent_fragment_downloads=1`）为例，单个片段的完整生命周期如下：

```
download_and_append_fragments 循环
  │
  ├─ ① download_fragment(fragment, ctx)
  │     │
  │     ├─ ctx['fragment_index'] = fragment['frag_index']   // 设为当前片段（例 N）
  │     ├─ byte_range → headers['Range']                    // manifest 级 Range
  │     ├─ _download_fragment(ctx, ...)
  │     │     │
  │     │     ├─ 检测 FragN 临时文件大小 → frag_resume_len  // 片段级续传探测
  │     │     └─ ctx['dl'].download(FragN, fragment_info_dict)   // 进入 HttpFD
  │     │           │
  │     │           ├─ 叠加 Range（manifest_start + frag_resume_len）
  │     │           ├─ 下载片段数据
  │     │           ├─ 完成后 try_rename(.part-FragN → FragN)
  │     │           └─ _hook_progress({'status': 'finished'})
  │     │                 │
  │     │                 └─ ★ frag_progress_hook 被触发
  │     │                       └─ ctx['fragment_index'] += 1   // 从 N 变为 N+1
  │     │
  │     └─ return   // 此时 ctx['fragment_index'] 已是 N+1
  │
  ├─ ② append_fragment(frag_content, frag_index, ctx)
  │     │
  │     └─ _append_fragment(ctx, pack_func(frag_content, frag_index))
  │           │
  │           ├─ ctx['dest_stream'].write(frag_content)     // 合并到最终 .part
  │           ├─ ctx['dest_stream'].flush()                 // 落盘
  │           ├─ if __do_ytdl_file:
  │           │     └─ ★ _write_ytdl_file(ctx)              // 写入 ctx['fragment_index'] = N+1
  │           ├─ if not keep_fragments:
  │           │     └─ try_remove(FragN)                    // 删除单片段文件
  │           └─ del ctx['fragment_filename_sanitized']
  │
  └─ 进入下一个片段循环
```

**关键观察**：
- `ctx['fragment_index']` 在 progress hook 中从 `N` 自增为 `N+1`，发生在片段文件下载完成但**尚未合并到最终文件**之前
- `_write_ytdl_file()` 在合并写入 + flush **之后**执行，此时写入的是 `N+1`
- 这保证了：只要 `.ytdl` 中记录了 `M`，就意味着片段 `1..M-1` 已经被合并写入最终 `.part` 文件，恢复时可以安全跳过

**并发模式（`max_workers > 1`）下的差异**：

```python
# fragment.py L487-L501
if max_workers > 1:
    def _download_fragment(fragment):
        ctx_copy = ctx.copy()
        download_fragment(fragment, ctx_copy)
        return fragment, fragment['frag_index'], ctx_copy.get('fragment_filename_sanitized')

    with tpe or concurrent.futures.ThreadPoolExecutor(max_workers) as pool:
        try:
            for fragment, frag_index, frag_filename in pool.map(_download_fragment, fragments):
                ctx.update({
                    'fragment_filename_sanitized': frag_filename,
                    'fragment_index': frag_index,   # 用主线程返回的 frag_index 更新
                })
                if not append_fragment(...):
                    return False
```

- 每个线程使用 `ctx.copy()`，避免并发修改主 ctx
- 主线程从 `pool.map` 返回结果中取得 `frag_index`（即 `fragment['frag_index']`，**不是自增后的值**）
- 但是！progress hook 中的自增仍然作用于全局 `state`/`ctx` 上
- 最终 `_write_ytdl_file` 在主线程 `append_fragment` 中被调用，此时 `ctx['fragment_index']` 的值取决于 progress hook 已累计自增了多少

### 4.5 恢复时：两层续传如何共同决定恢复位置

完整恢复流程：

```
恢复启动
  │
  ├─ FragmentFD._prepare_frag_download()
  │     ├─ 读 .part 文件大小 → resume_len（已完成片段字节数）
  │     ├─ 读 .ytdl JSON → ctx['fragment_index'] = M
  │     └─ 一致性校验通过
  │
  ├─ 构造 fragments 列表
  │     └─ for frag_index in 1..total:
  │           if frag_index <= M:   # M = ctx['fragment_index']
  │               continue          # 跳过片段 1..M
  │
  └─ 从片段 M+1 开始逐个下载
        │
        ├─ ① 检测 <name>.part-Frag<M+1> 是否存在
        │     ├─ 存在 → frag_resume_len = 文件大小（例：已下 200 字节）
        │     └─ 不存在 → frag_resume_len = 0
        │
        ├─ ② 构造 Range
        │     ├─ manifest byte_range：例 {start: 8000, end: 8999}
        │     ├─ frag_resume_len = 200
        │     └─ 最终请求 Range: bytes=8200-8999
        │
        ├─ ③ 下载剩余字节，追加写入 .part-Frag<M+1>
        │
        └─ ④ 下载完成 → progress hook 自增 fragment_index → 合并 → flush → 写 .ytdl
```

**如果在第 M+1 个片段下载中途中断**：
- `.part` 文件大小仍等于片段 1..M 的总大小（因为当前片段尚未合并）
- `.ytdl` 中 `fragment_index` 仍为 M（因为 `_write_ytdl_file` 还没被调用）
- `<name>.part-Frag<M+1>` 可能存在，大小为已下载的部分字节
- 下次恢复：`fragment_index = M`，跳过 1..M，从 M+1 开始；此时再检测 FragM+1 的临时文件，从第 frag_resume_len 字节继续

**如果在 progress hook 自增之后、_write_ytdl_file 之前中断**（极端时序）：
- 这种情况下 progress hook 已把 ctx['fragment_index'] 从 M 改为 M+1，但 `.ytdl` 还没写
- 下次恢复读 `.ytdl` 仍得到 M
- 会重新下载片段 M+1（即使上次其实已下载完成且合并 flush 成功了一部分——但因为 flush 成功和 _write_ytdl_file 之间窗口极小，实际中几乎不可能只成功前者而不执行后者；且两者在同一个 try-finally 块内）

### 4.6 一致性校验的设计意图

```python
# fragment.py L194-L195
is_corrupt = ctx.get('ytdl_corrupt') is True
is_inconsistent = ctx['fragment_index'] > 0 and resume_len == 0
```

- `is_inconsistent`：`.ytdl` 声称已完成 N 个片段，但 `.part` 文件大小为 0
  - 可能原因：`.part` 被手动删除，但 `.ytdl` 还在；或极端异常导致文件系统不一致
  - 处理：两个状态都清零，从头开始
- `is_corrupt`：`.ytdl` JSON 解析失败
  - 处理：同样清零重下，避免损坏的索引导致跳过不完整的片段

---

## 五、统一入口：FileDownloader.download() 的预检查

所有下载器在进入 `real_download()` 之前，都会经过 [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/common.py#L430-L482) 的前置判定：

```python
def download(self, filename, info_dict, subtitle=False):
    nooverwrites_and_exists = (
        not self.params.get('overwrites', True)
        and os.path.exists(filename)
    )

    if not hasattr(filename, 'write'):
        continuedl_and_exists = (
            self.params.get('continuedl', True)
            and os.path.isfile(filename)
            and not self.params.get('nopart', False)
        )

        # 目标文件已存在 → 报告已下载完成，直接返回
        if filename != '-' and (nooverwrites_and_exists or continuedl_and_exists):
            self.report_file_already_downloaded(filename)
            ...
            return True, False
```

**注意**：若目标文件（非 `.part`）本身已存在且启用了 `continuedl`，会**直接认为下载完成**，不会去校验 `.part` + `.ytdl`。这是最高优先级的短路判定。

---

## 六、可恢复判定流程图（综合）

```
启动下载
   │
   ├─ 目标文件已存在且 continuedl=True
   │   └─ → 直接判定"已下载完成" ──→ 结束 ✅
   │
   └─ 进入具体下载器
        │
        ├─ HttpFD 路径（普通文件）
        │   │
        │   ├─ ① 探测 .part 文件大小 → resume_len
        │   │
        │   ├─ ② 发送 Range: bytes=<resume_len>- 请求
        │   │
        │   ├─ ③ 检查响应 Content-Range
        │   │   ├─ 存在且 start 匹配 → ✅ 可续传（ab 模式追加）
        │   │   ├─ HTTP 416 → 校验文件大小 ±100B
        │   │   │   ├─ 匹配 → ✅ 已下载完成
        │   │   │   └─ 不匹配 → ❌ 清零重下（wb 模式）
        │   │   └─ Content-Range 缺失/不匹配 → ❌ 清零重下
        │   │
        │   └─ ④ 传输中断重试 → 重新读取磁盘文件大小作为 resume_len
        │
        └─ FragmentFD 路径（HLS/DASH 等分段）——两层续传协作
            │
            ├─ 【第一层：片段级】
            │   ├─ ① 探测 .part 文件大小 → resume_len（已完成片段字节数）
            │   ├─ ② 读取 .ytdl JSON → ctx['fragment_index'] = M
            │   ├─ ③ 一致性校验（.ytdl 损坏？index 与文件大小矛盾？）
            │   │   ├─ 不通过 → ❌ 清零重下
            │   │   └─ 通过 → ✅
            │   └─ ④ 跳过所有 frag_index <= M 的片段（1..M 已完成）
            │
            └─ 【第二层：字节级】针对每个待下载片段 N（从 M+1 开始）
                ├─ ① 探测 <name>.part-Frag<N> 临时文件 → frag_resume_len
                ├─ ② 解析 manifest byte_range（req_start, req_end）
                ├─ ③ 叠加：最终 Range = bytes=(req_start + frag_resume_len)-req_end
                ├─ ④ 下载片段；完成后 progress hook 自增 fragment_index
                ├─ ⑤ 合并片段内容到 .part + flush
                └─ ⑥ 写 .ytdl 持久化最新 fragment_index（=下一个待下载）
```

---

## 七、关键参数总览

| 参数 | 位置 | 默认 | 作用 |
|---|---|---|---|
| `continuedl` | common.py L58, http.py L59 | `True` | 总开关，关闭则从不续传 |
| `nopart` | common.py L60, L219 | `False` | 禁用 `.part` 临时文件（也禁用续传） |
| `overwrites` | common.py L435 | `True` | 目标文件已存在时是否覆盖 |
| `retries` / `fragment_retries` | common.py | CLI=10, API=0 | 下载/片段失败重试次数 |
| `http_chunk_size` | http.py L46-L49 | 0 | HTTP 分块大小，设为非 0 时启用分片下载（支持断点续传分片） |
| `keep_fragments` | fragment.py L36 | `False` | 是否保留已下载的单片段文件 |
| `_no_ytdl_file` | fragment.py L39 | `False` | 调试用，禁用 `.ytdl` 状态文件 |
| `skip_unavailable_fragments` | fragment.py | `True` | 跳过不可用片段（不中断整体下载） |

---

## 八、关键文件索引

| 文件 | 主要职责 |
|---|---|
| [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/common.py) | 基类 `FileDownloader`，临时文件命名、统一入口、进度钩子 |
| [http.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/http.py) | `HttpFD`，Range 请求构造、Content-Range 校验、416 处理 |
| [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/fragment.py) | `FragmentFD`，`.ytdl` 状态读写、片段调度、一致性校验 |
| [hls.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/hls.py) | `HlsFD`，解析 m3u8，跳过已下载片段索引 |
| [dash.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/dash.py) | `DashSegmentsFD`，解析 DASH manifest，跳过已下载片段索引 |
| [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/utils/_utils.py#L4875-L4882) | `parse_http_range()`，解析 Range/Content-Range 响应头 |
