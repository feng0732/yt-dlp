# yt-dlp 后处理器链执行顺序分析

## 1. 核心架构概述

后处理器（PostProcessor）是 yt-dlp 下载完成后对视频/音频进行后续处理的核心组件。后处理器链的执行由 [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py) 编排，所有后处理器继承自 [PostProcessor](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/common.py#L36-L150) 基类。

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

### 2.2 阶段执行顺序与调用点

| 阶段 | 调用时机 | 调用位置 |
|------|---------|---------|
| `playlist` | 播放列表处理完成后 | [YoutubeDL.py:2190](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L2190-L2190) |
| `pre_process` | 信息提取后、格式过滤前 | [YoutubeDL.py:3832](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3832-L3832) |
| `after_filter` | 格式过滤后、下载前 | [YoutubeDL.py:3040](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3040-L3040) |
| `video` | 视频信息确认后（不执行 PP，仅用于打印） | [YoutubeDL.py:3261](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3261-L3261) |
| `before_dl` | 实际下载开始前 | [YoutubeDL.py:3449](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3449-L3449) |
| `post_process` | 下载完成后（**最常用阶段**） | [YoutubeDL.py:3843](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3843-L3843) |
| `after_move` | 文件移动到最终目录后 | [YoutubeDL.py:3846](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3846-L3846) |
| `after_video` | 单个视频全部处理完成后 | [YoutubeDL.py:3146](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3146-L3146) |

### 2.3 链执行核心方法

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
- `additional_pps` 先于静态链执行
- 顺序执行，前一个的输出作为后一个的输入
- `info` 字典在链中传递和修改

#### [post_process](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3839-L3846)

下载后的完整后处理流程：

```python
def post_process(self, filename, info, files_to_move=None):
    info['filepath'] = filename
    info['__files_to_move'] = files_to_move or {}
    # 1. 执行动态后处理器 + 静态 post_process 链
    info = self.run_all_pps('post_process', info, additional_pps=info.get('__postprocessors'))
    # 2. 执行文件移动（强制插入）
    info = self.run_pp(MoveFilesAfterDownloadPP(self), info)
    del info['__files_to_move']
    # 3. 执行 after_move 链
    return self.run_all_pps('after_move', info)
```

### 2.4 动态后处理器注入

运行时可通过 `info['__postprocessors']` 动态注入后处理器，这些会在 `post_process` 阶段的**最前面**执行：

```python
# 示例：FFmpegMergerPP 的动态注入 [YoutubeDL.py:3565-3566]
if downloaded and merger.available and not self.params.get('allow_unplayable_formats'):
    info_dict['__postprocessors'].append(merger)
```

类似的还有各种 FFmpegFixupPP 在 [YoutubeDL.py:3621](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3621-L3621) 动态注入。

### 2.5 后处理器添加方法

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
# FFmpegVideoRemuxerPP: 不处理图片 [ffmpeg.py:556]
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

**示例 1：FFmpegVideoRemuxerPP [ffmpeg.py:557-562]**
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

### 3.3 常见跳过条件汇总

| 跳过原因 | 代码位置 |
|---------|---------|
| 模拟/跳过下载模式 | [common.py:121-122](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/common.py#L121-L122) |
| 媒体类型不匹配 | [common.py:127-131](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/common.py#L127-L131) |
| 已是目标格式（无需转换） | [ffmpeg.py:560-562](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L560-L562) |
| 容器格式不支持嵌入 | [ffmpeg.py:590-592](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L590-L592) |
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

    # 文件清理逻辑...
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

#### 4.3.1 `pre_process` 阶段 [YoutubeDL.py:3831-3836]

```python
try:
    info = self.run_all_pps(key, info)
except PostProcessingError as err:
    msg = f'Preprocessing: {err}'
    info.setdefault('__pending_error', msg)
    self.report_error(msg, is_error=False)
```

**行为**：异常被捕获，错误存入 `__pending_error`，**不会终止**后续流程（后续会在 `_raise_pending_errors` 检查）。

#### 4.3.2 `post_process` 阶段（调用处）[YoutubeDL.py:3656-3659]

```python
try:
    replace_info_dict(self.post_process(dl_filename, info_dict, files_to_move))
except PostProcessingError as err:
    self.report_error(f'Postprocessing: {err}')
    return  # 返回，标记当前视频处理失败
```

**行为**：如果 `run_pp` 中重新抛出异常（`ignoreerrors` 不为 `True`），则当前视频处理失败，返回错误。

### 4.4 失败传播路径图

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
                ├─ pre_process 调用处 → 存入 __pending_error → 继续 ⚠️
                │
                ├─ post_process 调用处 → 报告错误 → 视频处理失败 ❌
                │
                └─ 其他阶段 → 由上层调用者处理
```

### 4.5 文件清理规则

执行成功后，后处理器返回的 `files_to_delete` 列表中的文件会被处理：

- 如果 `keepvideo` 为 `True`：文件保留，加入 `__files_to_move`
- 否则：删除原始文件（[YoutubeDL.py:3813-3818](file:///d:/fz/0601-2/solo-dogfeeding/code/93-yt-dlp/yt_dlp/YoutubeDL.py#L3813-L3818)）

---

## 5. 完整执行流程图

```
信息提取完成
    │
    ├─ [playlist] 阶段 → 播放列表级后处理
    │
    ├─ [pre_process] 阶段 → 预提取后处理
    │    └─ 失败 → __pending_error，继续
    │
    ├─ 格式过滤
    │
    ├─ [after_filter] 阶段 → 格式过滤后处理
    │
    ├─ [video] 阶段 → 仅打印，不执行 PP
    │
    ├─ [before_dl] 阶段 → 下载前处理
    │
    ├─ 执行下载
    │
    └─ 下载成功？
         ├─ 是 → 动态注入 __postprocessors（Merger, Fixup 等）
         │    │
         │    └─ post_process() 方法
         │          ├─ additional_pps（__postprocessors）→ 先执行
         │          ├─ [post_process] 静态链 → 顺序执行
         │          ├─ MoveFilesAfterDownloadPP → 强制执行
         │          └─ [after_move] 阶段 → 文件移动后处理
         │                │
         │                ├─ 失败（ignoreerrors=True）→ 记录，继续
         │                └─ 失败（其他情况）→ 抛出 → 视频失败
         │
         └─ 否 → 报告下载错误，跳过后处理

    ├─ [after_video] 阶段 → 单个视频完成后
    │
    └─ 播放列表完成 → [playlist] 阶段
```

---

## 6. 易错点总结

1. **`additional_pps` 执行顺序**：动态添加的后处理器（如 Merger、Fixup）**先于**静态链执行，容易被忽略。

2. **`ignoreerrors` 的精确判断**：必须是 `True` 才忽略后处理错误，`'only_download'` 不生效。

3. **`pre_process` 的特殊错误处理**：错误会被暂存而不是立即终止，直到 `_raise_pending_errors` 才会检查。

4. **`MoveFilesAfterDownloadPP` 的强制插入**：在 `post_process` 和 `after_move` 之间强制执行，不经过常规链管理。

5. **`_restrict_to` 的 `simulated` 参数**：设为 `False` 时，模拟模式会跳过该后处理器。

6. **返回值约定**：跳过执行时必须返回 `([], info)` 而不是 `None`，否则元类包装会因解包失败报错。
