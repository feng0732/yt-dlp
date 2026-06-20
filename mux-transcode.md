# yt-dlp 音视频混流与转码主流程接入分析

## 一、总览架构

yt-dlp 的音视频混流（Muxing）与转码（Transcoding）流程是一个**多阶段、流水线式**的处理系统，核心由以下几层构成：

```
CLI 参数解析
    ↓
后处理器配置构建 (get_postprocessors)
    ↓
格式选择与合并决策 (process_video_result → _merge)
    ↓
下载执行 (支持两种策略: 分步下载后合并 / FFmpegFD 直接边下边合并)
    ↓
后处理器调度 (post_process → run_all_pps → run_pp)
    ↓
FFmpeg 外部工具调用 (real_run_ffmpeg)
    ↓
产物替换与信息更新 (filepath/ext/__files_to_move)
    ↓
文件搬移与最终落盘 (MoveFilesAfterDownloadPP)
```

**关键代码入口文件（仓库内路径）：**
- 主流程调度：[YoutubeDL.py](yt_dlp/YoutubeDL.py)
- 后处理器基类：[postprocessor/common.py](yt_dlp/postprocessor/common.py)
- FFmpeg 系列后处理器：[postprocessor/ffmpeg.py](yt_dlp/postprocessor/ffmpeg.py)
- 后处理器注册：[postprocessor/__init__.py](yt_dlp/postprocessor/__init__.py)
- 文件搬移 PP：[postprocessor/movefilesafterdownload.py](yt_dlp/postprocessor/movefilesafterdownload.py)
- CLI 参数转后处理器配置：[__init__.py](yt_dlp/__init__.py#L627-L729)
- FFmpeg 直接下载合并：[downloader/external.py](yt_dlp/downloader/external.py#L372-L571)

---

## 二、阶段一：CLI 参数 → 后处理器配置

### 2.1 后处理器执行时机 (POSTPROCESS_WHEN)

yt-dlp 定义了 **8 个后处理时机**，决定每个 PP 在流水线哪个阶段执行：

```python
# yt_dlp/utils/_utils.py:2858
POSTPROCESS_WHEN = ('pre_process', 'after_filter', 'video', 'before_dl',
                     'post_process', 'after_move', 'after_video', 'playlist')
```

**代码来源：** [utils/_utils.py:2858](yt_dlp/utils/_utils.py#L2858-L2858)

各时机含义：
| 时机 | 触发点 | 典型用途 |
|------|--------|----------|
| `pre_process` | 信息提取后、下载前 | 元数据预处理 |
| `after_filter` | 格式过滤之后 | SponsorBlock 处理 |
| `video` | 每个视频处理前 | 强制打印信息 |
| `before_dl` | 实际下载之前 | 字幕/缩略图格式转换 |
| **`post_process`** | **下载完成后（核心）** | **混流、转码、嵌入字幕、音频提取** |
| `after_move` | 文件移动到最终位置后 | 元数据写入 xattr |
| `after_video` | 单个视频所有处理完成 | - |
| `playlist` | 整个播放列表完成后 | 多文件拼接 Concat |

### 2.2 CLI 参数如何转化为后处理器

在 `get_postprocessors()` 函数中，命令行参数被转化为后处理器配置字典列表。

**代码来源：** [__init__.py:627-729](yt_dlp/__init__.py#L627-L729)

**核心映射关系：**

| CLI 参数 | 后处理器 Key | 构造参数 |
|----------|-------------|----------|
| `-x/--extract-audio` | `FFmpegExtractAudio` | `preferredcodec`, `preferredquality`, `nopostoverwrites` |
| `--remux-video FORMAT` | `FFmpegVideoRemuxer` | `preferedformat` |
| `--recode-video FORMAT` | `FFmpegVideoConvertor` | `preferedformat` |
| `--merge-output-format` | **非 PP，直接参数** | 影响合并容器选择 |
| `--embed-subs` | `FFmpegEmbedSubtitle` | `already_have_subtitle` |
| `--embed-metadata` | `FFmpegMetadata` | `add_metadata`, `add_chapters`, `add_infojson` |

**示例：**

```python
# 当用户执行: yt-dlp -x --audio-format mp3 --recode-video mp4 URL
# get_postprocessors() 会 yield:
[
    {'key': 'FFmpegExtractAudio', 'preferredcodec': 'mp3', 'preferredquality': '5', ...},
    {'key': 'FFmpegVideoConvertor', 'preferedformat': 'mp4'},
    # ... 其他 PP
]
```

### 2.3 后处理器实例化与注册

在 YoutubeDL 初始化时，配置字典被实例化为后处理器对象：

**代码来源：** [YoutubeDL.py:827-837](yt_dlp/YoutubeDL.py#L827-L837)

```python
for pp_def_raw in self.params.get('postprocessors', []):
    pp_def = dict(pp_def_raw)
    when = pp_def.pop('when', 'post_process')
    # Handle errors for ExecPP command validation
    try:
        self.add_post_processor(
            get_postprocessor(pp_def.pop('key'))(self, **pp_def),
            when=when)
    except UnsafeExecExpansionError as e:
        self.report_error(e)
        raise
```

- `get_postprocessor(key)` → [postprocessor/__init__.py:51-52](yt_dlp/postprocessor/__init__.py#L51-L52)：从全局注册表查找 `key + 'PP'` 对应的类
- `add_post_processor(pp, when)` → [YoutubeDL.py:942-946](yt_dlp/YoutubeDL.py#L942-L946)：将 PP 加入 `self._pps[when]` 列表

```python
def add_post_processor(self, pp, when='post_process'):
    """Add a PostProcessor object to the end of the chain."""
    assert when in POSTPROCESS_WHEN, f'Invalid when={when}'
    self._pps[when].append(pp)
    pp.set_downloader(self)
```

---

## 三、阶段二：格式选择与合并决策

### 3.1 何时触发格式合并

当用户的格式选择器（`-f` 参数）选择了**多个需要组合的格式**时（例如 `bestvideo+bestaudio`），格式选择系统会生成一个包含 `requested_formats` 字段的 info_dict。

**代码来源：** `_merge()` 函数 [YoutubeDL.py:2450-2522](yt_dlp/YoutubeDL.py#L2450-L2522)

```python
def _merge(formats_pair):
    format_1, format_2 = formats_pair
    formats_info = []
    formats_info.extend(format_1.get('requested_formats', (format_1,)))
    formats_info.extend(format_2.get('requested_formats', (format_2,)))

    # 过滤掉多余的同类型流（除非 allow_multiple_streams 开启）
    # ...

    # 计算兼容的输出容器
    output_ext = get_compatible_ext(
        vcodecs=[f.get('vcodec') for f in video_fmts],
        acodecs=[f.get('acodec') for f in audio_fmts],
        vexts=[f['ext'] for f in video_fmts],
        aexts=[f['ext'] for f in audio_fmts],
        preferences=(try_call(lambda: self.params['merge_output_format'].split('/'))
                     or (self.params.get('prefer_free_formats') and ('webm', 'mkv'))))

    new_dict = {
        'requested_formats': formats_info,  # 标记为多格式
        'ext': output_ext,                   # 合并后的目标扩展名
        # ... 其他聚合字段
    }
    return new_dict
```

### 3.2 输出容器兼容性判定

`get_compatible_ext()`：[utils/_utils.py:3088-3126](yt_dlp/utils/_utils.py#L3088-L3126)

判定规则（按优先级）：
1. **多流场景**：如果音频或视频流数量 >1，且 `mkv` 在偏好中 → 直接返回 `mkv`（唯一支持多轨的格式）
2. **编解码器兼容表**：检查 `{mp4, webm}` 的 `COMPATIBLE_CODECS` 集合，优先匹配偏好列表
3. **扩展名集合兼容**：检查是否属于 `{mp4/m4a/mov...}` 或 `{webm/weba}` 同族
4. **兜底策略**：返回 `mkv`（万能容器）或用户偏好列表最后一项

---

## 四、阶段三：下载执行与合并策略

### 4.1 两种合并路径

当 `info_dict['requested_formats']` 存在时，有两条路径完成格式合并：

```
                        ┌─ requested_formats 存在 ─┐
                        │                           │
              FFmpegFD.can_merge_formats()?      否则：分步下载
                  /            \                     │
               Yes              No                   │
                │               │                    │
    ┌───────────▼──────┐  ┌─────▼─────────────┐     │
    │ FFmpegFD 直接合并 │  │ 各格式分别独立下载  │     │
    │ (边下载边混流)     │  │  → f137.mp4, f140.m4a │  │
    └──────────────────┘  └─────────┬───────────┘    │
                                    │                │
                                    └───────┬────────┘
                                            ▼
                          FFmpegMergerPP 合并后处理
                          (将独立文件合成为一个)
```

**路径判断代码：** [YoutubeDL.py:3482-3569](yt_dlp/YoutubeDL.py#L3482-L3569)

### 4.2 路径 A：FFmpegFD 直接合并下载

当以下条件满足时，FFmpegFD 直接接管下载和合并：

```python
# downloader/external.py:387-393
FFmpegFD.can_merge_formats(info_dict, params) = (
    info_dict.get('requested_formats')        # 有多个待合并格式
    and info_dict.get('protocol')             # 有统一协议标识
    and not params.get('allow_unplayable_formats')
    and 'no-direct-merge' not in params.get('compat_opts', [])
    and cls.can_download(info_dict)           # FFmpegFD 支持协议
)
```

**代码来源：** [downloader/external.py:387-393](yt_dlp/downloader/external.py#L387-L393)

**FFmpegFD 合并实现：** [downloader/external.py:395-571](yt_dlp/downloader/external.py#L395-L571)

核心参数构建：
```python
# 为每个 requested_format 添加 -i 输入
for i, fmt in enumerate(selected_formats):
    args += [..., '-i', url]

# 为每个格式配置流映射
for i, fmt in enumerate(selected_formats):
    stream_number = fmt.get('manifest_stream_number', 0)
    args.extend(['-map', f'{i}:{stream_number}'])

# 流复制模式（无需转码）
args += ['-c', 'copy']
```

**优点**：一次 ffmpeg 调用完成下载+合并，效率高，无临时中间文件
**缺点**：仅适用于 FFmpeg 能直接读取的协议（http(s), m3u8, rtsp 等）

### 4.3 路径 B：独立下载 + FFmpegMergerPP 后处理合并

当 FFmpegFD 无法直接合并（协议不支持、需要特殊下载器等）时：

1. **为每个格式分配独立文件名**：[YoutubeDL.py:3519-3524](yt_dlp/YoutubeDL.py#L3519-L3524)
```python
for f in info_dict['requested_formats']:
    f['filepath'] = fname = prepend_extension(
        correct_ext(temp_filename, info_dict['ext']),
        f'f{f["format_id"]}', info_dict['ext'])
    # e.g., 视频 title.f137.mp4, 音频 title.f140.m4a
    downloaded.append(fname)
```

2. **逐个格式下载**：循环调用 `self.dl(fname, new_info)`

3. **注册合并后处理器**：[YoutubeDL.py:3565-3567](yt_dlp/YoutubeDL.py#L3565-L3567)
```python
if downloaded and merger.available and not self.params.get('allow_unplayable_formats'):
    info_dict['__postprocessors'].append(merger)       # 加入额外 PP 列表
    info_dict['__files_to_merge'] = downloaded          # 待合并文件列表
```

> **关键点**：`__postprocessors` 是**动态注入**的后处理器列表，与 `self._pps['post_process']` 中的静态 PP 合并执行。

---

## 五、阶段四：后处理器调度执行

### 5.1 入口：post_process() 方法

**代码来源：** [YoutubeDL.py:3839-3846](yt_dlp/YoutubeDL.py#L3839-L3846)

```python
def post_process(self, filename, info, files_to_move=None):
    info['filepath'] = filename                    # 当前处理文件路径
    info['__files_to_move'] = files_to_move or {}  # 文件移动映射表
    # 先执行: 动态注入的 __postprocessors + 静态配置的 self._pps['post_process']
    info = self.run_all_pps('post_process', info,
                            additional_pps=info.get('__postprocessors'))
    # 执行: 文件移动到最终目录
    info = self.run_pp(MoveFilesAfterDownloadPP(self), info)
    del info['__files_to_move']
    # 执行: after_move 阶段 PP
    return self.run_all_pps('after_move', info)
```

### 5.2 run_all_pps() - 批量调度

**代码来源：** [YoutubeDL.py:3821-3826](yt_dlp/YoutubeDL.py#L3821-L3826)

```python
def run_all_pps(self, key, info, *, additional_pps=None):
    if key != 'video':
        self._forceprint(key, info)
    # 合并: additional_pps (动态注入) + self._pps[key] (静态配置)
    for pp in (additional_pps or []) + self._pps[key]:
        info = self.run_pp(pp, info)   # 顺序执行，info 在 PP 间链式传递
    return info
```

### 5.3 run_pp() - 单个后处理器执行

**代码来源：** [YoutubeDL.py:3798-3819](yt_dlp/YoutubeDL.py#L3798-L3819)

```python
def run_pp(self, pp, infodict):
    files_to_delete = []
    if '__files_to_move' not in infodict:
        infodict['__files_to_move'] = {}
    try:
        # 调用 PP 的 run() 方法 (被元类 PostProcessorMetaClass 包装)
        files_to_delete, infodict = pp.run(infodict)
    except PostProcessingError as e:
        if self.params.get('ignoreerrors') is True:
            self.report_error(e)
            return infodict
        raise

    if not files_to_delete:
        return infodict
    if self.params.get('keepvideo', False):
        # 用户指定 -k 保留原文件: 将其加入移动列表（最终也会输出）
        for f in files_to_delete:
            infodict['__files_to_move'].setdefault(f, '')
    else:
        # 默认: 删除原文件 (被替换的中间产物)
        self._delete_downloaded_files(
            *files_to_delete, info=infodict,
            msg='Deleting original file %s (pass -k to keep)')
    return infodict
```

### 5.4 PostProcessor 元类：自动进度钩子

**代码来源：** [common.py:16-33](yt_dlp/postprocessor/common.py#L16-L33)

```python
class PostProcessorMetaClass(type):
    @staticmethod
    def run_wrapper(func):
        @functools.wraps(func)
        def run(self, info, *args, **kwargs):
            info_copy = self._copy_infodict(info)
            self._hook_progress({'status': 'started'}, info_copy)
            ret = func(self, info, *args, **kwargs)
            if ret is not None:
                _, info = ret
            self._hook_progress({'status': 'finished'}, info_copy)
            return ret
        return run

    def __new__(cls, name, bases, attrs):
        if 'run' in attrs:
            attrs['run'] = cls.run_wrapper(attrs['run'])
        return type.__new__(cls, name, bases, attrs)
```

所有 PP 子类只要定义了 `run()` 方法，元类 `__new__` 会在类创建时自动用 `run_wrapper` 包装它，在执行前后触发进度钩子（started / finished）。`run_wrapper` 内部还会：
- 通过 `@functools.wraps(func)` 保留原函数签名与文档
- 复制一份 info_dict（`info_copy`）传给钩子，避免钩子修改真实数据
- 对返回值做空值安全检查（`if ret is not None`）

**基类与接口：** [common.py:36-135](yt_dlp/postprocessor/common.py#L36-L135)
- `PostProcessor` 基类定义：[common.py:36](yt_dlp/postprocessor/common.py#L36-L36)
- `_restrict_to` 装饰器：[common.py:115](yt_dlp/postprocessor/common.py#L115-L115)
- `run()` 抽象接口：[common.py:135](yt_dlp/postprocessor/common.py#L135-L135)

---

## 六、阶段五：FFmpeg 外部工具调用机制

### 6.1 FFmpegPostProcessor 初始化

**代码来源：** [ffmpeg.py:86-198](yt_dlp/postprocessor/ffmpeg.py#L86-L198)

```python
class FFmpegPostProcessor(PostProcessor):
    def __init__(self, downloader=None):
        PostProcessor.__init__(self, downloader)
        self._paths = self._determine_executables()  # 确定 ffmpeg/ffprobe 路径
```

**可执行文件定位逻辑** `_determine_executables()`：[ffmpeg.py:102-128](yt_dlp/postprocessor/ffmpeg.py#L102-L128)
1. 优先使用 `--ffmpeg-location` 参数
2. 如果是目录 → 拼接 `ffmpeg.exe` / `ffprobe.exe`
3. 如果是文件（如指定了 `custom-ffmpeg.exe`）→ 将 basename 替换为对应程序名
4. 默认可直接调用系统 PATH 中的 `ffmpeg`/`ffprobe`

### 6.2 核心调用方法：real_run_ffmpeg()

**代码来源：** [ffmpeg.py:326-364](yt_dlp/postprocessor/ffmpeg.py#L326-L364)

```python
def real_run_ffmpeg(self, input_path_opts, output_path_opts, *, expected_retcodes=(0,)):
    self.check_version()  # 校验 ffmpeg 可用性与版本

    oldest_mtime = min(
        os.stat(path).st_mtime for path, _ in input_path_opts if path)

    cmd = [self.executable, '-y']  # -y: 覆盖输出
    if self.basename == 'ffmpeg':
        cmd += ['-loglevel', 'repeat+info']

    # 为每个输入/输出文件添加参数，并合并用户自定义配置参数
    for arg_type, path_opts in (('i', input_path_opts), ('o', output_path_opts)):
        cmd += itertools.chain.from_iterable(
            make_args(path, list(opts), arg_type, i + 1)
            for i, (path, opts) in enumerate(path_opts) if path)

    self.write_debug(f'ffmpeg command line: {shell_quote(cmd)}')
    _, stderr, returncode = Popen.run(
        cmd, text=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE)

    if returncode not in variadic(expected_retcodes):
        raise FFmpegPostProcessorError(stderr.strip().splitlines()[-1])

    # 更新输出文件的时间戳 = 最早输入文件的时间戳（保留原始文件时间）
    for out_path, _ in output_path_opts:
        if out_path:
            self.try_utime(out_path, oldest_mtime, oldest_mtime)
    return stderr
```

**便捷调用封装 `run_ffmpeg`：** [ffmpeg.py:366](yt_dlp/postprocessor/ffmpeg.py#L366-L366)
- `run_ffmpeg(path, out_path, opts)` → `run_ffmpeg_multiple_files([path], out_path, opts)`
- `run_ffmpeg_multiple_files(input_paths, out_path, opts)` → `real_run_ffmpeg(...)`

**用户自定义参数注入**（`_configuration_args`）：
通过 `--postprocessor-args`（或 `--ppa`）可以注入自定义参数。
- 输入参数配置键：`_i1`, `_i2`, ..., `_i`（通用）
- 输出参数配置键：`_o1`, `_o`, `''`（第一个输出的默认键）

### 6.3 通用流复制参数：stream_copy_opts()

**代码来源：** [ffmpeg.py:213-221](yt_dlp/postprocessor/ffmpeg.py#L213-L221)

```python
@staticmethod
def stream_copy_opts(copy=True, *, ext=None):
    yield from ('-map', '0')                  # 复制第一个输入的所有流
    yield from ('-dn', '-ignore_unknown')      # 不复制数据流，忽略未知流
    if copy:
        yield from ('-c', 'copy')              # 流复制（不转码，速度快）
    if ext in ('mp4', 'mov', 'm4a'):
        yield from ('-c:s', 'mov_text')        # MP4 字幕用 mov_text 编码
```

**版本检查 `check_version`：** [ffmpeg.py:223-231](yt_dlp/postprocessor/ffmpeg.py#L223-L231)

---

## 七、核心后处理器详解

### 7.1 FFmpegMergerPP：格式合并（混流）

**代码来源：** [ffmpeg.py:822-847](yt_dlp/postprocessor/ffmpeg.py#L822-L847)

**触发条件：** `info_dict['requested_formats']` 存在且文件已独立下载完成

**处理流程：**

```python
@PostProcessor._restrict_to(images=False)
def run(self, info):
    filename = info['filepath']                    # 最终输出文件路径
    temp_filename = prepend_extension(filename, 'temp')  # title.temp.mp4
    args = ['-c', 'copy']                          # 流复制（不重新编码）
    audio_streams = 0

    for (i, fmt) in enumerate(info['requested_formats']):
        # 音频流映射
        if fmt.get('acodec') != 'none':
            args.extend(['-map', f'{i}:a:0'])       # 第i个输入的第0个音频流
            # HLS m3u8 协议下载的 AAC 需要 ADTS 头移除比特流过滤器
            aac_fixup = fmt['protocol'].startswith('m3u8') and self.get_audio_codec(fmt['filepath']) == 'aac'
            if aac_fixup:
                args.extend([f'-bsf:a:{audio_streams}', 'aac_adtstoasc'])
            audio_streams += 1
        # 视频流映射
        if fmt.get('vcodec') != 'none':
            args.extend(['-map', f'{i}:v:0'])       # 第i个输入的第0个视频流

    self.to_screen(f'Merging formats into "{filename}"')
    # 执行: ffmpeg -i f137.mp4 -i f140.m4a -c copy -map 0:v:0 -map 1:a:0 temp.mp4
    self.run_ffmpeg_multiple_files(info['__files_to_merge'], temp_filename, args)
    os.rename(temp_filename, filename)              # 原子替换
    return info['__files_to_merge'], info           # 返回待删除的中间文件列表
```

**关键参数构建示例：**
- 输入1: `f137.mp4` (纯视频 1080p)
- 输入2: `f140.m4a` (纯音频 128k AAC)
- 输出: `title.mp4`

生成的 ffmpeg 命令（简化）：
```bash
ffmpeg -y -i title.f137.mp4 -i title.f140.m4a \
       -c copy -map 0:v:0 -map 1:a:0 \
       -movflags +faststart \
       title.temp.mp4
```

### 7.2 FFmpegVideoRemuxerPP：容器重封装（无损）

**代码来源：** [ffmpeg.py:573-578](yt_dlp/postprocessor/ffmpeg.py#L573-L578)

**用途：** 更改容器格式但不重新编码（如 webm → mkv，avi → mp4）
**CLI 参数：** `--remux-video FORMAT`（支持映射规则 `aac>m4a/mov>mp4/mkv`）

```python
class FFmpegVideoRemuxerPP(FFmpegVideoConvertorPP):
    _ACTION = 'remuxing'
    @staticmethod
    def _options(target_ext):
        return FFmpegPostProcessor.stream_copy_opts()  # 关键: -c copy 流复制
```

**特点：** 调用 `stream_copy_opts()` → `-c copy`，无损且快速

### 7.3 FFmpegVideoConvertorPP：视频转码（重编码）

**代码来源：** [ffmpeg.py:538-570](yt_dlp/postprocessor/ffmpeg.py#L538-L570)

**用途：** 完全重新编码视频（如任何格式 → mp4/h264/aac）
**CLI 参数：** `--recode-video FORMAT`

```python
class FFmpegVideoConvertorPP(FFmpegPostProcessor):
    @staticmethod
    def _options(target_ext):
        yield from FFmpegPostProcessor.stream_copy_opts(False)  # copy=False 强制重新编码!
        if target_ext == 'avi':
            yield from ('-c:v', 'libxvid', '-vtag', 'XVID')     # AVI 特殊编码

    @PostProcessor._restrict_to(images=False)
    def run(self, info):
        filename, source_ext = info['filepath'], info['ext'].lower()
        target_ext, _skip_msg = resolve_mapping(source_ext, self.mapping)
        if _skip_msg:  # 已经是目标格式
            return [], info

        outpath = replace_extension(filename, target_ext, source_ext)
        self.to_screen(f'Converting video from {source_ext} to {target_ext}; Destination: {outpath}')
        self.run_ffmpeg(filename, outpath, self._options(target_ext))

        # 产物替换: 更新 filepath, ext, format
        info['filepath'] = outpath
        info['format'] = info['ext'] = target_ext
        return [filename], info  # 原文件加入删除列表
```

**关键差异：** `stream_copy_opts(False)` 不包含 `-c copy`，ffmpeg 会根据目标容器自动选择编码器重新编码。

### 7.4 FFmpegExtractAudioPP：音频提取与转换

**代码来源：** [ffmpeg.py:432-535](yt_dlp/postprocessor/ffmpeg.py#L432-L535)

**CLI 参数：** `-x/--extract-audio`, `--audio-format`, `--audio-quality`
**构造参数赋值：** `self._nopostoverwrites = nopostoverwrites` [ffmpeg.py:441](yt_dlp/postprocessor/ffmpeg.py#L441-L441)

**处理策略决策：**

```
源文件音频编码 == 目标编码?
    ├─ Yes → -c copy (无损提取，仅剥离视频流)
    └─ No  → 指定编码器重新编码 (libmp3lame, aac, libopus 等)
```

**质量参数设置 `_quality_args`：** [ffmpeg.py:443](yt_dlp/postprocessor/ffmpeg.py#L443-L443)

**产物替换（特殊逻辑）：**
当输出扩展名与原扩展名相同时（如 m4a → m4a 但需要修复封装）：

**代码来源：** [ffmpeg.py:510-529](yt_dlp/postprocessor/ffmpeg.py#L510-L529)

```python
temp_path = new_path = replace_extension(path, extension, information['ext'])

if new_path == path:
    orig_path = prepend_extension(path, 'orig')  # [ffmpeg.py:515] title.orig.m4a
    temp_path = prepend_extension(path, 'temp')  # [ffmpeg.py:516] title.temp.m4a

# nopostoverwrites 检查（跳过条件：两个目标都已存在）
if (self._nopostoverwrites and os.path.exists(new_path)
        and os.path.exists(orig_path)):
    self.to_screen(f'Post-process file {new_path} exists, skipping')
    return [], information

self.run_ffmpeg(path, temp_path, acodec, more_opts)

os.replace(path, orig_path)        # 原文件 → .orig
os.replace(temp_path, new_path)    # 临时 → 最终路径
information['filepath'] = new_path # 更新 info_dict
information['ext'] = extension
return [orig_path], information    # .orig 加入删除列表
```

### 7.5 Fixup 系列后处理器：自动修复常见问题

**基类：** `FFmpegFixupPostProcessor` [ffmpeg.py:850](yt_dlp/postprocessor/ffmpeg.py#L850-L850)

这些 PP 由 `process_info()` 中的 `fixup()` 函数**动态注册**到 `info_dict['__postprocessors']`：

| Fixup PP | 代码位置 | 触发条件 | 修复操作 |
|----------|---------|---------|----------|
| `FFmpegFixupStretchedPP` | [ffmpeg.py:860](yt_dlp/postprocessor/ffmpeg.py#L860-L860) | `stretched_ratio != 1` | 加 `-aspect` 参数修复像素比 |
| `FFmpegFixupM4aPP` | [ffmpeg.py:870](yt_dlp/postprocessor/ffmpeg.py#L870-L870) | DASH 下载的 `m4a_dash` 容器 | `-f mp4` 重新封装 |
| `FFmpegFixupM3u8PP` | [ffmpeg.py:878](yt_dlp/postprocessor/ffmpeg.py#L878-L878) | HLS native 下载且检测为 MPEG-TS 封装 | 重新封装为正确 MP4 + `aac_adtstoasc` |
| `FFmpegFixupTimestampPP` | [ffmpeg.py:901](yt_dlp/postprocessor/ffmpeg.py#L901-L901) | WebSocket 分片下载的直播 | `-bsf setts` 或 `setpts` 滤镜 |
| `FFmpegFixupDurationPP` | [ffmpeg.py:931](yt_dlp/postprocessor/ffmpeg.py#L931-L931) | WebSocket 分片下载的直播 | 流复制重写 |
| `FFmpegFixupDuplicateMoovPP` | [ffmpeg.py:935](yt_dlp/postprocessor/ffmpeg.py#L935-L935) | DASH 多周期直播 | 流复制去除重复 MOOV |

---

## 八、产物替换与信息传递机制

### 8.1 info_dict 的链式传递

后处理器之间通过**修改同一个 info_dict 对象**实现状态传递：

```
初始 info_dict:
  {filepath: 'title.f137.mp4', ext: 'mp4', requested_formats: [...], ...}
      │
      ▼ FFmpegMergerPP.run()
  {filepath: 'title.temp/title.mp4', ext: 'mp4', __files_to_merge 被删除, ...}
      │
      ▼ FFmpegVideoConvertorPP.run()
  {filepath: 'title.temp/title.mp4', ext: 'mp4', ...} → 不变 (已是mp4)
      │
      ▼ FFmpegEmbedSubtitlePP.run()
  {filepath: 'title.temp/title.mp4', ...} → 内部替换 temp，内容变了路径不变
      │
      ▼ FFmpegExtractAudioPP.run()
  {filepath: 'title.temp/title.mp3', ext: 'mp3', ...} → 路径和扩展名都更新
      │
      ▼ MoveFilesAfterDownloadPP.run()
  {filepath: 'downloads/title.mp3', ext: 'mp3', ...} → 最终落盘路径
```

**关键更新字段：**

| 字段 | 含义 | 更新方 |
|------|------|--------|
| `filepath` | 当前处理文件的绝对/相对路径 | 所有修改文件的 PP |
| `ext` | 当前文件扩展名 | 改变容器格式的 PP |
| `format` | 当前格式标识 | FFmpegVideoConvertorPP |
| `__files_to_move` | `{源路径: 目标路径}` 映射表 | run_pp, MoveFilesAfterDownloadPP |
| `__postprocessors` | 动态注入的 PP 列表 | process_info() (合并、fixup) |
| `__files_to_merge` | FFmpegMergerPP 的输入文件列表 | process_info() |
| `__real_download` | 是否实际下载（用于跳过 fixup） | 下载器、process_info() |

### 8.2 文件删除策略：run_pp() 统一管理

每个 PP 的 `run()` 返回 `(files_to_delete, updated_info)`：
- **`files_to_delete`**：被替换/废弃的中间文件列表
- **返回 `[]`**：不删除任何文件（原文件继续保留/被后续使用）

**保留/删除决策：** [YoutubeDL.py:3811-3818](yt_dlp/YoutubeDL.py#L3811-L3818)
```python
if not files_to_delete:
    return infodict
if self.params.get('keepvideo', False):
    # -k 参数：将待删除文件加入 __files_to_move (最终会保留)
    for f in files_to_delete:
        infodict['__files_to_move'].setdefault(f, '')
else:
    # 默认行为: 物理删除中间文件
    self._delete_downloaded_files(
        *files_to_delete, info=infodict,
        msg='Deleting original file %s (pass -k to keep)')
```

**删除实现 `_delete_downloaded_files()`：** [YoutubeDL.py:3774-3783](yt_dlp/YoutubeDL.py#L3774-L3783)
```python
def _delete_downloaded_files(self, *files_to_delete, info={}, msg=None):
    for filename in set(filter(None, files_to_delete)):
        if msg:
            self.to_screen(msg % filename)
        try:
            os.remove(filename)
        except OSError:
            self.report_warning(f'Unable to delete file {filename}')
        # 重要：从移动映射表中删除已删除的文件，避免后续重复操作
        if filename in info.get('__files_to_move', []):
            del info['__files_to_move'][filename]
```

### 8.3 原子替换模式

所有产生新文件的操作都遵循**临时文件 + 原子替换**模式：

```
Step 1: 生成临时文件
    ffmpeg -i input.ext → output.temp.ext

Step 2: 原子替换 (os.replace / os.rename)
    temp.ext → final.ext
```

**实现位置示例：**
- FFmpegMergerPP → `os.rename(temp_filename, filename)`
- FFmpegExtractAudioPP → `os.replace(path, orig_path); os.replace(temp_path, new_path)`
- FFmpegFixupPostProcessor._fixup → `os.replace(temp_filename, filename)`

---

## 九、阶段六：文件搬移、覆盖与保留行为深度解析

### 9.1 `overwrites` 参数的三态行为

**参数定义（含三态注释）：** [YoutubeDL.py:291-293](yt_dlp/YoutubeDL.py#L291-L293)
```python
overwrites:        Overwrite all video and metadata files if True,
                   overwrite only non-video files if None
                   and don't overwrite any file if False
```

#### 9.1.1 关键机制：None 时自动移除参数

**代码来源：** [YoutubeDL.py:778-779](yt_dlp/YoutubeDL.py#L778-L779)
```python
elif self.params.get('overwrites') is None:
    self.params.pop('overwrites', None)
```

**原理**：当用户显式设置 `overwrites=None`（或 CLI 不传任何覆盖参数）时，`overwrites` 键被从 `params` 中**完全移除**。后续所有 `params.get('overwrites', default_overwrite)` 调用都会因 key 不存在而使用传入的 `default_overwrite`，从而实现视频/非视频差异化处理。

#### 9.1.2 差异化覆盖的核心：`default_overwrite` 参数

**`existing_file()` 函数签名：** [YoutubeDL.py:3320](yt_dlp/YoutubeDL.py#L3320-L3320)
```python
def existing_file(self, filepaths, *, default_overwrite=True):
```

**实现逻辑：** [YoutubeDL.py:3320-3328](yt_dlp/YoutubeDL.py#L3320-L3328)
```python
def existing_file(self, filepaths, *, default_overwrite=True):
    existing_files = list(filter(os.path.exists, orderedSet(filepaths)))
    # 条件取反：not True = False(覆盖)，not False = True(不覆盖)
    if existing_files and not self.params.get('overwrites', default_overwrite):
        return existing_files[0]  # 返回已有文件 → 跳过下载 = 不覆盖

    for file in existing_files:
        self.report_file_delete(file)  # [YoutubeDL.py:3326]
        os.remove(file)                 # 删除现有文件 → 覆盖
    return None
```

#### 9.1.3 三态行为完整对照表

**前提条件**：overwrites=None 时 key 已被 pop，`params.get()` 始终取 `default_overwrite`

| 调用场景 | 代码位置 | default_overwrite | 取到的值 | not 结果 | 行为 |
|----------|---------|-------------------|----------|----------|------|
| **overwrites=True** | 所有场景 | - | `True`（key存在） | False | **全部覆盖** |
| **overwrites=False** | 所有场景 | - | `False`（key存在） | True | **全部不覆盖** |
| **overwrites=None** ↓ | | | | | |
| ├ 视频 existing_video_file | [YoutubeDL.py:3466-3467](yt_dlp/YoutubeDL.py#L3466-L3467) | **False** | False | True | **不覆盖，跳过下载** |
| ├ 字幕 _write_subtitles | [YoutubeDL.py:4460](yt_dlp/YoutubeDL.py#L4460-L4460) | True（默认） | True | False | **覆盖，重新下载** |
| ├ 缩略图 _write_thumbnails | [YoutubeDL.py:4524](yt_dlp/YoutubeDL.py#L4524-L4524) | True（默认） | True | False | **覆盖，重新下载** |
| ├ info.json _write_info_json | [YoutubeDL.py:4396](yt_dlp/YoutubeDL.py#L4396-L4396) | True（默认） | True | 写分支 | **覆盖，重写文件** |
| ├ description _write_description | [YoutubeDL.py:4425](yt_dlp/YoutubeDL.py#L4425-L4425) | True（默认） | True | 写分支 | **覆盖，重写文件** |
| └ 搬移 MoveFilesAfterDownloadPP | [movefilesafterdownload.py:37-38](yt_dlp/postprocessor/movefilesafterdownload.py#L37-L38) | True（默认） | True | 删除目标 | **覆盖，移动文件** |

**下载前覆盖检查的视频专用入口：** `existing_video_file()` [YoutubeDL.py:3463-3470](yt_dlp/YoutubeDL.py#L3463-L3470)
```python
def existing_video_file(*filepaths):
    ext = info_dict.get('ext')
    converted = lambda file: replace_extension(file, self.params.get('final_ext') or ext, ext)
    file = self.existing_file(
        itertools.chain(*zip(map(converted, filepaths), filepaths, strict=True)),
        default_overwrite=False)           # ★ 关键：视频默认不覆盖！
    if file:
        info_dict['ext'] = os.path.splitext(file)[1][1:]
    return file
```

**"已下载"报告：** `report_file_already_downloaded()` [YoutubeDL.py:1172-1177](yt_dlp/YoutubeDL.py#L1172-L1177)，`report_file_delete()` [YoutubeDL.py:1179-1184](yt_dlp/YoutubeDL.py#L1179-L1184)

#### 9.1.4 其他非视频文件的特殊覆盖逻辑

**Internet 快捷方式文件（link）：** [YoutubeDL.py:3418-3420](yt_dlp/YoutubeDL.py#L3418-L3420)
```python
if self.params.get('overwrites', True) and os.path.exists(linkfn):
    self.to_screen(f'[info] Internet shortcut (.{link_type}) is already present')
    return True  # 存在即视为成功，不重写也不报错
```
→ overwrites=None 时，link 文件**跳过重写**（与注释的"只覆盖非视频"有细微差异：link 属于非视频但不覆盖，属于"已存在即成功"的轻处理）

### 9.2 `__files_to_move` 完整生命周期

`__files_to_move` 是一个字典 `{源路径: 目标路径}`，贯穿整个后处理流程：

```
┌─ 初始化: process_info() [YoutubeDL.py:3361]
│    files_to_move = {}
│
├─ 初始填充（下载前辅助文件）:
│    ├─ 字幕文件: files_to_move.update(dict(sub_files)) [YoutubeDL.py:3389]
│    ├─ 缩略图:   files_to_move.update(dict(thumb_files)) [YoutubeDL.py:3395]
│    └─ info.json: 直接写入最终目录，不经过 temp
│
├─ before_dl 阶段后处理:
│    pre_process(info_dict, 'before_dl', files_to_move) [YoutubeDL.py:3449]
│    → 内部: info['__files_to_move'] = files_to_move [YoutubeDL.py:3830]
│    各 PP 可以修改 __files_to_move
│    → 返回: new_info, files_to_move (弹出值) [YoutubeDL.py:3837]
│
├─ 下载后动态追加（多格式场景）:
│    若合并后处理器未注册（ffmpeg 不可用等）:
│        for file in downloaded:
│            files_to_move[file] = None  # [YoutubeDL.py:3572] 保留中间文件
│
├─ skip_download 特殊分支:
│    info['__files_to_move'] = files_to_move [YoutubeDL.py:3455]
│    → MoveFilesAfterDownloadPP(self, False) _downloaded=False [YoutubeDL.py:3456]
│    → 不追加主文件到移动列表
│
├─ post_process 入口:
│    info['__files_to_move'] = files_to_move [YoutubeDL.py:3842]
│
├─ -k/--keepvideo 保留策略（run_pp 中每个 PP 后）:
│    if keepvideo:
│        for f in files_to_delete:
│            __files_to_move.setdefault(f, '')  # [YoutubeDL.py:3815]
│
├─ post_process → MoveFilesAfterDownloadPP 初始化时:
│    _downloaded=True 默认:
│        info['__files_to_move'][info['filepath']] = finalpath [movefiles.py:25-26]
│        加入主文件的移动映射
│
├─ MoveFilesAfterDownloadPP.run() 执行搬移:
│    遍历 __files_to_move.items(), 逐个 shutil.move
│
└─ post_process 清理:
     del info['__files_to_move']  [YoutubeDL.py:3845]
```

**pre_process 函数签名：** [YoutubeDL.py:3828-3837](yt_dlp/YoutubeDL.py#L3828-L3837)

### 9.3 MoveFilesAfterDownloadPP 核心实现

**代码来源：** [movefilesafterdownload.py:11-53](yt_dlp/postprocessor/movefilesafterdownload.py#L11-L53)

```python
class MoveFilesAfterDownloadPP(PostProcessor):

    def __init__(self, downloader=None, downloaded=True):
        PostProcessor.__init__(self, downloader)
        self._downloaded = downloaded  # 是否需要添加主文件到移动列表

    @classmethod
    def pp_key(cls):
        return 'MoveFiles'

    def run(self, info):
        dl_path, dl_name = os.path.split(info['filepath'])
        finaldir = info.get('__finaldir', dl_path)  # 目标目录（--paths 参数）
        # __finaldir 来源: prepare_filename + dir_type 计算
        # 或 skip_download 分支设置 [YoutubeDL.py:3454]
        finalpath = os.path.join(finaldir, dl_name)

        # _downloaded=True 时，把当前处理的主文件加入移动列表
        if self._downloaded:
            info['__files_to_move'][info['filepath']] = finalpath

        make_newfilename = lambda old: os.path.join(finaldir, os.path.basename(old))
        for oldfile, newfile in info['__files_to_move'].items():
            # value 为 None/空字符串时，自动生成目标路径（同文件名，仅换目录）
            if not newfile:
                newfile = make_newfilename(oldfile)

            # 源和目标相同，跳过
            if os.path.abspath(oldfile) == os.path.abspath(newfile):
                continue

            # 源文件不存在，警告跳过
            if not os.path.exists(oldfile):
                self.report_warning(f'File "{oldfile}" cannot be found')
                continue

            # ============= 覆盖逻辑核心 =============
            if os.path.exists(newfile):
                # get_param 实现: [common.py:100-103]
                #   → self._downloader.params.get(name, default)
                if self.get_param('overwrites', True):
                    # 允许覆盖: 先删除目标
                    self.report_warning(f'Replacing existing file "{newfile}"')
                    os.remove(newfile)
                else:
                    # 不允许覆盖: 跳过并警告，文件留在 temp 目录
                    self.report_warning(
                        f'Cannot move file "{oldfile}" out of temporary directory since "{newfile}" already exists. ')
                    continue
            # =====================================

            # 创建目录
            try:
                make_parent_dirs(newfile)
            except OSError as e:
                raise PostProcessingError(f'Unable to create directory: {e}') from e

            # 执行移动 (shutil.move 支持跨卷)
            self.to_screen(f'Moving file "{oldfile}" to "{newfile}"')
            shutil.move(oldfile, newfile)  # os.rename cannot move between volumes

        # ★ 更新最终产物路径（无论是否成功移动！）
        info['filepath'] = finalpath
        return [], info
```

**get_param 实现：** [common.py:100-103](yt_dlp/postprocessor/common.py#L100-L103)
```python
def get_param(self, name, default=None, *args, **kwargs):
    if self._downloader:
        return self._downloader.params.get(name, default, *args, **kwargs)
    return default
```

### 9.4 三种保留/覆盖控制参数对比

| 参数 | 级别 | 作用阶段 | 行为 | 代码位置 |
|------|------|---------|------|---------|
| **`keepvideo` (`-k`)** | 后处理 | 每个 PP 完成后 | `files_to_delete` 不删除，转为 `__files_to_move` 保留 | [YoutubeDL.py:3813-3815](yt_dlp/YoutubeDL.py#L3813-L3815) |
| **`overwrites`** (三态) | 全局 | 下载前检查 + 搬移阶段 | `True`: 全部覆盖；`None`: 仅非视频覆盖；`False`: 全不覆盖 | [YoutubeDL.py:291-293](yt_dlp/YoutubeDL.py#L291-L293) |
| **`nopostoverwrites`** | ExtractAudio | 音频转换前 | `new_path` 和 `orig_path` 都存在时跳过 | [ffmpeg.py:517-520](yt_dlp/postprocessor/ffmpeg.py#L517-L520) |

### 9.5 `keepvideo` 对产物路径的影响

**典型场景：下载视频 + 提取音频 + `-k` 保留原视频**

```
命令: yt-dlp -x --audio-format mp3 -k URL

Step 1: 下载视频 → temp/title.mp4
Step 2: FFmpegExtractAudioPP 提取音频 → temp/title.mp3
        返回 files_to_delete = ['temp/title.mp4']

Step 3: run_pp() 处理删除列表: [YoutubeDL.py:3813-3815]
        if keepvideo:
            # title.mp4 不删除，加入移动列表
            __files_to_move.setdefault('temp/title.mp4', '')

Step 4: MoveFilesAfterDownloadPP.run(): [movefilesafterdownload.py:25-26]
        先追加主文件:
          __files_to_move['temp/title.mp3'] = 'downloads/title.mp3'
        遍历 __files_to_move:
          'temp/title.mp3' → 'downloads/title.mp3' (shutil.move)
          'temp/title.mp4' → 'downloads/title.mp4' (被保留的原文件, None→拼接文件名)

最终目录:
    downloads/title.mp3 (最终产物)
    downloads/title.mp4 (被保留的原视频)
```

### 9.6 搬移失败对产物路径的影响

当 `overwrites=False` 且目标文件已存在时：

```
假设目标目录已存在 downloads/title.mp3

Step 1: 后处理阶段顺利完成，生成 temp/title.mp3
        info['filepath'] = 'temp/title.mp3'

Step 2: MoveFilesAfterDownloadPP.run():
        检测到 'downloads/title.mp3' 已存在
        overwrites=False → continue，不移动
        但 info['filepath'] 仍被更新为 finalpath = 'downloads/title.mp3'
        (即使文件实际没有移动过去!)

Step 3: 调用方获取 info['filepath'] = 'downloads/title.mp3'
        但实际文件仍在 temp/title.mp3

⚠️ 注意：这是一个潜在的状态不一致风险点
```

**验证代码：** [movefilesafterdownload.py:52](yt_dlp/postprocessor/movefilesafterdownload.py#L52-L52)
```python
info['filepath'] = finalpath  # 无论移动是否成功，都无条件更新!
return [], info
```

### 9.7 跨卷移动的处理

MoveFilesAfterDownloadPP 使用 `shutil.move()` 而非 `os.rename()`：
- `os.rename()` 无法跨卷/跨文件系统移动
- `shutil.move()` 自动 fallback 到 copy + delete 模式

**代码：** [movefilesafterdownload.py:50](yt_dlp/postprocessor/movefilesafterdownload.py#L50-L50)
```python
shutil.move(oldfile, newfile)  # os.rename cannot move between volumes
```

### 9.8 特殊场景：合并被取消时的中间文件保留

当 FFmpegMergerPP 因为 ffmpeg 不可用等原因无法执行时，已下载的中间文件会被保留：

**代码：** [YoutubeDL.py:3570-3572](yt_dlp/YoutubeDL.py#L3570-L3572)
```python
else:  # merger 不可用 或 allow_unplayable_formats
    for file in downloaded:
        files_to_move[file] = None  # 每个中间文件都加入移动列表
```

最终目录会同时存在：
```
downloads/title.f137.mp4 (视频流文件)
downloads/title.f140.m4a (音频流文件)
```

---

## 十、完整调用链路图

### 10.1 单个视频处理的完整链路（含搬移）

```
YoutubeDL.process_info(info_dict)  [YoutubeDL.py:3331]
  │
  ├─ 初始化: files_to_move = {}  [YoutubeDL.py:3361]
  │
  ├─ 字幕文件写入 + 加入 files_to_move  [YoutubeDL.py:3389]
  ├─ 缩略图下载 + 加入 files_to_move  [YoutubeDL.py:3395]
  ├─ info.json 写入最终目录 (不经 temp)
  │
  ├─ pre_process('before_dl', files_to_move)  [YoutubeDL.py:3449]
  │
  ├─ 下载分支判断:
  │   ├─ FFmpegFD.can_merge_formats()? ─Yes─► 一次调用 ffmpeg 完成下载+合并
  │   │                          (不触发 FFmpegMergerPP)
  │   └─ No:
  │       ├─ 有 requested_formats?
  │       │   └─ Yes: 分别下载各格式 → 记录 downloaded[]
  │       │           合并成功:
  │       │             → __postprocessors.append(FFmpegMergerPP) [YoutubeDL.py:3566]
  │       │             → __files_to_merge = downloaded
  │       │           合并失败(ffmpeg不可用等):
  │       │             → files_to_move[file] = None [YoutubeDL.py:3572]
  │       └─ No: 直接下载单个文件
  │
  ├─ fixup(): 动态检测问题, 追加各种 FixupPP 到 __postprocessors
  │
  └─ post_process(dl_filename, info_dict, files_to_move)  [YoutubeDL.py:3839]
      │
      ├─ 设置: info['filepath'] = filename
      │       info['__files_to_move'] = files_to_move [YoutubeDL.py:3842]
      │
      ├─ run_all_pps('post_process', info,
      │               additional_pps = info['__postprocessors'])
      │   │
      │   ├─ 动态追加的 PP (按添加顺序):
      │   │   ├─ FFmpegMergerPP        (如果需要合并)
      │   │   ├─ FFmpegFixupStretchedPP (如果需要)
      │   │   ├─ FFmpegFixupM3u8PP     (如果需要)
      │   │   └─ ...其他 Fixup PP
      │   │
      │   └─ 静态配置的 PP (self._pps['post_process']):
      │       ├─ FFmpegSubtitlesConvertorPP
      │       ├─ FFmpegExtractAudioPP     (-x 参数)
      │       ├─ FFmpegVideoRemuxerPP     (--remux-video)
      │       ├─ FFmpegVideoConvertorPP   (--recode-video)
      │       ├─ FFmpegEmbedSubtitlePP    (--embed-subs)
      │       ├─ ModifyChaptersPP         (章节/赞助商移除)
      │       ├─ FFmpegMetadataPP         (--add-metadata)
      │       ├─ EmbedThumbnailPP         (--embed-thumbnail)
      │       └─ FFmpegSplitChaptersPP    (--split-chapters)
      │
      │   每个 PP.run() 可能:
      │     - 更新 info['filepath'] / ['ext']
      │     - 返回 files_to_delete 列表
      │   run_pp() 处理返回 [YoutubeDL.py:3798]:
      │     if keepvideo → files_to_delete 加入 __files_to_move.setdefault
      │     else         → _delete_downloaded_files 删除 [YoutubeDL.py:3774]
      │
      ├─ run_pp(MoveFilesAfterDownloadPP(self))  [YoutubeDL.py:3843]
      │   │
      │   ├─ 主文件追加: __files_to_move[filepath] = finalpath
      │   ├─ 遍历映射表 [movefilesafterdownload.py:28-50]:
      │   │   ├─ 路径相同 → skip
      │   │   ├─ 源不存在 → warning + skip
      │   │   ├─ 目标已存在:
      │   │   │   ├─ overwrites∈{True,None} → os.remove(newfile) + move
      │   │   │   └─ overwrites=False → warning + skip (文件留temp)
      │   │   └─ 正常 → make_parent_dirs + shutil.move
      │   └─ 更新 info['filepath'] = finalpath (无条件!)
      │
      ├─ del info['__files_to_move']  [YoutubeDL.py:3845]
      │
      └─ run_all_pps('after_move', info)
          └─ XAttrMetadataPP (--xattrs, 写入扩展属性)
```

### 10.2 PP 执行顺序的重要性

PP 的顺序**不是任意的**，在 `get_postprocessors()` 中有严格的顺序设计：

1. **SubtitlesConvertor** (`before_dl`) → 先转字幕格式
2. **FFmpegExtractAudio** → 先提取音频（如果后续要转视频格式）
3. **FFmpegVideoRemuxer** / **FFmpegVideoConvertor** → 转换容器/编码
4. **FFmpegEmbedSubtitle** → 容器确定后嵌入字幕
5. **ModifyChapters** → 必须在 Metadata 之前（可能修改章节结构）
6. **FFmpegMetadata** → 容器不再变化，写入元数据
7. **EmbedThumbnail** → 嵌入缩略图
8. **FFmpegSplitChapters** → 切分章节（写入文件的最终步骤）
9. **XAttrMetadata** (`after_move`) → 文件落盘后，写入文件系统扩展属性

### 10.3 三种保留/覆盖参数的交互时序（典型案例）

```
用户命令: yt-dlp -k --no-overwrites -x --audio-format mp3 URL

下载前:
  existing_video_file() 检查 → default_overwrite=False
    overwrites=False (key 存在) → not False = True → 返回已有文件
    → 报告"已下载"，跳过整个流程

下载后 (假设是新文件):
  下载得到 temp/title.mp4

FFmpegExtractAudioPP.run():
  生成 temp/title.mp3
  返回 files_to_delete = ['temp/title.mp4']

run_pp() 处理删除列表 [YoutubeDL.py:3813-3815]:
  keepvideo=True → __files_to_move.setdefault('temp/title.mp4', '')
  (不删除原视频，加入移动映射)

MoveFilesAfterDownloadPP.run():
  __files_to_move['temp/title.mp3'] = 'downloads/title.mp3' (主文件追加)
  遍历:
    'temp/title.mp3':
      overwrites=False (key 存在且为 False)
        → 若 downloads/title.mp3 已存在 → skip (文件留 temp)
    'temp/title.mp4':
      同样 overwrites=False → 若目标已存在 → skip
  info['filepath'] = 'downloads/title.mp3' (无条件更新!)

最终结果:
  ✅ 音频生成成功，但可能卡在 temp 目录 (目标已存在时)
  ✅ 原视频保留（通过 keepvideo）
  ⚠️ 若目标文件已存在，info_dict 路径与实际文件位置不一致
```

---

## 十一、扩展开发：自定义混流/转码后处理器

如果需要自定义混流或转码逻辑，可以按以下步骤实现：

### 11.1 继承关系

```
PostProcessor (common.py)
    └── FFmpegPostProcessor (ffmpeg.py)
            └── YourCustomMuxerPP
```

### 11.2 最小实现模板（正确处理搬移与删除）

```python
from yt_dlp.postprocessor.ffmpeg import FFmpegPostProcessor
from yt_dlp.postprocessor.common import PostProcessor
from yt_dlp.utils import replace_extension

class MyCustomMuxerPP(FFmpegPostProcessor):
    @PostProcessor._restrict_to(images=False)  # 不处理纯图片
    def run(self, info):
        input_path = info['filepath']
        target_ext = 'mkv'  # 示例目标格式
        output_path = replace_extension(input_path, target_ext, info['ext'])

        # 1. 检查是否需要处理（已是目标格式则跳过）
        if info['ext'] == target_ext:
            return [], info  # 返回空列表 = 不处理，不删除

        # 2. 检查 nopostoverwrites (可选，避免重复处理)
        # if self._nopostoverwrites and os.path.exists(output_path):
        #     return [], info

        # 3. 调用 ffmpeg (run_ffmpeg 会自动调用 real_run_ffmpeg)
        opts = ['-c', 'copy', '-f', 'matroska']
        self.run_ffmpeg(input_path, output_path, opts)

        # 4. 更新 info_dict (产物替换 —— 必须更新，后续 PP 依赖此状态)
        info['filepath'] = output_path
        info['ext'] = target_ext

        # 5. 返回 (待删除文件列表, 更新后的 info)
        # 注意: run_pp() 会根据 keepvideo 参数决定是否实际删除
        # PP 本身无需处理 keepvideo 逻辑
        return [input_path], info
```

### 11.3 通过 API 注册

```python
from yt_dlp import YoutubeDL

ydl_opts = {
    'postprocessors': [{
        'key': 'MyCustomMuxer',         # 通过插件注册后使用名称
        'when': 'post_process',         # 可选时机
        # 'custom_param1': 'value',     # 构造函数参数
    }],
    'keepvideo': True,                  # 保留原文件
    'overwrites': False,                # 不覆盖已存在文件
}

# 或手动注册（程序化使用）:
ydl = YoutubeDL(ydl_opts)
ydl.add_post_processor(MyCustomMuxerPP(ydl), when='post_process')
```

---

## 关键文件速查表（带精确行号锚点）

> **核准说明**：所有条目均按「功能名称 = 锚点实际代码 = 说明描述」三方对齐原则逐一核实。
> 单行锚点标注为 `Lxxxx`，多行范围标注为 `Lxxxx-Lyyyy`；说明列描述该行/范围真实功能，避免重复行号。

### PP 基础与调度层

| 功能 | 仓库内路径 | 精确行 | 锚点真实代码 / 说明 |
|------|---------|--------|-------------------|
| get_postprocessor 按名称查 PP 类 | [__init__.py:L51-L52](yt_dlp/postprocessor/__init__.py#L51-L52) | L51-L52 | `return postprocessors.value[key + 'PP']`，将字符串 key 映射为 PP 类 |
| _default_pps 默认 PP 注册表构建 | [__init__.py:L62-L68](yt_dlp/postprocessor/__init__.py#L62-L68) | L62-L68 | 收集模块内所有 `*PP` 类注入全局注册表 |
| PP 基类 + 元类 run_wrapper 包装 | [common.py:L16-L135](yt_dlp/postprocessor/common.py#L16-L135) | L16-L135 | PostProcessorMetaClass(L16-33) + PostProcessor 基类(L36-135) |
| PP.get_param() 参数读取封装 | [common.py:L100-L103](yt_dlp/postprocessor/common.py#L100-L103) | L100-L103 | `self._downloader.params.get(name, default)` —— 所有 PP 读取配置的统一入口 |
| PP._restrict_to 装饰器 | [common.py:L115-L115](yt_dlp/postprocessor/common.py#L115-L115) | L115 | 控制 PP 在视频/音频/图片场景下是否跳过 |
| PP.run() 抽象接口 | [common.py:L135-L135](yt_dlp/postprocessor/common.py#L135-L135) | L135 | 所有 PP 必须实现的抽象方法 |
| YoutubeDL.add_post_processor | [YoutubeDL.py:L942-L946](yt_dlp/YoutubeDL.py#L942-L946) | L942-L946 | `self._pps[when].append(pp)` —— PP 真正加入调度链的唯一入口 |
| YoutubeDL PP 实例化循环 | [YoutubeDL.py:L827-L834](yt_dlp/YoutubeDL.py#L827-L834) | L827-L834 | 遍历 params['postprocessors']，按 key 查找类、实例化后调用 add_post_processor |
| run_pp 单个 PP 执行（删除/保留决策） | [YoutubeDL.py:L3798-L3819](yt_dlp/YoutubeDL.py#L3798-L3819) | L3798-L3819 | 调用 pp.run()，处理 files_to_delete：keepvideo 则转移动表，否则物理删除 |
| run_all_pps 批量调度 | [YoutubeDL.py:L3821-L3826](yt_dlp/YoutubeDL.py#L3821-L3826) | L3821-L3826 | 合并执行 additional_pps (动态注入) + self._pps[key] (静态配置)，info 链式传递 |
| pre_process 阶段处理 | [YoutubeDL.py:L3828-L3837](yt_dlp/YoutubeDL.py#L3828-L3837) | L3828-L3837 | before_dl / pre_process 时机入口，弹出并返回 files_to_move |
| post_process 后处理总入口 | [YoutubeDL.py:L3839-L3846](yt_dlp/YoutubeDL.py#L3839-L3846) | L3839-L3846 | 依次执行 post_process 阶段 PP → MoveFilesAfterDownloadPP → after_move 阶段 PP |

### overwrites 三态覆盖控制

| 功能 | 仓库内路径 | 精确行 | 锚点真实代码 / 说明 |
|------|---------|--------|-------------------|
| overwrites 参数三态注释 | [YoutubeDL.py:L291-L293](yt_dlp/YoutubeDL.py#L291-L293) | L291-L293 | 文档定义：True=全部覆盖 / None=仅非视频覆盖 / False=全不覆盖 |
| overwrites=None 时移除 key | [YoutubeDL.py:L778-L779](yt_dlp/YoutubeDL.py#L778-L779) | L778-L779 | `params.pop('overwrites', None)` —— 差异化覆盖的核心机制（让 default_overwrite 生效） |
| existing_file() 覆盖判定核心 | [YoutubeDL.py:L3320-L3328](yt_dlp/YoutubeDL.py#L3320-L3328) | L3320-L3328 | `not params.get('overwrites', default_overwrite)` —— 为 True 则返回已有文件=跳过下载 |
| existing_video_file() 视频专用入口 | [YoutubeDL.py:L3463-L3470](yt_dlp/YoutubeDL.py#L3463-L3470) | L3463-L3470 | 调用 existing_file 时传 **default_overwrite=False**，实现 None 时视频不覆盖 |
| info.json 覆盖参数读取 | [YoutubeDL.py:L4396-L4396](yt_dlp/YoutubeDL.py#L4396-L4396) | L4396 | `overwrite = params.get('overwrites', True)` —— None→True→覆盖 |
| description 覆盖条件检查 | [YoutubeDL.py:L4425-L4425](yt_dlp/YoutubeDL.py#L4425-L4425) | L4425 | `not params.get('overwrites', True) and os.path.exists(descfn)` —— None→True→覆盖 |
| 字幕 existing_file 检查 | [YoutubeDL.py:L4460-L4460](yt_dlp/YoutubeDL.py#L4460-L4460) | L4460 | `existing_file(...)` default_overwrite 默认为 True —— None→覆盖 |
| 缩略图 existing_file 检查 | [YoutubeDL.py:L4524-L4524](yt_dlp/YoutubeDL.py#L4524-L4524) | L4524 | 同上，default_overwrite=True —— None→覆盖 |
| Internet link 文件特殊分支 | [YoutubeDL.py:L3418-L3420](yt_dlp/YoutubeDL.py#L3418-L3420) | L3418-L3420 | overwrites=True 且 link 文件已存在 → 视为成功直接 return，**不重写**（与普通非视频行为不同） |
| 搬移覆盖判断核心 | [movefilesafterdownload.py:L37-L38](yt_dlp/postprocessor/movefilesafterdownload.py#L37-L38) | L37-L38 | `get_param('overwrites', True)` —— None→True→先 os.remove 再移动 |
| report_file_already_downloaded | [YoutubeDL.py:L1172-L1177](yt_dlp/YoutubeDL.py#L1172-L1177) | L1172-L1177 | 「文件已下载」提示输出 |
| report_file_delete | [YoutubeDL.py:L1179-L1184](yt_dlp/YoutubeDL.py#L1179-L1184) | L1179-L1184 | 「删除已有文件」提示输出 |

### 主流程与产物搬移

| 功能 | 仓库内路径 | 精确行 | 锚点真实代码 / 说明 |
|------|---------|--------|-------------------|
| prepare_filename 文件名生成 | [YoutubeDL.py:L1551-L1570](yt_dlp/YoutubeDL.py#L1551-L1570) | L1551-L1570 | 根据 dir_type / outtmpl 计算最终路径 |
| _ensure_dir_exists 目录创建检查 | [YoutubeDL.py:L2038-L2044](yt_dlp/YoutubeDL.py#L2038-L2044) | L2038-L2044 | `make_parent_dirs(path)` 并捕获异常返回 True/False |
| _merge 格式合并决策 | [YoutubeDL.py:L2450-L2522](yt_dlp/YoutubeDL.py#L2450-L2522) | L2450-L2522 | 多格式组合 + get_compatible_ext 计算输出容器 |
| process_info 主流程入口 | [YoutubeDL.py:L3331-L3668](yt_dlp/YoutubeDL.py#L3331-L3668) | L3331-L3668 | 单个视频完整处理链：准备→下载→fixup→post_process |
| files_to_move 初始化 | [YoutubeDL.py:L3361-L3361](yt_dlp/YoutubeDL.py#L3361-L3361) | L3361 | `files_to_move = {}` 空字典创建 |
| 字幕文件加入移动表 | [YoutubeDL.py:L3389-L3389](yt_dlp/YoutubeDL.py#L3389-L3389) | L3389 | `files_to_move.update(dict(sub_files))` |
| 缩略图加入移动表 | [YoutubeDL.py:L3395-L3395](yt_dlp/YoutubeDL.py#L3395-L3395) | L3395 | `files_to_move.update(dict(thumb_files))` |
| 动态注册 MergerPP | [YoutubeDL.py:L3565-L3567](yt_dlp/YoutubeDL.py#L3565-L3567) | L3565-L3567 | `info_dict['__postprocessors'].append(merger)` —— 多格式下载后动态注入合并 PP |
| 合并取消时保留中间文件 | [YoutubeDL.py:L3570-L3572](yt_dlp/YoutubeDL.py#L3570-L3572) | L3570-L3572 | merger 不可用时，所有 downloaded 文件加入 files_to_move 保留输出 |
| _delete_downloaded_files 删除实现 | [YoutubeDL.py:L3774-L3783](yt_dlp/YoutubeDL.py#L3774-L3783) | L3774-L3783 | 物理删除文件，并同步从 __files_to_move 中移除 |
| __files_to_move 在 post_process 入口设置 | [YoutubeDL.py:L3842-L3842](yt_dlp/YoutubeDL.py#L3842-L3842) | L3842 | `info['__files_to_move'] = files_to_move` |
| __files_to_move 在 post_process 结束清理 | [YoutubeDL.py:L3845-L3845](yt_dlp/YoutubeDL.py#L3845-L3845) | L3845 | `del info['__files_to_move']` |
| MoveFilesAfterDownloadPP 文件搬移 PP | [movefilesafterdownload.py:L11-L53](yt_dlp/postprocessor/movefilesafterdownload.py#L11-L53) | L11-L53 | 完整搬移类：主文件追加 + 遍历移动表 + 覆盖判断 + shutil.move |
| 搬移跨卷 shutil.move | [movefilesafterdownload.py:L50-L50](yt_dlp/postprocessor/movefilesafterdownload.py#L50-L50) | L50 | `shutil.move(oldfile, newfile)` —— 自动 fallback copy+delete 支持跨卷 |
| 搬移无条件更新 filepath（⚠️风险点） | [movefilesafterdownload.py:L52-L52](yt_dlp/postprocessor/movefilesafterdownload.py#L52-L52) | L52 | `info['filepath'] = finalpath` —— 无论移动是否成功都会更新，可能导致路径与实际文件位置不一致 |