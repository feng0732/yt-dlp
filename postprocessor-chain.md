# yt-dlp 后处理器链执行顺序分析

## 1. 核心架构概述

后处理器（PostProcessor）是 yt-dlp 下载前后对视频/音频及元数据进行处理的核心组件。后处理器链的执行由 [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py) 编排，所有后处理器继承自 [PostProcessor](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/common.py#L36-L150) 基类。

### 1.1 元类自动包装

[PostProcessorMetaClass](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/common.py#L16-L33) 元类会自动包装所有后处理器的 `run` 方法，添加进度钩子支持：

```python
@functools.wraps(func)
def run(self, info, *args, **kwargs):
    info_copy = self._copy_infodict(info)
    self._hook_progress({'status': 'started'}, info_copy)
    ret = func(self, info, *args, **kwargs)
    if ret is not None:
        _, info = ret
    self._hook_progress({'status': 'finished'}, info_copy)
    return ret
```

---

## 2. 任务编排：8 个执行阶段

后处理器按执行时机分为 8 个阶段，定义在 [_utils.py:2858](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/utils/_utils.py#L2858-L2858)：

```python
POSTPROCESS_WHEN = ('pre_process', 'after_filter', 'video', 'before_dl',
                    'post_process', 'after_move', 'after_video', 'playlist')
```

### 2.1 阶段存储结构

[YoutubeDL.__init__](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L640-L640) 中初始化链存储：

```python
self._pps = {k: [] for k in POSTPROCESS_WHEN}
```

### 2.2 阶段执行方式分类

后处理器链有三种调用方式，每种方式的错误处理行为不同：

| 调用方式 | 包含阶段 | 错误处理 |
|---------|---------|---------|
| `pre_process()` 包装 | `pre_process`, `after_filter`, `video`, `before_dl` | 错误暂存到 `__pending_error`，不立即终止 |
| `post_process()` 方法 | `post_process`, `after_move` | 错误直接抛出，外层 try-catch 处理 |
| 直接 `run_all_pps()` | `after_video`, `playlist` | 错误直接抛出，由调用者处理 |

### 2.3 各阶段调用点详解

#### 方式一：通过 `pre_process()` 方法调用（4 个阶段）

[pre_process](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3828-L3837) 方法统一处理错误暂存：

```python
def pre_process(self, ie_info, key='pre_process', files_to_move=None):
    info = dict(ie_info)
    info['__files_to_move'] = files_to_move or {}
    try:
        info = self.run_all_pps(key, info)
    except PostProcessingError as err:
        msg = f'Preprocessing: {err}'
        info.setdefault('__pending_error', msg)
        self.report_error(msg, is_error=False)
    return info, info.pop('__files_to_move', None)
```

**4 个阶段的调用点：**

| 阶段 | 调用位置 | 调用时机 |
|------|---------|---------|
| `pre_process` | [YoutubeDL.py:3034](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3034-L3034) | 信息提取后、格式匹配前 |
| `after_filter` | [YoutubeDL.py:3040](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3040-L3040) | 格式过滤后、下载前 |
| `video` | [YoutubeDL.py:3354](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3354-L3354) | 单视频处理入口、文件名确定前 |
| `before_dl` | [YoutubeDL.py:3449](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3449-L3449) | 字幕/缩略图写入后、实际下载前 |

> **重要更正**：`video` 阶段**会执行后处理器**，并非仅用于打印。它通过 `pre_process` 方法调用，具有错误暂存机制。

#### 方式二：通过 `post_process()` 方法调用（2 个阶段）

[post_process](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3839-L3846) 方法包含下载后的完整处理流程：

```python
def post_process(self, filename, info, files_to_move=None):
    info['filepath'] = filename
    info['__files_to_move'] = files_to_move or {}
    # 1. 动态后处理器 + 静态 post_process 链
    info = self.run_all_pps('post_process', info, additional_pps=info.get('__postprocessors'))
    # 2. 强制执行文件移动（不在链中，硬编码插入）
    info = self.run_pp(MoveFilesAfterDownloadPP(self), info)
    del info['__files_to_move']
    # 3. after_move 链
    return self.run_all_pps('after_move', info)
```

**关键点**：
- `MoveFilesAfterDownloadPP` 是**硬编码插入**的，不在 `self._pps` 链中
- 两个阶段之间隔着文件移动操作
- 错误会直接抛出，不会暂存

#### 方式三：直接调用 `run_all_pps()`（2 个阶段）

| 阶段 | 调用位置 | 调用时机 |
|------|---------|---------|
| `after_video` | [YoutubeDL.py:3146](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3146-L3146) | 单个视频所有格式处理完成后 |
| `playlist` | [YoutubeDL.py:2190](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L2190-L2190) | 整个播放列表处理完成后 |

这两个阶段直接调用 `run_all_pps`，错误直接抛出不暂存。

### 2.4 链执行核心方法

#### [run_all_pps](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3821-L3826)

```python
def run_all_pps(self, key, info, *, additional_pps=None):
    if key != 'video':
        self._forceprint(key, info)
    for pp in (additional_pps or []) + self._pps[key]:
        info = self.run_pp(pp, info)
    return info
```

**关键特性**：
- `additional_pps`（动态注入）先于静态链执行
- 顺序执行，前一个的输出作为后一个的输入
- `info` 字典在链中传递和修改
- `video` 阶段不触发 `_forceprint`（打印在 `__forced_printings` 中单独处理）

### 2.5 动态后处理器注入

运行时可通过 `info['__postprocessors']` 动态注入后处理器，这些**只作用于 `post_process` 阶段**，且在静态链**之前**执行：

```python
# 示例：FFmpegMergerPP 的动态注入 [YoutubeDL.py:3565-3566]
if downloaded and merger.available and not self.params.get('allow_unplayable_formats'):
    info_dict['__postprocessors'].append(merger)
```

类似的还有各种 FFmpegFixupPP 在 [YoutubeDL.py:3621](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3621-L3621) 动态注入。

### 2.6 后处理器添加方法

[add_post_processor](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L942-L946)：

```python
def add_post_processor(self, pp, when='post_process'):
    assert when in POSTPROCESS_WHEN, f'Invalid when={when}'
    self._pps[when].append(pp)
    pp.set_downloader(self)
```

---

## 3. 条件跳过规则

后处理器的跳过机制分为三层：

### 3.1 基类装饰器：`_restrict_to`

[PostProcessor._restrict_to](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/common.py#L115-L133) 是最高优先级的跳过机制：

```python
@staticmethod
def _restrict_to(*, video=True, audio=True, images=True, simulated=True):
    allowed = {'video': video, 'audio': audio, 'images': images}

    def decorator(func):
        @functools.wraps(func)
        def wrapper(self, info):
            # 模拟/跳过下载模式检查
            if not simulated and (self.get_param('simulate') or self.get_param('skip_download')):
                return [], info

            # 格式类型判断
            format_type = (
                'video' if info.get('vcodec') != 'none'
                else 'audio' if info.get('acodec') != 'none'
                else 'images')

            if allowed[format_type]:
                return func(self, info)
            else:
                self.to_screen(f'Skipping {format_type}')
                return [], info
        return wrapper
    return decorator
```

**使用示例**：

```python
# FFmpegVideoConvertorPP: 不处理图片 [ffmpeg.py:556]
@PostProcessor._restrict_to(images=False)
def run(self, info): ...

# FFmpegFixupStretchedPP: 只处理视频 [ffmpeg.py:861]
@PostProcessor._restrict_to(images=False, audio=False)
def run(self, info): ...

# FFmpegMergerPP: 不处理图片 [ffmpeg.py:825]
@PostProcessor._restrict_to(images=False)
def run(self, info): ...
```

### 3.2 后处理器内部条件

装饰器检查通过后，具体后处理器还会进行内部条件判断，不满足则返回 `([], info)` 跳过。

**示例 1：FFmpegVideoConvertorPP [ffmpeg.py:557-562]**
```python
def run(self, info):
    filename, source_ext = info['filepath'], info['ext'].lower()
    target_ext, _skip_msg = resolve_mapping(source_ext, self.mapping)
    if _skip_msg:
        self.to_screen(f'Not {self._ACTION} media file "{filename}"; {_skip_msg}')
        return [], info  # 跳过
```

**示例 2：FFmpegEmbedSubtitlePP [ffmpeg.py:589-596]**
```python
def run(self, info):
    if info['ext'] not in self.SUPPORTED_EXTS:
        self.to_screen(f'Subtitles can only be embedded in {", ".join(self.SUPPORTED_EXTS)} files')
        return [], info  # 跳过
    subtitles = info.get('requested_subtitles')
    if not subtitles:
        self.to_screen('There aren\'t any subtitles to embed')
        return [], info  # 跳过
```

### 3.3 常见跳过条件汇总（已核准行号）

| 跳过原因 | 代码位置（核准） |
|---------|-----------------|
| 模拟/跳过下载模式（`simulated=False` 时） | [common.py:121-122](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/common.py#L121-L122) |
| 媒体类型不匹配（如 images=False 时处理图片） | [common.py:127-131](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/common.py#L127-L131) |
| 已是目标格式（无需转换/重封装） | [ffmpeg.py:560-562](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L560-L562) |
| 容器格式不支持嵌入字幕 | [ffmpeg.py:590-592](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L590-L592) |
| 没有字幕需要嵌入 | [ffmpeg.py:593-596](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L593-L596) |

---

## 4. 失败传播规则

### 4.1 核心错误处理方法：[run_pp](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3798-L3819)

```python
def run_pp(self, pp, infodict):
    files_to_delete = []
    if '__files_to_move' not in infodict:
        infodict['__files_to_move'] = {}
    try:
        files_to_delete, infodict = pp.run(infodict)
    except PostProcessingError as e:
        # Must be True and not 'only_download'
        if self.params.get('ignoreerrors') is True:
            self.report_error(e)
            return infodict  # 继续执行
        raise  # 终止链执行

    if not files_to_delete:
        return infodict
    if self.params.get('keepvideo', False):
        for f in files_to_delete:
            infodict['__files_to_move'].setdefault(f, '')
    else:
        self._delete_downloaded_files(
            *files_to_delete, info=infodict, msg='Deleting original file %s (pass -k to keep)')
    return infodict
```

### 4.2 `ignoreerrors` 参数的三种模式

| 值 | 含义 | 后处理错误行为 |
|----|------|---------------|
| `False` | 不忽略任何错误（API 默认） | ❌ 立即抛出异常，终止链 |
| `'only_download'` | 仅忽略下载错误（CLI 默认） | ❌ 立即抛出异常，终止链 |
| `True` | 忽略所有错误 | ✅ 记录错误，继续下一个 PP |

> **关键细节**：`ignoreerrors` 必须是 `True` 才会忽略后处理错误，`'only_download'` **不会**忽略后处理错误。

### 4.3 不同阶段的错误处理差异

#### 4.3.1 `pre_process` 包装的 4 个阶段

通过 `pre_process()` 方法调用的 4 个阶段（`pre_process`、`after_filter`、`video`、`before_dl`）有统一的错误暂存机制：

```python
# [YoutubeDL.py:3831-3836]
try:
    info = self.run_all_pps(key, info)
except PostProcessingError as err:
    msg = f'Preprocessing: {err}'
    info.setdefault('__pending_error', msg)
    self.report_error(msg, is_error=False)
```

**行为特点**：
- 异常被捕获，错误存入 `__pending_error`
- 用 `is_error=False` 调用 `report_error`，不会立即终止
- 后续流程继续执行

### 4.4 暂存错误的重新抛出机制（4 个检查点详解）

[_raise_pending_errors](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L2822-L2825) 定义：

```python
def _raise_pending_errors(self, info):
    err = info.pop('__pending_error', None)
    if err:
        self.report_error(err, tb=False)
```

**行为**：
- 弹出并清除 `__pending_error`（`pop` 操作，下次调用时已清空）
- 调用 `report_error`（默认 `is_error=True`）
- 若 `ignoreerrors is False`，`report_error` 内部会抛出 `DownloadError`

整个代码流程中共有 **4 个检查点**，覆盖从平铺结果到下载完成后的所有路径：

---

#### 检查点 ①：平铺结果返回后

**调用位置**：[YoutubeDL.py:1934](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L1934-L1934)

**代码上下文**：
```python
# process_ie_result 中，extract_flat 模式处理路径
info_copy, _ = self.pre_process(info_copy)       # 可能产生 __pending_error
self._fill_common_fields(info_copy, False)
self.__forced_printings(info_copy)
self._raise_pending_errors(info_copy)            # ← 检查点①
return ie_result
```

**触发场景**：`--flat-playlist` 或 `extract_flat=True` 时，不进行实际下载，仅提取播放列表条目的元信息。此时 `pre_process` 阶段暂存的错误在此处抛出。

---

#### 检查点 ②：视频结果处理完成后

**调用位置**：[YoutubeDL.py:1942](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L1942-L1942)

**代码上下文**：
```python
# process_ie_result 中，video 类型处理路径
if result_type == 'video':
    self.add_extra_info(ie_result, extra_info)
    ie_result = self.process_video_result(ie_result, download=download)  # 内部有检查点③④
    self._raise_pending_errors(ie_result)       # ← 检查点②
    additional_urls = (ie_result or {}).get('additional_urls')
```

**触发场景**：整个视频（含所有格式）的处理流程完成后，再次检查 `__pending_error`。这是**最终兜底检查点**——即使 `process_video_result` 内部某些路径未触发检查，这里也会确保暂存错误被处理。

---

#### 检查点 ③：单个格式 `process_info` 返回后

**调用位置**：[YoutubeDL.py:3132](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3132-L3132)

**代码上下文**：
```python
# process_video_result 中，对每个下载格式循环
for fmt, chapter in itertools.product(formats_to_download, requested_ranges):
    downloaded_formats.append(new_info)
    try:
        self.process_info(new_info)             # 内部有检查点④
    except MaxDownloadsReached:
        max_downloads_reached = True
    self._raise_pending_errors(new_info)        # ← 检查点③
```

**触发场景**：每个具体格式（每个视频分辨率/音频编码组合）处理完成后立即检查。如果一个视频请求了多个格式（如 DASH 分离流），每个格式都会独立触发此检查。

---

#### 检查点 ④：下载完成后、后处理之前

**调用位置**：[YoutubeDL.py:3596](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3596-L3596)

**代码上下文**：
```python
# process_info 内部
# ... 执行下载、捕获下载异常 ...
self._raise_pending_errors(info_dict)          # ← 检查点④
if success and full_filename != '-':
    fixup()                                     # 动态注入 Fixup PP
    replace_info_dict(self.post_process(dl_filename, info_dict, files_to_move))
```

**触发场景**：下载完成（或下载异常被捕获）后，**在执行 fixup 和 `post_process` 之前**检查暂存错误。如果 `before_dl` 阶段暂存了错误，会在进行任何后处理操作之前被抛出。

---

#### 4 个检查点的执行顺序

```
process_ie_result
    │
    ├─ extract_flat 模式 → 检查点①（L1934）→ return
    │
    └─ video 模式
         │
         └─ process_video_result
              │
              ├─ pre_process / after_filter 阶段 → 暂存错误
              │
              └─ 对每个格式循环
                   │
                   └─ process_info
                        │
                        ├─ _match_entry 不匹配？ ── 是 → 【路径 1】return ← 无暂存错误
                        │
                        ├─ video 阶段 → 暂存错误
                        │
                        ├─ 【组 A：路径 2-10】
                        │    ├─ simulate / 文件名缺失 / 目录创建失败 / 文件写入失败
                        │    └─ 提前 return → 跳过检查点④
                        │
                        ├─ before_dl 阶段 → 暂存错误
                        │
                        ├─ 【组 B：路径 11-13】
                        │    ├─ skip_download / 正常下载
                        │    ├─ 检查点④（L3596）← 下载后、后处理前
                        │    ├─ fixup + post_process（不暂存，直接抛出）
                        │    └─ 正常结束
                        │
                        └─ return
                   │
                   └─ 检查点③（L3132）← 每个格式处理完（所有路径都经过）
              │
              └─ 检查点②（L1942）← 整个视频处理完（兜底）
```

**检查点覆盖率说明**：

| 检查点 | 组 A（路径 1-10） | 组 B（路径 11-13） |
|--------|------------------|------------------|
| ④（L3596） | ❌ 不经过 | ✅ 经过 |
| ③（L3132） | ✅ 经过 | ✅ 经过 |
| ②（L1942） | ✅ 经过 | ✅ 经过 |
| ①（L1934） | 仅 extract_flat | 仅 extract_flat |

---

#### 4.3.2 `post_process` / `after_move` 阶段

在 `post_process` 方法内部，错误直接向上抛出，在外层调用处被捕获：

```python
# [YoutubeDL.py:3656-3659]
try:
    replace_info_dict(self.post_process(dl_filename, info_dict, files_to_move))
except PostProcessingError as err:
    self.report_error(f'Postprocessing: {err}')
    return  # 返回，标记当前视频处理失败
```

**行为**：当前视频处理失败，返回错误，但播放列表可能继续（取决于外层循环）。

#### 4.3.3 `after_video` / `playlist` 阶段

直接调用 `run_all_pps`，没有外层 try-catch，错误直接向上传播。

---

#### 4.3.4 `process_info` 中的 11 条返回路径与错误检查落点

`process_info` 是单个格式处理的核心函数，内部有 **11 条返回路径**，每条路径经过的后处理器阶段和错误检查点不同。

##### 关键代码位置

`process_info` 入口：[YoutubeDL.py:3331](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3331-L3331)

内部原地修改机制：
```python
# [YoutubeDL.py:3347-3352]
def replace_info_dict(new_info):
    nonlocal info_dict
    if new_info == info_dict:
        return
    info_dict.clear()
    info_dict.update(new_info)  # 原地修改，外层可直接读取
```

**`info_dict` 通过 `clear()` + `update()` 原地修改**，因此 `__pending_error` 等字段会同步到外层调用者的字典中。

##### 11 条返回路径详解

| 路径编号 | 返回位置 | 触发条件 | 经过的 PP 阶段 | 经过的错误检查点 |
|---------|---------|---------|--------------|----------------|
| 1 | [L3342](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3342-L3342) | `_match_entry` 不匹配（如标题过滤、日期范围过滤） | 无（`video` 阶段之前） | 检查点③ → 检查点② |
| 2 | [L3373](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3373-L3373) | `simulate` 模拟模式 | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 3 | [L3376](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3376-L3376) | `full_filename is None`（文件名缺失） | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 4 | [L3378](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3378-L3378) | `_ensure_dir_exists(full_filename)` 失败 | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 5 | [L3380](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3380-L3380) | `_ensure_dir_exists(temp_filename)` 失败 | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 6 | [L3384](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3384-L3384) | `_write_description` 失败（返回 None） | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 7 | [L3388](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3388-L3388) | `_write_subtitles` 失败（返回 None） | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 8 | [L3394](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3394-L3394) | `_write_thumbnails` 失败（返回 None） | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 9 | [L3404](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3404-L3404) | `_write_info_json` 失败（返回 None） | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 10 | [L3447](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3447-L3447) | `_write_link_file` 失败 | `video` 阶段（L3354） | 检查点③ → 检查点② |
| 11 | [L3594](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3594-L3594) | 下载异常（网络错误、文件过短等） | `video` + `before_dl` 阶段 | 检查点④ → 检查点③ → 检查点② |
| 12 | [L3659](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3659-L3659) | `post_process` 异常 | `video` + `before_dl` 阶段 | 检查点④ → 检查点③ → 检查点② |
| 13 | 函数结束 | 正常执行完成 | `video` + `before_dl` 阶段 | 检查点④ → 检查点③ → 检查点② |

> **注意**：路径 1-10 共 10 条提前返回路径，加上路径 11-13 共 13 种退出方式。文档中说"11 条返回路径"是指提前 return 的路径（L3342-L3447 共 10 条 + L3594 共 11 条），L3659 是在外层 catch 中 return，不算函数内部的 return。

---

##### 路径 2：`simulate` 模拟运行路径详解

**代码位置**：[YoutubeDL.py:3370-3373](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3370-L3373)

```python
if self.params.get('simulate'):
    info_dict['__write_download_archive'] = self.params.get('force_write_download_archive')
    check_max_downloads()
    return
```

**执行顺序**：
1. `pre_process(info_dict, 'video')` [L3354] → 执行 `video` 阶段后处理器
   - `_restrict_to` 装饰器中 `simulated=True` 的后处理器会执行
   - `simulated=False` 的后处理器会被跳过（返回 `[], info`）
2. `prepare_filename` [L3359] → 生成文件名
3. `__forced_printings` [L3364] → 强制打印
4. **return** [L3373] → 不执行后续的文件写入和下载

**错误检查落点**：
- `video` 阶段的 `__pending_error` 不会经过检查点④（L3596）
- 会经过检查点③（L3132）和检查点②（L1942）

---

##### 路径 3：文件名缺失 `full_filename is None`

**代码位置**：[YoutubeDL.py:3375-3376](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3375-L3376)

```python
if full_filename is None:
    return
```

`prepare_filename` 返回 `None` 的场景：
- 输出模板解析错误（`ValueError`），定义在 [_prepare_filename:1547-1549](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L1547-L1549)
  ```python
  except ValueError as err:
      self.report_error('Error in output template: ' + str(err) + '...')
      return None
  ```

**执行顺序**：
1. `pre_process(info_dict, 'video')` [L3354] → 执行 `video` 阶段
2. `prepare_filename` [L3359] → 返回 `None`
3. **return** [L3376]

**错误检查落点**：
- 不会执行 `before_dl` 阶段（L3449）
- 不会经过检查点④（L3596）
- 会经过检查点③（L3132）和检查点②（L1942）

---

##### 路径 4-5：目录创建失败

**代码位置**：[YoutubeDL.py:3377-3380](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3377-L3380)

```python
if not self._ensure_dir_exists(full_filename):
    return
if not self._ensure_dir_exists(temp_filename):
    return
```

`_ensure_dir_exists` 定义在 [YoutubeDL.py:2038-2044](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L2038-L2044)：
```python
def _ensure_dir_exists(self, path):
    try:
        make_parent_dirs(path)
        return True
    except OSError as e:
        self.report_error(f'Unable to create directory: {e}')
        return False
```

**触发场景**：
- 磁盘空间不足
- 权限不足
- 路径包含非法字符
- 磁盘只读

**执行顺序**：
1. `pre_process(info_dict, 'video')` [L3354] → 执行 `video` 阶段
2. `prepare_filename` [L3359] → 生成文件名
3. `_ensure_dir_exists` → 失败返回 `False`
4. **return** [L3378/L3380]

**错误检查落点**：
- 不会执行 `before_dl` 阶段
- 不会经过检查点④（L3596）
- 会经过检查点③（L3132）和检查点②（L1942）

---

##### `skip_download` 跳过下载路径详解

**注意**：`skip_download` 不是提前返回路径，而是不执行下载，但继续执行后续流程。

**代码位置**：[YoutubeDL.py:3452-3457](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3452-L3457)

```python
if self.params.get('skip_download'):
    info_dict['filepath'] = temp_filename
    info_dict['__finaldir'] = os.path.dirname(os.path.abspath(full_filename))
    info_dict['__files_to_move'] = files_to_move
    replace_info_dict(self.run_pp(MoveFilesAfterDownloadPP(self, False), info_dict))
    info_dict['__write_download_archive'] = self.params.get('force_write_download_archive')
else:
    # 正常下载流程...
```

**执行顺序**：
1. `pre_process(info_dict, 'video')` [L3354] → `video` 阶段
2. 文件写入（描述、字幕、缩略图、info.json）[L3382-3447]
3. `pre_process(info_dict, 'before_dl')` [L3449] → `before_dl` 阶段
4. `skip_download` 分支 [L3452]
   - 直接调用 `MoveFilesAfterDownloadPP` 移动文件
5. 检查点④ `_raise_pending_errors` [L3596]
6. fixup + `post_process` [L3656]

**错误检查落点**：
- 会经过检查点④（L3596）
- 会经过检查点③（L3132）和检查点②（L1942）
- `post_process` 阶段的错误不暂存，直接在外层捕获

---

##### 正常下载路径

**代码位置**：[YoutubeDL.py:3458-3659](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3458-L3659)

**执行顺序**：
1. `pre_process(info_dict, 'video')` [L3354] → `video` 阶段
2. 文件写入 [L3382-3447]
3. `pre_process(info_dict, 'before_dl')` [L3449] → `before_dl` 阶段
4. 执行下载（try-catch 包裹）[L3461-3594]
5. 检查点④ `_raise_pending_errors` [L3596]
6. fixup 后处理器动态注入 [L3598-3621]
7. `post_process` [L3656] → `post_process` + `after_move` 阶段

**错误检查落点**：
- 下载异常会被捕获，然后执行检查点④（L3596）
- 正常下载成功也会执行检查点④（L3596）
- `post_process` 阶段错误不暂存，直接在外层捕获

---

### 4.5 失败传播路径图

```
后处理器抛出 PostProcessingError
        │
        ▼
    run_pp 捕获
        │
        ├─ ignoreerrors is True → 记录错误 → 返回 info → 继续下一个 PP ✅
        │
        └─ 其他情况 → 重新 raise → 向上传递
                │
                ├─ pre_process() 包装 → 存入 __pending_error → 继续后续步骤 ⚠️
                │    （pre_process / after_filter / video / before_dl）
                │    │
                │    ├─ 【路径 1-10】提前 return（L3342-L3447）
                │    │    └─ 跳过检查点④ → 检查点③（L3132）→ 检查点②（L1942）
                │    │
                │    ├─ 【路径 11-13】下载/正常结束（L3594-函数结束）
                │    │    └─ 检查点④（L3596）→ 检查点③（L3132）→ 检查点②（L1942）
                │    │
                │    ├─ 检查点①（L1934）：平铺结果返回后
                │    └─ 检查点②（L1942）：视频结果最终返回后（兜底）
                │
                ├─ post_process() 方法 → 向上抛出 → 外层捕获 → 视频失败 ❌
                │    （post_process / after_move）
                │
                └─ 直接 run_all_pps() → 直接向上抛出 → 由调用者处理 ⚡
                     （after_video / playlist）
```

**路径分组说明**：
- **路径 1**：`_match_entry` 不匹配（L3342），无暂存错误
- **路径 2-10**：`simulate`、文件名缺失、目录创建失败、文件写入失败（L3373-L3447）
  - 只经过 `video` 阶段（L3354）
  - 不经过检查点④（L3596）
  - 由检查点③（L3132）处理
- **路径 11-13**：下载异常、`post_process` 异常、正常结束（L3594-函数结束）
  - 经过 `video` + `before_dl` 两个阶段
  - 经过检查点④（L3596）
  - 再经过检查点③（L3132）

### 4.6 文件清理规则

执行成功后，后处理器返回的 `files_to_delete` 列表中的文件会被处理：

- 如果 `keepvideo` 为 `True`：文件保留，加入 `__files_to_move`
- 否则：删除原始文件（[YoutubeDL.py:3813-3818](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3813-L3818)）

---

## 5. 完整执行流程图

```
process_ie_result
    │
    ├─ extract_flat 模式？ ── 是 → pre_process → 检查点① → return
    │
    └─ video 模式
         │
         └─ process_video_result
              │
              ├─ [pre_process] 阶段 ── 错误暂存
              │
              ├─ 格式匹配 (_match_entry)
              │
              ├─ post_extract
              │
              ├─ [after_filter] 阶段 ── 错误暂存
              │
              ├─ 格式选择
              │
              └─ 对每个下载格式循环
                   │
                   └─ process_info
                        │
                        ├─ _match_entry 匹配？ ── 否 → 【路径 1】return → 检查点③
                        │
                        ├─ [video] 阶段 ── 错误暂存
                        │
                        ├─ prepare_filename（确定文件名）
                        │    │
                        │    ├─ full_filename is None？ ── 是 → 【路径 3】return → 检查点③
                        │    │
                        │    ├─ _ensure_dir_exists 失败？ ── 是 → 【路径 4-5】return → 检查点③
                        │    │
                        │    └─ simulate 模式？ ── 是 → 【路径 2】return → 检查点③
                        │
                        ├─ 写入描述/字幕/缩略图/info.json/链接文件
                        │    │
                        │    └─ 任一写入失败？ ── 是 → 【路径 6-10】return → 检查点③
                        │
                        ├─ [before_dl] 阶段 ── 错误暂存
                        │
                        ├─ skip_download？
                        │    │
                        │    ├─ 是 → 直接执行 MoveFilesAfterDownloadPP
                        │    │
                        │    └─ 否 → 执行下载（try-catch 捕获异常）
                        │         │
                        │         └─ 下载异常？ ── 是 → 【路径 11】
                        │
                        ├─ 检查点④：_raise_pending_errors ← 后处理之前
                        │    │
                        │    └─ （路径 2-10 不经过这里）
                        │
                        ├─ 下载成功？
                        │    │
                        │    ├─ 否 → return → 检查点③
                        │    │
                        │    └─ 是 → 动态注入 __postprocessors（Merger, Fixup）
                        │         │
                        │         └─ post_process() 方法
                        │              ├─ additional_pps（__postprocessors）
                        │              ├─ [post_process] 静态链
                        │              ├─ MoveFilesAfterDownloadPP（硬编码）
                        │              └─ [after_move] 阶段
                        │              │
                        │              └─ 异常？ ── 是 → 【路径 12】
                        │
                        └─ 【路径 13】正常结束 → 检查点③
                   │
                   └─ 检查点③：_raise_pending_errors ← 每个格式完成

    ├─ [after_video] 阶段 ── 所有格式完成后
    │
    └─ 检查点②：_raise_pending_errors ← 视频结果（兜底）
         │
         └─ additional_urls 处理
              │
              └─ 播放列表完成 → [playlist] 阶段
```

### 路径汇总表

| 路径 | 触发条件 | 经过 PP 阶段 | 经过检查点 |
|------|---------|-------------|-----------|
| 1 | `_match_entry` 不匹配 | 无 | ③ → ② |
| 2 | `simulate` 模拟模式 | `video` | ③ → ② |
| 3 | `full_filename is None` | `video` | ③ → ② |
| 4-5 | 目录创建失败 | `video` | ③ → ② |
| 6-10 | 文件写入失败 | `video` | ③ → ② |
| 11 | 下载异常 | `video` + `before_dl` | ④ → ③ → ② |
| 12 | `post_process` 异常 | `video` + `before_dl` | ④ → ③ → ② |
| 13 | 正常完成 | `video` + `before_dl` | ④ → ③ → ② |

---

## 6. 易错点总结

1. **`video` 阶段会执行后处理器**：并非"仅打印"，它通过 `pre_process` 方法调用，有错误暂存机制。

2. **4 个阶段有错误暂存**：`pre_process`、`after_filter`、`video`、`before_dl` 通过 `pre_process()` 方法调用，错误暂存到 `__pending_error`，不会立即终止。

3. **暂存错误有 4 个检查点**（按执行顺序）：
   - 检查点④ [L3596]：下载完成后、后处理之前（最内层、最早触发）
   - 检查点③ [L3132]：每个格式 `process_info` 返回后
   - 检查点① [L1934]：`extract_flat` 平铺结果返回后
   - 检查点② [L1942]：视频结果最终返回后（兜底）

4. **`_raise_pending_errors` 会清空错误**：使用 `pop` 操作，错误被处理后不会重复抛出。

5. **`additional_pps` 执行顺序**：动态添加的后处理器（如 Merger、Fixup）**先于**静态链执行，且只作用于 `post_process` 阶段。

6. **`ignoreerrors` 的精确判断**：必须是 `True` 才忽略后处理错误，`'only_download'` 不生效。

7. **`MoveFilesAfterDownloadPP` 的硬编码插入**：在 `post_process` 和 `after_move` 之间强制执行，不经过常规链管理。`skip_download` 模式下会更早执行（`before_dl` 之后直接调用）。

8. **`_restrict_to` 的 `simulated` 参数**：设为 `False` 时，模拟模式会跳过该后处理器。

9. **返回值约定**：跳过执行时必须返回 `([], info)` 而不是 `None`，否则元类包装会因解包失败报错。

10. **`simulate` 模式的检查点跳跃**：`simulate` 模式下 `process_info` 在 L3373 提前返回，**不会经过检查点④**（L3596），暂存错误会推迟到检查点③（L3132）才处理。

11. **文件名缺失/目录创建失败也会跳过检查点④**：`full_filename is None` 或 `_ensure_dir_exists` 失败时，同样在 `before_dl` 阶段之前返回，不会执行检查点④。

12. **13 条退出路径分两组**：
    - **组 A（路径 1-10）**：`before_dl` 阶段之前返回，只经过 `video` 阶段，不经过检查点④
    - **组 B（路径 11-13）**：经过完整流程，经过 `video` + `before_dl` 两个阶段，会经过检查点④

13. **`info_dict` 原地修改机制**：通过 `clear()` + `update()` 原地修改，外层调用者无需重新赋值即可读取 `__pending_error`。

14. **`_match_entry` 不匹配路径最特殊**：在 `video` 阶段之前返回，不会执行任何后处理器，也不会产生 `__pending_error`。

15. **`skip_download` 不是提前返回**：不执行下载，但会继续执行 `before_dl` 阶段、检查点④和 `post_process`，错误检查路径与正常下载完全相同。
