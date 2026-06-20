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

- `current_fragment.index`：**当前正在下载**的片段索引（0-based）
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

## 四、统一入口：FileDownloader.download() 的预检查

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

## 五、可恢复判定流程图（综合）

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
        └─ FragmentFD 路径（HLS/DASH 等分段）
            │
            ├─ ① 探测 .part 文件大小 → resume_len（已完成片段字节数）
            │
            ├─ ② 读取 .ytdl JSON → fragment_index, extra_state
            │
            ├─ ③ 一致性校验
            │   ├─ .ytdl 损坏 → ❌ 清零重下
            │   ├─ fragment_index > 0 但 .part 大小=0 → ❌ 清零重下
            │   └─ 一致 → ✅ 进入续传
            │
            ├─ ④ 跳过所有 frag_index <= ctx['fragment_index'] 的片段
            │
            ├─ ⑤ 每个片段下载时，片段文件本身也走 HttpFD 续传逻辑
            │
            └─ ⑥ 每完成一个片段 → flush .part + 写 .ytdl 更新 fragment_index
```

---

## 六、关键参数总览

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

## 七、关键文件索引

| 文件 | 主要职责 |
|---|---|
| [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/common.py) | 基类 `FileDownloader`，临时文件命名、统一入口、进度钩子 |
| [http.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/http.py) | `HttpFD`，Range 请求构造、Content-Range 校验、416 处理 |
| [fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/fragment.py) | `FragmentFD`，`.ytdl` 状态读写、片段调度、一致性校验 |
| [hls.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/hls.py) | `HlsFD`，解析 m3u8，跳过已下载片段索引 |
| [dash.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/downloader/dash.py) | `DashSegmentsFD`，解析 DASH manifest，跳过已下载片段索引 |
| [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/87-yt-dlp/yt_dlp/utils/_utils.py#L4875-L4882) | `parse_http_range()`，解析 Range/Content-Range 响应头 |
