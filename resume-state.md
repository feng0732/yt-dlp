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

- `current_fragment.index`：**最后一个已安全合并写入 .part 文件**的片段索引（1-based，详见 4.3 节语义辨析）
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

### 4.3 fragment_index 的语义辨析："已安全合并的边界"

`.ytdl` 中 `current_fragment.index` 存储的到底是什么语义？经过精确代码追踪，答案是：

> **最后一个已安全合并写入 .part 文件的片段索引（1-based）。片段 1..M 已完成，下一次从 M+1 开始。**

**关键证据：`state` 与 `ctx` 的独立生命周期**

在 [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/fragment.py#L230-L237) 中，`state` 是一个独立的闭包变量，有自己的生命周期：

```python
# fragment.py L230-L237
state = {
    'status': 'downloading',
    'downloaded_bytes': resume_len,
    'fragment_index': ctx['fragment_index'],   # 初始值来自 ctx（恢复值或 0）
    'fragment_count': total_frags,
    'filename': ctx['filename'],
    'tmpfilename': ctx['tmpfilename'],
}
```

`state['fragment_index']` 只在 progress hook 中被修改（自增），**不与 `L443` 处的 `ctx['fragment_index'] = fragment['frag_index']` 赋值同步**：

```python
# fragment.py L270-L273
if s['status'] == 'finished':
    state['fragment_index'] += 1           # 只改 state，累计已完成下载的片段数
    ctx['fragment_index'] = state['fragment_index']   # 同步给 ctx
    progress.thread_reset()
```

**证据 2：恢复时的跳过判定使用 `<=`**

HLS 和 DASH 中跳过已下载片段的条件是 `<=` [hls.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/hls.py#L202-L214)：

```python
# hls.py L202-L214
frag_index = 0              # 初始为 0
for line in s.splitlines():
    if not line.startswith('#'):
        frag_index += 1     # 第一次循环后为 1，第二次 2...
        if frag_index <= ctx['fragment_index']:
            continue        # 跳过已完成的片段
```

若 `.ytdl` 中 `ctx['fragment_index'] = 3`：
- frag_index=1 → 1 <= 3 → continue ✓（跳过）
- frag_index=2 → 2 <= 3 → continue ✓（跳过）
- frag_index=3 → 3 <= 3 → continue ✓（跳过）
- frag_index=4 → 4 > 3 → 加入 fragments

这直接证明：`M=3` 表示片段 **1、2、3 都已完成**，从 **4** 开始。语义是"已完成边界"，不是"下一个待下载"。

### 4.4 完整调用时序：fragment_index 何时写入 .ytdl？

#### 单线程模式（`concurrent_fragment_downloads=1`）

以从头下载（无恢复）为例，`ctx['fragment_index']` 和 `state['fragment_index']` 的独立生命周期如下：

```
初始：ctx['fragment_index'] = 0, state['fragment_index'] = 0

download_and_append_fragments 循环
  │
  ├─ 处理片段 1（fragment['frag_index'] = 1）
  │   │
  │   ├─ ① download_fragment(fragment, ctx)
  │   │     ├─ L443: ctx['fragment_index'] = 1   // 只改 ctx，state 仍为 0
  │   │     ├─ byte_range → headers['Range']
  │   │     └─ _download_fragment → ctx['dl'].download(Frag1, ...)
  │   │           ├─ 下载完成 → try_rename(.part-Frag1 → Frag1)
  │   │           └─ _hook_progress({'status': 'finished'})
  │   │                 └─ frag_progress_hook 触发
  │   │                       ├─ L271: state['fragment_index'] += 1   // 0 → 1
  │   │                       └─ L272: ctx['fragment_index'] = state['fragment_index']   // 1
  │   │
  │   └─ ② append_fragment(..., frag_index=1, ctx)
  │         └─ _append_fragment(ctx, ...)
  │               ├─ write(frag_content) + flush()   // 片段 1 合并到 .part
  │               └─ L152: _write_ytdl_file(ctx)     // 写入 ctx['fragment_index'] = 1
  │                                                           // 语义：片段 1 已完成
  │
  ├─ 处理片段 2（fragment['frag_index'] = 2）
  │   │
  │   ├─ ① download_fragment(fragment, ctx)
  │   │     ├─ L443: ctx['fragment_index'] = 2   // 只改 ctx，state 仍为 1
  │   │     ├─ ...
  │   │     └─ 下载完成 → progress hook
  │   │                       ├─ state['fragment_index'] += 1   // 1 → 2
  │   │                       └─ ctx['fragment_index'] = 2
  │   │
  │   └─ ② append_fragment(..., frag_index=2, ctx)
  │         └─ _append_fragment
  │               ├─ write + flush   // 片段 2 合并到 .part
  │               └─ _write_ytdl_file(ctx)   // 写入 2
  │                                                     // 语义：片段 1,2 已完成
  │
  └─ ...以此类推
```

**关键观察（单线程）**：
- `L443` 处的 `ctx['fragment_index'] = fragment['frag_index']` **只修改 ctx，不修改 state**
- `state['fragment_index']` 是独立的计数器，**只在 progress hook 中自增**，表示"已完成下载的片段数"
- progress hook 同时将 ctx 同步为 state 的值
- `_write_ytdl_file()` 在合并 + flush **之后**执行，写入的值等于已安全合并的最大片段索引
- 写入 M = N 表示：**片段 1..N 已安全合并**

#### 并发模式（`max_workers > 1`）

并发模式下语义保持一致，但数据来源不同：

```python
# fragment.py L487-L507
if max_workers > 1:
    def _download_fragment(fragment):
        ctx_copy = ctx.copy()
        download_fragment(fragment, ctx_copy)
        return fragment, fragment['frag_index'], ctx_copy.get('fragment_filename_sanitized')
        # ↑ 返回的是 fragment['frag_index']（原始值 N），不是 progress hook 自增后的值

    with tpe or concurrent.futures.ThreadPoolExecutor(max_workers) as pool:
        try:
            for fragment, frag_index, frag_filename in pool.map(_download_fragment, fragments):
                ctx.update({
                    'fragment_filename_sanitized': frag_filename,
                    'fragment_index': frag_index,   # ★ 覆盖为原始值 N
                })
                if not append_fragment(...):
                    return False
```

**关键观察（并发）**：
- 每个子线程使用 `ctx.copy()`，`L443` 的赋值只修改 `ctx_copy`
- 子线程下载完成触发 progress hook 时，**修改的是主 `state` 和主 `ctx`**（因为 progress hook 是闭包，捕获的是主 ctx）
- 但主线程在 `L498` 会将 `ctx['fragment_index']` 覆盖为 `fragment['frag_index']`（即当前正在合并的片段的原始索引 N）
- 由于 `pool.map` **按原始顺序返回**（不按完成顺序），合并是顺序的
- 最终 `_write_ytdl_file(ctx)` 写入的值 = `frag_index` = 当前正在合并的片段索引 = 最后一个已完成的片段索引

**并发竞态说明**：
- progress hook 自增的主 `ctx['fragment_index']` 会被主线程的赋值覆盖
- 但 `state['fragment_index']` 仍用于进度估算（L263），不会被覆盖
- 无论竞态如何，写入 `.ytdl` 的值始终等于当前顺序合并的片段索引，语义不变

### 4.5 恢复时：两层续传如何共同决定恢复位置

完整恢复流程（假设上次中断时 `.ytdl` 中 M=3，表示片段 1..3 已安全合并）：

```
恢复启动
  │
  ├─ FragmentFD._prepare_frag_download()
  │     ├─ 读 .part 文件大小 → resume_len（已完成片段 1..3 的字节总和）
  │     ├─ 读 .ytdl JSON → ctx['fragment_index'] = M = 3
  │     └─ 一致性校验通过（fragment_index > 0 且 resume_len > 0）
  │
  ├─ 构造 fragments 列表
  │     └─ for frag_index in 1..total:
  │           if frag_index <= 3:   # M = 3
  │               continue          # 跳过片段 1,2,3
  │
  └─ 从片段 4 开始逐个下载
        │
        ├─ ① 检测 <name>.part-Frag4 是否存在
        │     ├─ 存在 → frag_resume_len = 文件大小（例：已下 200 字节）
        │     └─ 不存在 → frag_resume_len = 0
        │
        ├─ ② 构造 Range
        │     ├─ manifest byte_range：例 {start: 8000, end: 8999}
        │     ├─ frag_resume_len = 200
        │     └─ 最终请求 Range: bytes=8200-8999
        │
        ├─ ③ 下载剩余字节，追加写入 .part-Frag4
        │
        └─ ④ 下载完成 → progress hook 自增 state → 合并 → flush → 写 .ytdl（M=4）
```

**场景 1：在第 4 个片段下载中途中断**
- `.part` 文件大小仍等于片段 1..3 的总大小（因为片段 4 尚未合并）
- `.ytdl` 中 `fragment_index` 仍为 3（因为 `_write_ytdl_file` 还没被调用）
- `<name>.part-Frag4` 可能存在，大小为已下载的部分字节（例：200）
- 下次恢复：`fragment_index = 3`，跳过 1..3，从 4 开始；检测到 Frag4 临时文件，从第 200 字节继续

**场景 2：progress hook 已触发但合并尚未完成时中断**
- progress hook 已把 `state['fragment_index']` 和 `ctx['fragment_index']` 从 3 改为 4（表示片段 4 已下载完成）
- 但合并、flush、`_write_ytdl_file` 尚未执行
- `.ytdl` 仍为 3，`<name>.part-Frag4` 文件已存在且完整
- 下次恢复：读 `.ytdl` 得 M=3，跳过 1..3，从 4 开始；HttpFD 检测到 Frag4 已完整，立即完成，进入合并

**场景 3：合并 flush 完成但 _write_ytdl_file 执行前极端中断（如断电）**
- 片段 4 已写入 `.part` 并 flush，数据已落盘
- 但 `_write_ytdl_file` 在 finally 块中还没来得及执行（理论上 finally 一定会执行，除非进程被硬杀或断电）
- 下次恢复：读 `.ytdl` 得 M=3，跳过 1..3，从 4 开始；会重复下载片段 4 并再次合并写入
- 这是**"至少一次"语义**：宁可重复下载，也不跳过未完成的片段

### 4.6 一致性校验的设计意图

```python
# fragment.py L194-L195
is_corrupt = ctx.get('ytdl_corrupt') is True
is_inconsistent = ctx['fragment_index'] > 0 and resume_len == 0
```

- `is_inconsistent`：`.ytdl` 声称已完成 M 个片段（M > 0），但 `.part` 文件大小为 0
  - 可能原因：`.part` 被手动删除但 `.ytdl` 还在；或极端异常导致文件系统不一致
  - 处理：两个状态都清零，从头开始
- `is_corrupt`：`.ytdl` JSON 解析失败
  - 处理：同样清零重下，避免损坏的索引导致跳过不完整的片段

**设计原则**：当"索引状态"与"实际字节"不一致时，始终以"实际字节"为准。

### 4.7 HLS 初始化片段的跳过边界分析

HLS 规范中 `#EXT-X-MAP` 标签定义了媒体初始化片段（通常是 fMP4 的 init 段）。它的跳过逻辑与普通媒体片段**不一致**，可能导致恢复时重复合并。

#### 4.7.1 初始化片段与普通媒体片段的 frag_index 分配

在 [hls.py L202-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/hls.py#L202-L263) 中，`frag_index` 从 0 开始递增：

| 片段类型 | 代码位置 | frag_index 赋值 | 是否有跳过判断 |
|---|---|---|---|
| 初始化片段 (#EXT-X-MAP) | L233-L263 | `frag_index += 1` → 1 | **无** |
| 普通媒体片段 | L207-L227 | `frag_index += 1` → 2,3,4... | **有**（L213） |

**初始化片段处理代码**（无跳过判断）：

```python
# hls.py L233-L263
elif line.startswith('#EXT-X-MAP'):
    if format_index is not None and discontinuity_count != format_index:
        continue
    if frag_index > 0:
        self.report_error(
            'Initialization fragment found after media fragments, unable to download')
        return False
    frag_index += 1           # frag_index 从 0 → 1
    ...
    fragments.append({
        'frag_index': frag_index,  # = 1
        'url': frag_url,
        ...
    })
    # ↑ 没有 if frag_index <= ctx['fragment_index']: continue 判断！
```

**普通媒体片段处理代码**（有跳过判断）：

```python
# hls.py L207-L214
if not line.startswith('#'):
    ...
    frag_index += 1                 # 从 1 → 2, 2 → 3...
    if frag_index <= ctx['fragment_index']:
        continue                    # ← 只有这里有跳过判断
    fragments.append({
        'frag_index': frag_index,   # = 2,3,4...
        ...
    })
```

#### 4.7.2 两种场景下的 frag_index 对比

| 片段 | 有 #EXT-X-MAP | 无 #EXT-X-MAP |
|---|---|---|
| 初始化片段 | frag_index = 1 | - |
| 媒体片段 1 | frag_index = 2 | frag_index = 1 |
| 媒体片段 2 | frag_index = 3 | frag_index = 2 |
| 媒体片段 3 | frag_index = 4 | frag_index = 3 |

初始化片段的存在会让所有后续媒体片段的 `frag_index` 都 **+1**。

#### 4.7.3 恢复时的跳过行为对比

假设 `.ytdl` 中 `ctx['fragment_index'] = M = 3`（表示片段 1,2,3 已安全合并）：

**场景 A：有 #EXT-X-MAP（frag_index 分配见上表）**
- 初始化片段 frag_index=1：**无跳过判断** → 加入 fragments 列表
- 媒体片段 1 frag_index=2：`2 <= 3` → 跳过 ✓
- 媒体片段 2 frag_index=3：`3 <= 3` → 跳过 ✓
- 媒体片段 3 frag_index=4：`4 > 3` → 加入

**结果**：初始化片段被**重新加入**，会被重复下载和合并！

**场景 B：无 #EXT-X-MAP**
- 媒体片段 1 frag_index=1：`1 <= 3` → 跳过 ✓
- 媒体片段 2 frag_index=2：`2 <= 3` → 跳过 ✓
- 媒体片段 3 frag_index=3：`3 <= 3` → 跳过 ✓
- 媒体片段 4 frag_index=4：`4 > 3` → 加入

**结果**：所有已完成片段都正确跳过。

#### 4.7.4 片段文件生命周期：Frag1 是否保留？

在分析重复下载和重复合并之前，先明确单个片段文件的完整生命周期：

```
文件名推导：
  ctx['filename'] = video.mp4
  ctx['tmpfilename'] = temp_name(video.mp4) = video.mp4.part

  fragment_filename = '%s-Frag%d' % (ctx['tmpfilename'], 1)
                    = video.mp4.part-Frag1

  temp_name(fragment_filename) = video.mp4.part-Frag1.part   ← 下载中的临时文件

片段下载与清理流程：
  ① HttpFD 下载片段 N：写入 video.mp4.part-FragN.part
  ② 下载成功：try_rename(video.mp4.part-FragN.part → video.mp4.part-FragN)
  ③ _read_fragment：读取 video.mp4.part-FragN 的内容
  ④ _append_fragment（L146-L155）
       ├─ write + flush → 合并到 video.mp4.part
       ├─ _write_ytdl_file
       ├─ if not keep_fragments(default False):
       │     try_remove(video.mp4.part-FragN)    ← ★ 默认删除正式片段文件
       └─ del ctx['fragment_filename_sanitized']
```

**关键结论**：
- `keep_fragments` 默认值为 `False`（[fragment.py L36](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/fragment.py#L36)）
- 合并后 `video.mp4.part-FragN`（正式片段文件）会被 `try_remove` 删除
- 因此**恢复时已合并的 FragN 文件不存在**，需要重新下载
- 只有 `.part-FragN.part`（下载中的临时文件）可能残留（如果上次在片段下载中途中断）

#### 4.7.5 重复下载和重复合并的完整时序（有缺陷）

恢复时 M=3，`.part` 文件已包含片段 1（初始化）+ 2 + 3 的数据，且 `keep_fragments=False`（默认）：

```
恢复启动
  │
  ├─ .ytdl 中 M=3，ctx['fragment_index']=3，state['fragment_index']=3
  │
  ├─ 构造 fragments 列表
  │   ├─ 初始化片段 frag_index=1：无跳过判断 → 加入
  │   ├─ 媒体片段 1 frag_index=2：2 <= 3 → 跳过
  │   ├─ 媒体片段 2 frag_index=3：3 <= 3 → 跳过
  │   └─ 媒体片段 3 frag_index=4：4 > 3 → 加入
  │
  └─ download_and_append_fragments 处理
       │
       ├─ 处理初始化片段（frag_index=1）
       │   │
       │   ├─ L443: ctx['fragment_index'] = 1
       │   │
       │   ├─ _download_fragment：
       │   │     ├─ fragment_filename = video.mp4.part-Frag1
       │   │     ├─ 检测 temp_name(Frag1) = video.mp4.part-Frag1.part → 不存在（上次
       │   │     │     合并后 Frag1 已删除，.part-Frag1.part 也不存在）
       │   │     ├─ frag_resume_len = 0
       │   │     └─ ctx['dl'].download(Frag1, ...)
       │   │           ├─ ★ 重新下载整个片段（★ 重复下载！）
       │   │           ├─ 写入 video.mp4.part-Frag1.part
       │   │           ├─ 下载成功：rename(.part-Frag1.part → .part-Frag1)
       │   │           └─ progress hook 触发：
       │   │                 state.fragment_index = 3+1 = 4
       │   │                 ctx.fragment_index = 4
       │   │
       │   └─ 合并：_append_fragment
       │         ├─ _read_fragment → 读取 video.mp4.part-Frag1 内容（init 段）
       │         ├─ ctx['dest_stream'].write(init_data)  // ★ 重复写入！open_mode='ab'
       │         ├─ flush()
       │         ├─ _write_ytdl_file(ctx) → 写入 M=4  // ★ M 被错误更新！
       │         ├─ try_remove(video.mp4.part-Frag1)    // 删除重新下载的 Frag1
       │         └─ 此时 .part = init + 2 + 3 + init（损坏！）
       │
       └─ 处理媒体片段 3（frag_index=4）
           ├─ L443: ctx['fragment_index'] = 4
           ├─ _download_fragment → 下载 Frag4（正常首次下载，无重复）
           ├─ progress hook: state.fragment_index = 4+1 = 5，ctx.fragment_index = 5
           ├─ 合并 → _write_ytdl_file → 写入 M=5
           └─ 此时 M=5：表示片段 1..5 已完成，但实际上片段 2,3（媒体片段 1,2）
                被跳过但 M 也统计了它们（1=初始化，4=媒体片段 3，5=媒体片段 4），
                实际合并的是：初始化片段 + 媒体片段 3 + 媒体片段 4 = 3 个片段，
                但 M=5 与实际不一致（因为媒体片段 1,2 被跳过未重新合并，
                但 M 的自增是按 progress hook 触发次数，不是按实际合并次数）
```

**缺陷 1：重复下载**
- 默认 `keep_fragments=False`，合并后 Frag1 文件已被删除
- 恢复时 `frag_resume_len = 0`，初始化片段从 0 字节**重新完整下载**
- 若设置了 `keep_fragments=True`，Frag1 文件保留，HttpFD 检测到文件已完整则立即返回（不重复下载），但仍会重复合并

**缺陷 2：重复合并**
- `.part` 文件以 `ab` 追加模式打开（`open_mode='ab'`，因为 `.part` 文件已有数据且 `resume_len > 0`）
- 初始化片段数据被**再次追加**到 `.part` 文件末尾
- 对于 fMP4 格式，重复的 init 段会导致文件**无法播放**
- 对于 MPEG-TS 格式，可能表现为播放开始处短暂卡顿

**缺陷 3：M 与实际合并进度不一致**
- `state.fragment_index` 自增的是**全局计数器**，基于 progress hook 触发次数
- 处理初始化片段后，M 从 3 → 4，语义上表示"片段 1..4 已完成"
- 但实际上：片段 1 被重复合并、片段 2,3 被跳过（未重新合并）、片段 4 尚未处理
- M 的语义（"最后一个已安全合并的片段索引"）在**恢复场景下被破坏**

#### 4.7.6 DASH 的正确处理对比

DASH 对所有片段（包括初始化片段）使用**统一的跳过判断** [dash.py L78-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/dash.py#L78-L82)：

```python
# dash.py L78-L82
frag_index = 0
for i, fragment in enumerate(fragments):
    frag_index += 1
    if frag_index <= ctx['fragment_index']:
        continue   # ← 所有片段统一处理，无例外
```

DASH 不存在初始化片段跳过判断缺失的问题。

#### 4.7.7 extra_state 防重复机制参考

其他分段下载器（ism.py、mhtml.py）使用 `extra_state` 记录特殊头部是否已写入，避免恢复时重复写入：

```python
# ism.py L247-L272
extra_state = ctx.setdefault('extra_state', {
    'ism_track_written': False,
})
...
if not extra_state['ism_track_written']:
    write_piff_header(ctx['dest_stream'], info_dict['_download_params'])
    extra_state['ism_track_written'] = True
```

`extra_state` 会被持久化到 `.ytdl` 中，恢复后可以正确跳过已写入的特殊头部。但 HLS 中虽然设置了 `extra_state = ctx.setdefault('extra_state', {})`，却**没有用它记录初始化片段是否已写入**。

#### 4.7.8 修复思路（潜在）

为 HLS 初始化片段添加与普通媒体片段相同的跳过判断：

```python
# hls.py L233-L263（修改后）
elif line.startswith('#EXT-X-MAP'):
    if format_index is not None and discontinuity_count != format_index:
        continue
    if frag_index > 0:
        self.report_error(...)
        return False
    frag_index += 1
    if frag_index <= ctx['fragment_index']:  # ← 添加此行
        continue                              # ← 添加此行
    ...
    fragments.append(...)
```

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
            │   ├─ ① 探测 .part 文件大小 → resume_len（已完成片段 1..M 的字节总和）
            │   ├─ ② 读取 .ytdl JSON → ctx['fragment_index'] = M（最后一个已安全合并的片段索引）
            │   ├─ ③ 一致性校验（.ytdl 损坏？M > 0 但 resume_len == 0？）
            │   │   ├─ 不通过 → ❌ 清零重下
            │   │   └─ 通过 → ✅
            │   └─ ④ 跳过所有 frag_index <= M 的片段（1..M 已完成）
            │
            └─ 【第二层：字节级】针对每个待下载片段 N（从 M+1 开始）
                ├─ ① 探测 <name>.part-Frag<N> 临时文件 → frag_resume_len（该片段已下载字节）
                ├─ ② 解析 manifest byte_range（req_start, req_end）
                ├─ ③ 叠加：最终 Range = bytes=(req_start + frag_resume_len)-req_end
                ├─ ④ 下载片段；完成后 progress hook 自增 state.fragment_index
                ├─ ⑤ 合并片段内容到 .part + flush
                └─ ⑥ 写 .ytdl 持久化 M = N（表示片段 1..N 已安全合并）
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
