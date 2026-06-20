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
```

**关键代码入口文件：**
- 主流程调度：[YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py)
- 后处理器基类：[postprocessor/common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/common.py)
- FFmpeg 系列后处理器：[postprocessor/ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py)
- 后处理器注册：[postprocessor/__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/__init__.py)
- CLI 参数转后处理器配置：[__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/__init__.py#L627-L729)
- FFmpeg 直接下载合并：[downloader/external.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/downloader/external.py#L372-L571)

---

## 二、阶段一：CLI 参数 → 后处理器配置

### 2.1 后处理器执行时机 (POSTPROCESS_WHEN)

yt-dlp 定义了 **8 个后处理时机**，决定每个 PP 在流水线哪个阶段执行：

```python
# yt_dlp/utils/_utils.py:2858
POSTPROCESS_WHEN = ('pre_process', 'after_filter', 'video', 'before_dl', 
                     'post_process', 'after_move', 'after_video', 'playlist')
```

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

在 [yt_dlp/__init__.py:627-729](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/__init__.py#L627-L729) 的 `get_postprocessors()` 函数中，命令行参数被转化为后处理器配置字典列表。

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

在 [YoutubeDL.py:827-834](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L827-L834) 中，配置字典被实例化为后处理器对象：

```python
for pp_def_raw in self.params.get('postprocessors', []):
    pp_def = dict(pp_def_raw)
    when = pp_def.pop('when', 'post_process')  # 默认时机: post_process
    self.add_post_processor(
        get_postprocessor(pp_def.pop('key'))(self, **pp_def),  # 通过名称查找类并实例化
        when=when)
```

- `get_postprocessor(key)` → [postprocessor/__init__.py:51-52](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/__init__.py#L51-L52)：从全局注册表查找 `key + 'PP'` 对应的类
- `add_post_processor(pp, when)` → [YoutubeDL.py:942-946](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L942-L946)：将 PP 加入 `self._pps[when]` 列表

---

## 三、阶段二：格式选择与合并决策

### 3.1 何时触发格式合并

当用户的格式选择器（`-f` 参数）选择了**多个需要组合的格式**时（例如 `bestvideo+bestaudio`），格式选择系统会生成一个包含 `requested_formats` 字段的 info_dict。

**关键代码：** [YoutubeDL.py:2450-2522](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L2450-L2522) 的 `_merge()` 函数

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

`get_compatible_ext()` → [_utils.py:3088-3126](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/utils/_utils.py#L3088-L3126)

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

**路径判断代码：** [YoutubeDL.py:3482-3569](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L3482-L3569)

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

**FFmpegFD 合并实现：** [downloader/external.py:395-571](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/downloader/external.py#L395-L571)

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

1. **为每个格式分配独立文件名** [YoutubeDL.py:3519-3524](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L3519-L3524)：
```python
for f in info_dict['requested_formats']:
    f['filepath'] = fname = prepend_extension(
        correct_ext(temp_filename, info_dict['ext']),
        f'f{f["format_id"]}', info_dict['ext'])
    # e.g., 视频 title.f137.mp4, 音频 title.f140.m4a
    downloaded.append(fname)
```

2. **逐个格式下载**：循环调用 `self.dl(fname, new_info)`

3. **注册合并后处理器** [YoutubeDL.py:3565-3567](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L3565-L3567)：
```python
if downloaded and merger.available and not self.params.get('allow_unplayable_formats'):
    info_dict['__postprocessors'].append(merger)       # 加入额外 PP 列表
    info_dict['__files_to_merge'] = downloaded          # 待合并文件列表
```

> **关键点**：`__postprocessors` 是**动态注入**的后处理器列表，与 `self._pps['post_process']` 中的静态 PP 合并执行。

---

## 五、阶段四：后处理器调度执行

### 5.1 入口：post_process() 方法

[YoutubeDL.py:3839-3846](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L3839-L3846)

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

[YoutubeDL.py:3821-3826](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L3821-L3826)

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

[YoutubeDL.py:3798-3819](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py#L3798-L3819)

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

[common.py:16-33](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/common.py#L16-L33)

所有 PP 的 `run()` 方法被 `PostProcessorMetaClass.run_wrapper` 自动包装：
- 执行前 → 触发 `_hook_progress({'status': 'started'})`
- 执行后 → 触发 `_hook_progress({'status': 'finished'})`

---

## 六、阶段五：FFmpeg 外部工具调用机制

### 6.1 FFmpegPostProcessor 初始化

[ffmpeg.py:86-198](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L86-L198)

```python
class FFmpegPostProcessor(PostProcessor):
    def __init__(self, downloader=None):
        PostProcessor.__init__(self, downloader)
        self._paths = self._determine_executables()  # 确定 ffmpeg/ffprobe 路径
```

**可执行文件定位逻辑** `_determine_executables()` [ffmpeg.py:102-128](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L102-L128)：
1. 优先使用 `--ffmpeg-location` 参数
2. 如果是目录 → 拼接 `ffmpeg.exe` / `ffprobe.exe`
3. 如果是文件（如指定了 `custom-ffmpeg.exe`）→ 将 basename 替换为对应程序名
4. 默认可直接调用系统 PATH 中的 `ffmpeg`/`ffprobe`

### 6.2 核心调用方法：real_run_ffmpeg()

[ffmpeg.py:326-364](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L326-L364)

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

**用户自定义参数注入** (`_configuration_args`)：
通过 `--postprocessor-args`（或 `--ppa`）可以注入自定义参数。
- 输入参数配置键：`_i1`, `_i2`, ..., `_i`（通用）
- 输出参数配置键：`_o1`, `_o`, `''`（第一个输出的默认键）

### 6.3 通用流复制参数：stream_copy_opts()

[ffmpeg.py:212-221](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L212-L221)

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

### 6.4 便捷调用封装

| 方法 | 用途 | 调用链 |
|------|------|--------|
| `run_ffmpeg(path, out_path, opts)` | 单文件处理 | → `run_ffmpeg_multiple_files([path], out_path, opts)` |
| `run_ffmpeg_multiple_files(input_paths, out_path, opts)` | 多文件输入单输出 | → `real_run_ffmpeg([(p,[]) for p in input_paths], [(out_path, opts)])` |
| `real_run_ffmpeg(input_path_opts, output_path_opts)` | 完整灵活调用 | 最终执行 |

---

## 七、核心后处理器详解

### 7.1 FFmpegMergerPP：格式合并（混流）

**代码位置：** [ffmpeg.py:822-847](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L822-L847)

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

**代码位置：** [ffmpeg.py:573-578](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L573-L578)

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

**代码位置：** [ffmpeg.py:538-570](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L538-L570)

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

**代码位置：** [ffmpeg.py:432-535](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L432-L535)

**CLI 参数：** `-x/--extract-audio`, `--audio-format`, `--audio-quality`

**处理策略决策：**

```
源文件音频编码 == 目标编码?
    ├─ Yes → -c copy (无损提取，仅剥离视频流)
    └─ No  → 指定编码器重新编码 (libmp3lame, aac, libopus 等)
```

**产物替换（特殊逻辑）：**
当输出扩展名与原扩展名相同时（如 m4a → m4a 但需要修复封装）：
```python
if new_path == path:
    orig_path = prepend_extension(path, 'orig')  # title.orig.m4a
    temp_path = prepend_extension(path, 'temp')  # title.temp.m4a
# 处理完成后:
os.replace(path, orig_path)        # 原文件 → .orig
os.replace(temp_path, new_path)    # 临时 → 最终路径
information['filepath'] = new_path # 更新 info_dict
information['ext'] = extension
return [orig_path], information    # .orig 加入删除列表
```

### 7.5 Fixup 系列后处理器：自动修复常见问题

**代码位置：** [ffmpeg.py:850-937](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L850-L937)

这些 PP 由 `process_info()` 中的 `fixup()` 函数**动态注册**到 `info_dict['__postprocessors']`：

| Fixup PP | 触发条件 | 修复操作 |
|----------|---------|----------|
| `FFmpegFixupStretchedPP` | `stretched_ratio != 1` | 加 `-aspect` 参数修复像素比 |
| `FFmpegFixupM4aPP` | DASH 下载的 `m4a_dash` 容器 | `-f mp4` 重新封装 |
| `FFmpegFixupM3u8PP` | HLS native 下载的 mp4/m4a 且检测为 MPEG-TS 封装 | 重新封装为正确 MP4 + `aac_adtstoasc` |
| `FFmpegFixupTimestampPP` | WebSocket 分片下载的直播 | `-bsf setts` 或 `setpts` 滤镜 |
| `FFmpegFixupDurationPP` | WebSocket 分片下载的直播 | 流复制重写 |
| `FFmpegFixupDuplicateMoovPP` | DASH 多周期直播 | 流复制去除重复 MOOV |

---

## 八、产物替换与信息传递机制

### 8.1 info_dict 的链式传递

后处理器之间通过**修改同一个 info_dict 对象**实现状态传递：

```
初始 info_dict:
  {filepath: 'title.f137.mp4', ext: 'mp4', requested_formats: [...], ...}
      │
      ▼ FFmpegMergerPP.run()
  {filepath: 'title.mp4', ext: 'mp4', __files_to_merge 被删除, ...}
      │
      ▼ FFmpegVideoConvertorPP.run()
  {filepath: 'title.mp4', ext: 'mp4', ...} → 不变 (已是mp4)
      │
      ▼ FFmpegEmbedSubtitlePP.run()
  {filepath: 'title.mp4', ...} → 内部替换 temp，内容变了路径不变
      │
      ▼ FFmpegExtractAudioPP.run()
  {filepath: 'title.mp3', ext: 'mp3', ...} → 路径和扩展名都更新
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

**是否实际删除由 `run_pp()` 根据 `-k/--keepvideo` 参数决定：**
```python
if self.params.get('keepvideo', False):
    # -k 参数：将待删除文件加入 __files_to_move (最终会保留)
    for f in files_to_delete:
        infodict['__files_to_move'].setdefault(f, '')
else:
    # 默认行为: 物理删除中间文件
    self._delete_downloaded_files(*files_to_delete, ...)
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

## 九、完整调用链路图

### 9.1 单个视频处理的完整链路

```
YoutubeDL.process_info(info_dict)  [L3331]
  │
  ├─ 格式已选择完成, info_dict['requested_formats'] 可能存在
  │
  ├─ 下载分支判断:
  │   ├─ FFmpegFD 可直接合并? ─Yes─► 一次调用 ffmpeg 完成下载+合并
  │   │                          (不触发 FFmpegMergerPP)
  │   └─ No:
  │       ├─ 有 requested_formats?
  │       │   └─ Yes: 分别下载各格式 → 记录 downloaded[]
  │       │           → info_dict['__postprocessors'].append(FFmpegMergerPP)
  │       │           → info_dict['__files_to_merge'] = downloaded
  │       └─ No: 直接下载单个文件
  │
  ├─ fixup(): 动态检测问题, 追加各种 FixupPP 到 __postprocessors
  │
  └─ post_process(dl_filename, info_dict)  [L3839]
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
      │   (每个 PP.run() 都可能更新 filepath / ext / 返回待删除文件)
      │
      ├─ run_pp(MoveFilesAfterDownloadPP)
      │   └─ 把 temp 目录的文件移动到最终目录, 批量 os.replace
      │
      └─ run_all_pps('after_move', info)
          └─ XAttrMetadataPP (--xattrs, 写入扩展属性)
```

### 9.2 PP 执行顺序的重要性

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

---

## 十、扩展开发：自定义混流/转码后处理器

如果需要自定义混流或转码逻辑，可以按以下步骤实现：

### 10.1 继承关系

```
PostProcessor (common.py)
    └── FFmpegPostProcessor (ffmpeg.py)
            └── YourCustomMuxerPP
```

### 10.2 最小实现模板

```python
from yt_dlp.postprocessor.ffmpeg import FFmpegPostProcessor, PostProcessingError
from yt_dlp.postprocessor.common import PostProcessor

class MyCustomMuxerPP(FFmpegPostProcessor):
    @PostProcessor._restrict_to(images=False)  # 不处理纯图片
    def run(self, info):
        input_path = info['filepath']
        output_path = ...  # 计算输出路径
        
        # 1. 检查是否需要处理
        if not self._should_process(info):
            return [], info  # 返回空列表 = 不处理，不删除
        
        # 2. 调用 ffmpeg
        opts = ['-c', 'copy', ...]  # 你的自定义参数
        self.run_ffmpeg(input_path, output_path, opts)
        
        # 3. 更新 info_dict (产物替换)
        info['filepath'] = output_path
        info['ext'] = determine_ext(output_path)
        
        # 4. 返回 (待删除文件列表, 更新后的 info)
        return [input_path], info
```

### 10.3 通过 API 注册

```python
from yt_dlp import YoutubeDL

ydl_opts = {
    'postprocessors': [{
        'key': 'MyCustomMuxer',        # 或通过插件注册后使用名称
        'when': 'post_process',        # 可选时机
        # 'custom_param1': 'value',    # 构造函数参数
    }],
}

# 或手动注册:
ydl = YoutubeDL(ydl_opts)
ydl.add_post_processor(MyCustomMuxerPP(ydl), when='post_process')
```

---

## 关键文件速查表

| 功能 | 文件路径 | 关键行范围 |
|------|---------|-----------|
| 后处理器注册机制 | [postprocessor/__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/__init__.py) | L51-L68 |
| PP 基类 + 元类包装 | [postprocessor/common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/common.py) | L16-L150 |
| FFmpeg 工具调用封装 | [postprocessor/ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py) | L326-L364 |
| 格式合并 (FFmpegMergerPP) | [postprocessor/ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py) | L822-L847 |
| 视频转码 (VideoConvertor) | [postprocessor/ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py) | L538-L578 |
| 音频提取 (ExtractAudio) | [postprocessor/ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/postprocessor/ffmpeg.py) | L432-L535 |
| 主流程 process_info | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py) | L3331-L3668 |
| 格式合并决策 _merge | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py) | L2450-L2522 |
| 后处理调度 post_process | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py) | L3839-L3846 |
| run_pp / run_all_pps | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/YoutubeDL.py) | L3798-L3826 |
| CLI 转 PP 配置 | [__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/__init__.py) | L627-L729 |
| FFmpegFD 直接合并下载 | [downloader/external.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/downloader/external.py) | L372-L571 |
| 容器兼容判定 get_compatible_ext | [utils/_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/utils/_utils.py) | L3088-L3126 |
| 后处理时机定义 | [utils/_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/94-yt-dlp/yt_dlp/utils/_utils.py) | L2858 |
