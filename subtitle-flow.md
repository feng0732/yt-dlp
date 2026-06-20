# yt-dlp 字幕提取与转码链路详解

本文档追踪 yt-dlp 中字幕的完整生命周期：从发现、格式选择、下载写入到格式转换和嵌入。

---

## 1. 字幕发现（Subtitle Discovery）

字幕数据在 **Extractor（提取器）** 层产出，最终汇入 `info_dict` 的两个字段：

| 字段 | 含义 |
|---|---|
| `subtitles` | 人工字幕（手动上传） |
| `automatic_captions` | 自动生成字幕（ASR） |

### 1.1 数据结构

字幕的统一数据结构定义在 [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L331-L339)：

```python
subtitles = {
    lang_code: [       # key 是语言代码（如 'en', 'zh-Hans'）
        {
            'ext': 'srt',          # 必填：字幕格式扩展名
            'url': 'https://...',  # 与 data 二选一：字幕文件下载地址
            'data': '...'         # 与 url 二选一：字幕文件内联内容
            'name': 'English',    # 可选：字幕描述/名称
        },
        ...  # 同一语言可有多种格式，按偏好从低到高排列
    ]
}
```

> **关键约定**：同一 `lang_code` 下的列表是**有序**的，越靠后偏好越高。`process_subtitles` 选格式时取 `formats[-1]` 即"最佳"。

### 1.2 Extractor 如何产出字幕

#### 1.2.1 各站点独立实现 `_get_subtitles` / `_get_automatic_captions`

基类 [InfoExtractor](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3885-L3892) 定义了入口：

```python
def extract_subtitles(self, *args, **kwargs):
    if (self.get_param('writesubtitles', False)
            or self.get_param('listsubtitles')):
        return self._get_subtitles(*args, **kwargs)
    return {}

def _get_subtitles(self, *args, **kwargs):
    raise NotImplementedError('This method must be implemented by subclasses')
```

- 仅在 `--write-subs` 或 `--list-subs` 被指定时才真正调用 `_get_subtitles`，否则直接返回空字典，**避免不必要的网络请求**。
- 各站点 Extractor（YouTube、Bilibili、BBC 等）各自重写 `_get_subtitles`，从站点 API 获取字幕列表。

#### 1.2.2 从流式清单中解析（M3U8 / DASH MPD / SMIL / ISM）

除了站点 API，字幕也可内嵌在流式清单中。InfoExtractor 提供了配套的解析方法：

| 方法 | 来源 |
|---|---|
| [_extract_m3u8_formats_and_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2177-L2222) | HLS M3U8 master playlist |
| [_extract_mpd_formats_and_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2806-L2835) | DASH MPD manifest |
| [_extract_smil_formats_and_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2521-L2531) | SMIL 文件 |
| [_extract_ism_formats_and_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3218-L3233) | IIS Smooth Streaming |
| [_extract_akamai_formats_and_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3486-L3542) | Akamai HD manifest |

**M3U8 字幕发现流程**（最常见）：

1. 请求 M3U8 URL，获取 playlist 文本。
2. 在 [_parse_m3u8_formats_and_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2224-L2311) 中解析 `#EXT-X-MEDIA:TYPE=SUBTITLES` 标签。
3. 提取 `LANGUAGE`、`URI`、`NAME` 等属性。
4. 若 URI 指向 `.m3u8`，按 RFC 8216 §3.1 推断 ext 为 `vtt`，并标记 `protocol: 'm3u8_native'`。
5. 按 `LANGUAGE` 键值存入 `subtitles` 字典。

**DASH MPD 字幕发现**：在 [_parse_mpd_formats_and_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2840-L2870) 中，`content_type == 'text'` 的 Representation 被归入 `period['subtitles']`，最终按语言合并。

#### 1.2.3 不关心字幕的便捷方法

若 Extractor 只需要格式不需要字幕，可使用无后缀的便捷方法（如 `_extract_m3u8_formats`），内部仍调用带字幕版，但丢弃字幕并打印警告 [_report_ignoring_subs](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2165-L2169)。

### 1.3 字幕合并 `_merge_subtitles`

[_merge_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3938-L3946) 将多个来源的字幕字典合并：

```python
@classmethod
def _merge_subtitles(cls, *dicts, target=None):
    if target is None:
        target = {}
    for d in filter(None, dicts):
        for lang, subs in d.items():
            target[lang] = cls._merge_subtitle_items(target.get(lang, []), subs)
    return target
```

底层 [_merge_subtitle_items](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3929-L3936) 按 `(url, data)` 去重合并同一语言下的条目。典型场景：YouTube 同时从 HLS 和 DASH 清单中发现字幕，用 `_merge_subtitles` 合并。

### 1.4 YouTube 字幕发现实例

YouTube 的字幕发现最为复杂，位于 [youtube/_video.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/youtube/_video.py#L4195-L4336)：

1. 从多个 player response 中遍历 `captionTracks`。
2. `kind != 'asr'` → 人工字幕，存入 `subtitles`；`kind == 'asr'` → 自动字幕，存入 `automatic_captions`。
3. 对每个 caption track，按 [_SUBTITLE_FORMATS](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/youtube/_video.py#L142) = `('json3', 'srv1', 'srv2', 'srv3', 'ttml', 'srt', 'vtt')` 生成多种格式的 URL（通过 `fmt` 查询参数）。
4. 同时处理翻译字幕（`translationLanguages`），生成带语言后缀的条目如 `en-ja`。
5. PO Token 机制：部分字幕需要 PO Token 才能访问，缺少时跳过并报告。

---

## 2. 字幕处理与格式选择（Subtitle Processing）

Extractor 产出原始字幕数据后，控制权交还给 [YoutubeDL.process_info](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L2896-L2915)。

### 2.1 URL 清理与 ext 推断

[YoutubeDL.py L2901-L2909](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L2901-L2909)：

```python
for cc_kind in ('subtitles', 'automatic_captions'):
    cc = info_dict.get(cc_kind)
    if cc:
        for _, subtitle in cc.items():
            for subtitle_format in subtitle:
                if subtitle_format.get('url'):
                    subtitle_format['url'] = sanitize_url(subtitle_format['url'])
                if subtitle_format.get('ext') is None:
                    subtitle_format['ext'] = determine_ext(subtitle_format['url']).lower()
```

- 对所有字幕 URL 做 `sanitize_url`（规范化）。
- 若 Extractor 未设置 `ext`，从 URL 推断文件扩展名。

#### 2.1.1 determine_ext 扩展名推断算法详解

并非所有 Extractor 都会显式设置字幕条目的 `ext` 字段。当 `ext` 缺失时，yt-dlp 会在**多个层级**通过 [determine_ext](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L1310-L1320) 兜底推断：

**determine_ext 的算法**：

```python
def determine_ext(url, default_ext='unknown_video'):
    if url is None or '.' not in url:
        return default_ext
    guess = url.partition('?')[0].rpartition('.')[2]   # 去 query，取最后一个 '.' 后内容
    if re.match(r'^[A-Za-z0-9]+$', guess):            # 纯字母数字
        return guess
    elif guess.rstrip('/') in KNOWN_EXTENSIONS:       # 末尾有 / 但去掉后是已知扩展名
        return guess.rstrip('/')
    else:
        return default_ext
```

三层判断：
1. **基础匹配**：去掉查询串，取 URL 最后一个点号后的字符串，如果纯由字母数字组成，就是扩展名。例：`https://x.com/sub.srt?t=1` → `srt`。
2. **尾部斜杠处理**：某些 URL 形如 `https://x.com/sub.vtt/?download`，去掉末尾 `/` 后若在 `KNOWN_EXTENSIONS` 中（`m3u8, mpd, srt, vtt, ...`）则命中。
3. **兜底返回**：都不匹配则返回 `default_ext`。字幕场景下的 `default_ext` 为 `'unknown_video'`（由调用点传入），此时字幕格式会是错误的，后续 `process_subtitles` 的格式匹配会失败并退化回取最后一条（"best"策略）。

**各层级的 ext 推断调用点（仅与字幕相关）**：

| 调用位置 | 所属解析入口 | 场景 | default_ext |
|---|---|---|---|
| [common.py L2302](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2302) | `_parse_m3u8_formats_and_subtitles` | HLS M3U8 中 `#EXT-X-MEDIA:TYPE=SUBTITLES` 标签的 URI | —（无默认值，即 `'unknown_video'`）|
| [common.py L2738](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2738) | `_parse_smil_subtitles` | SMIL 中 `<textstream>` 标签的 `src` 属性 | —（无默认值），前面已有 `ext`/`mimetype2ext` 两道兜底 |
| [YoutubeDL.py L2908](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L2908) | 全局最后防线（`process_info`）| 所有尚未设置 `ext` 的字幕条目（如 HTML5 `<track kind="subtitles">` 标签的字幕） | —（无默认值）|

**非字幕解析的 determine_ext 调用点（此前误归入，特此澄清）**：

| 行号 | 所属函数 | 场景 | 类型 |
|---|---|---|---|
| [common.py L2113](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2113) | `_extract_f4m_formats` | F4M 清单 URL 解析 | 音视频清单 |
| [common.py L2635](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2635) | `_parse_smil_formats_and_subtitles` | SMIL `<video>/<audio>/<media>` 标签 `src` | 音视频媒体 |
| [common.py L3370](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3370) | `_parse_html5_media_entries` | HTML5 `<video>/<audio>` 标签 `src` | 音视频媒体 |
| [common.py L3691](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3691) | `_parse_jwplayer_formats` | JWPlayer sources 数据解析 | 音视频媒体 |

**其他流格式字幕的 ext 设置方式（不使用 determine_ext）**：

| 格式 | ext 设置方式 | 代码位置 |
|---|---|---|
| DASH MPD | `mimetype2ext(mime_type)` 从 MIME 类型映射 | [common.py L2989](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2989) |
| ISM | 硬编码 `'ismt'` | [common.py L3304](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3304) |
| HTML5 `<track>` | 未设置 ext，依赖 YoutubeDL L2908 全局推断 | [common.py L3469-L3471](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3469-L3471) |

**特例 1：M3U8 指向 m3u8 时**：在 L2302 推断出 `ext='m3u8'` 后，紧接着 L2304-L2309 会**硬编码覆盖**为 `ext='vtt'` 并标记 `protocol='m3u8_native'`（依据 RFC 8216 §3.1：m3u8 字幕清单只能包含 WebVTT 内容）。这确保后续下载器知道需要按 m3u8 分片方式下载 VTT 片段。

**特例 2：YouTube 字幕**：URL 无路径扩展名（格式靠查询串 `fmt=srt` 控制），因此 YouTube Extractor 必须**显式设置**每个字幕条目的 `ext`（在 [youtube/_video.py L4201-L4211](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/youtube/_video.py#L4201-L4211) 中循环 `_SUBTITLE_FORMATS` 生成条目时直接写入 `'ext': fmt`），否则 `determine_ext` 会推断失败并产出无法使用的 `unknown_video` 扩展名。

**推断失败的影响**：若某字幕 URL 推断出异常扩展名，该条目的 ext 字段就是异常值，`process_subtitles` 的 `--sub-format srt/vtt/...` 选择器将匹配不到任何条目，最终取 `formats[-1]` 并打印警告：`No subtitle format found matching "srt" for language zh, using unknown_video.`，随后 `_write_subtitles` 写出的文件将以 `*.unknown_video` 为后缀，后续转换和嵌入也会受影响。

### 2.2 process_subtitles：语言与格式筛选

[process_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3154-L3212) 完成两件事：

**① 语言筛选**

| 参数 | 行为 |
|---|---|
| `writesubtitles` | 将 `subtitles`（人工字幕）加入候选 |
| `writeautomaticsub` | 将 `automatic_captions` 中**尚未在人工字幕中出现的语言**加入候选 |
| `allsubtitles` | 请求所有候选语言 |
| `subtitleslangs` | 按正则/精确匹配筛选语言，支持特殊值 `'all'` |
| 默认（均未指定） | 取第一个 `en` 开头的语言，若无则取第一个可用语言 |

**② 格式筛选**

`subtitlesformat` 参数（`--sub-format`，默认 `'best'`）控制格式偏好：

- `'best'` → 取列表最后一个（偏好最高的）。
- `'srt/vtt/best'` → 依次尝试找 ext=srt、ext=vtt，都没找到则取 best。
- 找不到匹配格式时取 best 并打印警告。

最终产出 `requested_subtitles` 字典：

```python
requested_subtitles = {
    'en': {'ext': 'vtt', 'url': '...', 'name': 'English'},
    'zh-Hans': {'ext': 'srt', 'url': '...', 'name': 'Chinese (Simplified)'},
}
```

注意：与原始 `subtitles` 不同，`requested_subtitles` **每个语言只保留一条**（选中格式的那个）。

### 2.3 listsubtitles：列出可用字幕

当 `--list-subs` 指定时，[YoutubeDL.py L3049-L3053](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3049-L3053) 调用 [render_subtitles_table](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L4051-L4063) 以表格形式展示所有可用字幕语言和格式，然后以 simulate 模式结束。

---

## 3. 字幕下载与写入（Subtitle Download & Write）

### 3.1 _write_subtitles

[_write_subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L4440-L4494) 在 [process_info](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3386) 中被调用：

```python
sub_files = self._write_subtitles(info_dict, temp_filename)
```

流程：

1. 检查是否需要写字幕（`writesubtitles` / `writeautomaticsub`），不需要则直接返回。
2. 调用 [subtitles_filename](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L1323-L1324) 生成文件名：`视频名.语言.扩展名`（如 `video.en.vtt`）。
3. 检查是否已存在同名字幕文件，存在则跳过下载。
4. 两种写入方式：
   - **`data` 不为 None**：直接将内存中的字幕内容写入文件（`open(sub_filename, 'w')`）。
   - **`data` 为 None**：通过 `self.dl(sub_filename, sub_copy, subtitle=True)` 从 URL 下载。
5. 写入成功后，将 `filepath` 记录到 `sub_info` 中（供后续格式转换和嵌入使用）。
6. 返回 `(sub_filename, sub_filename_final)` 元组列表，纳入 `files_to_move` 管理（临时文件 → 最终文件）。

### 3.2 subtitles_filename

[subtitles_filename](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L1323-L1324) 通过替换扩展名实现：

```python
def subtitles_filename(filename, sub_lang, sub_format, expected_real_ext=None):
    return replace_extension(filename, sub_lang + '.' + sub_format, expected_real_ext)
```

例如：`video.mp4` → `video.en.srt`。

---

## 4. 字幕格式转换（Subtitle Format Conversion）

### 4.1 触发条件

通过 `--convert-subs FORMAT` 命令行选项触发，在 [__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/__init__.py#L644-L649) 中注册为后处理器：

```python
if opts.convertsubtitles:
    yield {
        'key': 'FFmpegSubtitlesConvertor',
        'format': opts.convertsubtitles,
        'when': 'before_dl',   # 在下载完成后、后续处理前执行
    }
```

### 4.2 FFmpegSubtitlesConvertorPP

[FFmpegSubtitlesConvertorPP](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L939-L1012) 负责格式转换：

**支持的输出格式**：[MEDIA_EXTENSIONS.subtitles](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L5097) = `('srt', 'vtt', 'ass', 'lrc')`

**转换逻辑**：

```
对于 requested_subtitles 中的每种语言：
  ├─ 目标格式 == 原格式 → 跳过（已满足）
  ├─ 原格式 == 'json' → 跳过并警告（JSON 字幕无法转换）
  ├─ 原格式为 dfxp/ttml/tt → 先用 dfxp2srt() 转为 SRT
  │   ├─ 目标格式 == 'srt' → 完成
  │   └─ 否则 → 再用 ffmpeg 将 SRT 转为目标格式
  └─ 其他格式 → 直接用 ffmpeg 转换
```

**特殊处理：dfxp/ttml → srt**

[dfxp2srt](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L3414) 是一个纯 Python 解析器，将 TTML/DFXP XML 转换为 SRT 纯文本格式。它解析 TTML 的样式元素（颜色、字体、加粗、斜体、下划线），将其映射为 SRT 兼容的 HTML 标签。

### 4.3 ffmpeg 转换命令

```python
self.run_ffmpeg(old_file, new_file, ['-f', new_format])
```

其中 `new_format` 是 ffmpeg 的格式名（`vtt` → `webvtt`）。ffmpeg 负责处理 srt↔vtt、srt↔ass 等常见格式互转。

#### 4.3.1 转码后字幕文件与 `__files_to_move` 的集成

转换后的新字幕文件必须被**正确注册到移动映射**，才能跟着视频一起从临时目录搬到最终目录。这需要区分子目标格式就是 `srt`（dfxp→srt 为终点），还是还需进一步转换（dfxp→srt→vtt/ass，srt 为中间格式），两种情形处理逻辑差异很大，且代码中存在一处**设计缺陷**。

首先明确两段关键代码：

**① 新格式注册逻辑**（仅 ffmpeg 转换后才执行，L1009-L1010）：

```python
info['__files_to_move'][new_file] = replace_extension(
    info['__files_to_move'][sub['filepath']], new_ext)
```

**重要细节**：`sub` 是循环开始时从 `subs.items()` 取出的引用，而 dfxp2srt 分支中 `subs[lang] = {...}` 是**整体替换对象**（L989），因此 `sub` 仍指向**原始 dfxp 条目**，`sub['filepath']` 仍为 dfxp 原文件路径。这里是**碰巧正确**的实现——我们确实需要基于 dfxp 的最终路径来替换扩展名。

映射表含义：`key` 是临时目录中存在的源文件，`value` 是移动后的目标路径（value 为 `''` 或 `None` 表示默认根据 key 的文件名组合 `finaldir`）。

**② 旧文件回收逻辑**（run_pp L3798-L3818 + L3782）：

```python
files_to_delete, infodict = pp.run(infodict)
for filename in files_to_delete:
    if filename in info.get('__files_to_move', {}):  # L3782：从移动映射中删除
        del info['__files_to_move'][filename]
if self.params.get('keepvideo', False):              # -k 参数
    for f in files_to_delete:
        infodict['__files_to_move'].setdefault(f, '') # 保留文件 → 也搬去最终目录
else:
    self._delete_downloaded_files(*files_to_delete)   # 正常删除
```

---

下面分两种情形详述：

##### 情形 A：dfxp→srt（目标格式就是 srt，设计缺陷）

```
初始 __files_to_move：
  {'tmp/video.en.dfxp': 'final/video.en.dfxp'}

L970: old_file = 'tmp/video.en.dfxp'
L971: sub_filenames = ['tmp/video.en.dfxp']   ← dfxp 加入待删除
L972: new_file = 'tmp/video.en.srt'

L979-L987: dfxp2srt 转换，生成 srt_file = 'tmp/video.en.srt'
           old_file = 'tmp/video.en.srt'

L989-993: subs['en'] = {                       ← 整体替换对象，sub 仍指向旧对象
    'ext': 'srt',
    'data': srt_data,
    'filepath': 'tmp/video.en.srt'
}

L995: new_ext == 'srt' → TRUE
L996: continue  → 直接跳过 L1000-1010！
       ↓
       × 不会运行 ffmpeg
       × 不会执行 L1009-1010 的 __files_to_move 更新
       × srt 文件未注册到移动映射！
```

**后续处理**：
- `sub_filenames = ['tmp/video.en.dfxp']` → run_pp 中**删除 dfxp 文件**，同时**从 __files_to_move 删除 key `tmp/video.en.dfxp`**
- **srt 文件 `tmp/video.en.srt`**：既不在 `files_to_delete` 中，也不在 `__files_to_move` 中，**最终留在临时目录，不会被移动到最终目录**！

> **代码缺陷**：dfxp→srt 且目标就是 srt 时，srt 文件永远不会被移动到最终目录。如需修复，需在 `continue` 前补充注册 srt 文件到 `__files_to_move`。

---

##### 情形 B：dfxp→srt→vtt（目标格式非 srt，srt 为中间格式）

```
初始 __files_to_move：
  {'tmp/video.en.dfxp': 'final/video.en.dfxp'}

L970: old_file = 'tmp/video.en.dfxp'
L971: sub_filenames = ['tmp/video.en.dfxp']   ← dfxp 加入待删除
L972: new_file = 'tmp/video.en.vtt'

L979-L987: dfxp2srt 转换，生成 srt_file = 'tmp/video.en.srt'
           old_file = 'tmp/video.en.srt'

L989-993: subs['en'] = {'ext': 'srt', ..., 'filepath': 'tmp/video.en.srt'}

L995: new_ext != 'srt' → FALSE
L998: sub_filenames.append('tmp/video.en.srt')
      → sub_filenames = ['tmp/video.en.dfxp', 'tmp/video.en.srt']

L1000: run_ffmpeg('tmp/video.en.srt', 'tmp/video.en.vtt', ['-f', 'webvtt'])

L1002-1007: subs['en'] = {'ext': 'vtt', ..., 'filepath': 'tmp/video.en.vtt'}

L1009-1010: 注册 vtt 到移动映射
  info['__files_to_move']['tmp/video.en.vtt']
    = replace_extension(
        info['__files_to_move']['tmp/video.en.dfxp'], 'vtt')
                             ↑ sub['filepath'] 仍是 dfxp 路径（碰巧正确）
    = 'final/video.en.vtt'
```

**此时状态**：
```
__files_to_move = {
  'tmp/video.en.dfxp': 'final/video.en.dfxp',
  'tmp/video.en.vtt':  'final/video.en.vtt'
}
sub_filenames = ['tmp/video.en.dfxp', 'tmp/video.en.srt']
```

**后续处理**：
1. run_pp 处理 `files_to_delete = ['tmp/video.en.dfxp', 'tmp/video.en.srt']`
2. 从 `__files_to_move` 删除 `tmp/video.en.dfxp`（srt 不在其中，跳过）
3. 默认模式：**删除 dfxp 和 srt 文件**
4. 最终 `__files_to_move = {'tmp/video.en.vtt': 'final/video.en.vtt'}`
5. MoveFilesAfterDownloadPP 正常移动 vtt 到最终目录 ✓

---

##### 情形 C：普通格式转换（非 dfxp/ttml，如 vtt→srt）

无需中间格式，流程更简单：
- old_file 加入 `sub_filenames` 待删除
- ffmpeg 转换生成 new_file
- L1009-L1010 注册 new_file 到移动映射
- run_pp 删除 old_file 并从映射移除
- new_file 正常移动

---

## 5. 下载结果组织与最终移动

### 5.1 files_to_move 的生命周期

yt-dlp 所有"输出文件"都通过一个统一的字典 `files_to_move` 组织：`{临时文件路径: 最终文件路径}`。整个生命周期如下：

```
① 初始化
    YoutubeDL.process_info L3361: files_to_move = {}

② 字幕写入
    _write_subtitles 返回 [(sub_filename, sub_filename_final), ...]
    files_to_move.update(dict(sub_files))  # L3389

③ 缩略图/描述/infojson 等写入
    _write_thumbnails / _write_description / _write_info_json
    → 同样返回 [(tmp_final), ...]，并入 files_to_move

④ before_dl 阶段 PP（含字幕转换）
    pre_process(info_dict, 'before_dl', files_to_move)  # L3449
      → info['__files_to_move'] = files_to_move  (pre_process L3830)
      → 运行所有 when='before_dl' 的 PP
          · FFmpegSubtitlesConvertorPP 在此阶段执行：
              - 以新文件路径替换旧文件路径的映射
              - 返回 files_to_delete（原始字幕文件）
              - run_pp 根据 keepvideo 决定删除 or 保留
      → 返回 (new_info, 更新后的 __files_to_move)
    files_to_move ← 返回值

⑤ 视频下载
    下载 temp_filename，主视频文件本身加入 files_to_move：
      · 合并下载时（如 video+audio），各碎片文件同样注册
      · L3568-L3572 files_to_move[file] = None for file in downloaded  # None=用默认方式生成最终路径

⑥ post_process 阶段 PP（含字幕嵌入）
    post_process(dl_filename, info_dict, files_to_move)  # L3657
      → info['__files_to_move'] = files_to_move  (post_process L3842)
      → 运行所有 when='post_process' 的 PP
          · FFmpegEmbedSubtitlePP 在此阶段执行：
              - 读取 info['requested_subtitles'] 中的 filepath 并嵌入视频
              - 返回 files_to_delete（字幕文件），run_pp 中同样按 keepvideo 删除/保留
              - 注意：嵌入 PP 不会修改字幕文件的映射，但会删除被嵌入的字幕文件
      → 运行 MoveFilesAfterDownloadPP（见 5.2）
      → 运行所有 when='after_move' 的 PP
```

### 5.2 MoveFilesAfterDownloadPP：真正的搬移动作

[MoveFilesAfterDownloadPP.run()](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/movefilesafterdownload.py#L21-L53) 在所有后处理器之后、`after_move` 阶段之前执行：

```python
def run(self, info):
    dl_path, dl_name = os.path.split(info['filepath'])
    finaldir = info.get('__finaldir', dl_path)
    finalpath = os.path.join(finaldir, dl_name)

    # ① 主视频文件自己也需要搬，所以补注册
    if self._downloaded:
        info['__files_to_move'][info['filepath']] = finalpath

    # ② 遍历整个 files_to_move 映射，逐个 shutil.move
    make_newfilename = lambda old: os.path.join(finaldir, os.path.basename(old))
    for oldfile, newfile in info['__files_to_move'].items():
        if not newfile:                                    # value 为空 → 默认移动到 finaldir
            newfile = make_newfilename(oldfile)
        if os.path.abspath(oldfile) == os.path.abspath(newfile):
            continue                                        # 路径相同，无需移动
        if not os.path.exists(oldfile):
            self.report_warning(f'File "{oldfile}" cannot be found')
            continue
        if os.path.exists(newfile) and not self.get_param('overwrites', True):
            ...                                             # 已存在且非覆盖模式 → 跳过
        make_parent_dirs(newfile)
        self.to_screen(f'Moving file "{oldfile}" to "{newfile}"')
        shutil.move(oldfile, newfile)                       # 跨卷也安全

    info['filepath'] = finalpath
    return [], info
```

### 5.3 字幕文件在移动链中的四种结局

| 场景 | 处理方式 | 代码位置 |
|---|---|---|
| 只 `--write-subs` | 字幕文件正常 `files_to_move`，被 MoveFilesAfterDownloadPP 搬到 finaldir | [YoutubeDL L3386-3389](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3386-L3389) |
| `--write-subs --convert-subs FORMAT`（普通格式/dfxp 转非 srt） | 转码后新格式字幕替换旧文件映射注册，旧格式被 run_pp 删除（带 `-k` 则保留并一起搬） | [ffmpeg.py L1009-1010](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L1009-L1010) + [YoutubeDL L3798-L3818](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3798-L3818) |
| `--write-subs --convert-subs srt`（dfxp→srt 终点）⚠️ **设计缺陷** | dfxp 原文件被删除，srt 文件未注册到 `__files_to_move`，**不会被移动到最终目录，留在临时目录** | [ffmpeg.py L995-L996](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L995-L996) `continue` 跳过注册 |
| `--embed-subs` 且未保留 | 字幕嵌入后被 FFmpegEmbedSubtitlePP 列为 files_to_delete，由 run_pp 删除，因此不会出现在最终目录 | [ffmpeg.py L658](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L658) |
| `--embed-subs --write-subs` | 嵌入时 `already_have_subtitle=True`，因此不在 files_to_delete 中，继续留在 files_to_move 随视频一起搬移 | [__init__.py L674-679](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/__init__.py#L674-L679) + [ffmpeg.py L658](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L658) |

---

## 6. 字幕嵌入（Subtitle Embedding）

### 6.1 触发条件

通过 `--embed-subs` 选项触发，在 [__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/__init__.py#L674-L679) 中注册：

```python
if opts.embedsubtitles:
    keep_subs = 'no-keep-subs' not in opts.compat_opts
    yield {
        'key': 'FFmpegEmbedSubtitle',
        'already_have_subtitle': opts.writesubtitles and keep_subs,
    }
```

### 6.2 FFmpegEmbedSubtitlePP

[FFmpegEmbedSubtitlePP](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L581-L659) 将字幕轨嵌入视频容器：

**支持嵌入的容器**：`('mp4', 'mov', 'm4a', 'webm', 'mkv', 'mka')`

**嵌入规则**：

| 容器 | 可嵌入的字幕格式 | 限制 |
|---|---|---|
| mp4/mov/m4a/mkv/mka | srt, vtt, ass, ... | ASS 嵌入 mp4 时会警告"可能有兼容问题" |
| webm | 仅 vtt | 非 VTT 字幕嵌入 webm 会警告并跳过 |
| 任意 | json | 不可嵌入，直接跳过 |

**ffmpeg 命令构造**：

```
ffmpeg -i video.mp4 -i sub1.vtt -i sub2.srt \
       -map 0:v -map 0:a -map -0:s \   # 不复制原有字幕轨
       -map 1:0 -metadata:s:s:0 language=eng \
       -map 2:0 -metadata:s:s:1 language=chi \
       -c copy output.temp.mp4
```

`already_have_subtitle` 参数控制嵌入后是否删除字幕文件：若用户同时指定了 `--write-subs`，则保留字幕文件；否则嵌入后删除。

---

## 7. 端到端流程总结

```
用户命令行参数
  │
  ├─ --write-subs / --write-auto-subs / --all-subs / --sub-langs
  ├─ --sub-format (默认 'best')
  ├─ --convert-subs FORMAT
  └─ --embed-subs
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Extractor._real_extract()                                    │
│   ├─ 站点 API: _get_subtitles() → {lang: [{ext,url,data,...}]}  │
│   ├─ 流清单: _extract_m3u8/mpd/smil_formats_and_subtitles()    │
│   │   · 各层 determine_ext 推断 ext, M3U8 子清单→硬编码 vtt     │
│   └─ _merge_subtitles() 合并多个来源                             │
│   → info_dict['subtitles'] + info_dict['automatic_captions']    │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. YoutubeDL.process_info()                                     │
│   ├─ URL 清理 + 最后一道 determine_ext 推断 (L2901-2909)        │
│   ├─ process_subtitles() → requested_subtitles                  │
│   │   ├─ 语言筛选 (writesubtitles/writeautomaticsub/subslangs)  │
│   │   └─ 格式筛选 (subtitlesformat → best/srt/vtt/...)          │
│   └─ listsubtitles? → 渲染表格 → simulate 退出                  │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. _write_subtitles()                                           │
│   ├─ 已存在? → 跳过                                             │
│   ├─ data 模式 → 直接写文件                                     │
│   └─ url 模式 → self.dl() 下载写文件                            │
│   → sub_info['filepath'] = 写入的文件路径                        │
│   → 返回 [(tmp_sub, final_sub)] 加入 files_to_move 字典         │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. before_dl PP: FFmpegSubtitlesConvertorPP                     │
│   ├─ json → 不可转换，跳过                                       │
│   ├─ dfxp/ttml → dfxp2srt() → srt                               │
│   └─ 其他 → ffmpeg -f <target_format> 转换                      │
│   → 更新 sub_info 的 ext/data/filepath                          │
│   → __files_to_move[new_file] = final_path（替换映射）          │
│   → 返回 files_to_delete → run_pp 按 keepvideo 删除/保留        │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 视频下载 + 碎片文件加入 files_to_move                        │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. post_process PP: FFmpegEmbedSubtitlePP                       │
│   ├─ 检查容器兼容性 (mp4/mkv/webm/...)                          │
│   ├─ 检查字幕格式兼容性 (webm仅vtt, json不可嵌入)               │
│   └─ ffmpeg -map 嵌入字幕轨 + 设置语言 metadata                 │
│   → 嵌入后根据 already_have_subtitle 决定是否删除字幕文件       │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. MoveFilesAfterDownloadPP                                     │
│   ├─ 补注册视频主文件的移动映射                                  │
│   └─ 遍历 __files_to_move.items(): shutil.move 到最终目录       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. 关键代码位置索引

| 功能 | 文件 | 行号 |
|---|---|---|
| 字幕数据结构定义 | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L331-L339) | L331-339 |
| extract_subtitles 入口 | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3885-L3892) | L3885-3892 |
| _merge_subtitles | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3938-L3946) | L3938-3946 |
| M3U8 字幕解析（含 determine_ext 调用） | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2284-L2311) | L2284-2311 |
| SMIL textstream 字幕解析（含 determine_ext 调用） | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2730-L2742) | L2730-L2742 |
| DASH MPD 字幕解析 | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3202-L3209) | L3202-L3209 |
| DASH 字幕 ext 设置（mimetype2ext，不用 determine_ext） | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2989) | L2989 |
| ISM 字幕 ext 硬编码（不用 determine_ext） | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3304) | L3304 |
| HTML5 track 字幕解析（未设置 ext，依赖全局推断） | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3461-L3471) | L3461-L3471 |
| determine_ext 推断扩展名 | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L1310-L1320) | L1310-1320 |
| process_subtitles | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3154-L3212) | L3154-3212 |
| URL清理+全局ext推断（最后防线） | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L2901-L2909) | L2901-L2909 |
| _write_subtitles | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L4440-L4494) | L4440-4494 |
| subtitles_filename | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L1323-L1324) | L1323-1324 |
| dfxp2srt 转换 | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L3414) | L3414 |
| FFmpegSubtitlesConvertorPP 主流程 | [ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L939-L1012) | L939-1012 |
| subs[lang] 整体替换（导致 sub 引用旧对象） | [ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L989-L993) | L989-L993 |
| dfxp→srt 终点跳过注册（设计缺陷） | [ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L995-L996) | L995-L996 |
| 转码字幕注册到__files_to_move | [ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L1009-L1010) | L1009-1010 |
| FFmpegEmbedSubtitlePP | [ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L581-L659) | L581-L659 |
| run_pp: 从__files_to_move删除待删除文件键 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3782-L3785) | L3782-L3785 |
| run_pp: files_to_delete 删除/保留逻辑 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3798-L3818) | L3798-L3818 |
| pre_process / post_process 生命周期 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3828-L3846) | L3828-3846 |
| MoveFilesAfterDownloadPP | [movefilesafterdownload.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/movefilesafterdownload.py#L11-L53) | L11-53 |
| 后处理器注册 | [__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/__init__.py#L644-L679) | L644-679 |
| YouTube _SUBTITLE_FORMATS | [youtube/_video.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/youtube/_video.py#L142) | L142 |
| YouTube process_language | [youtube/_video.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/youtube/_video.py#L4199-L4211) | L4199-4211 |
| MEDIA_EXTENSIONS.subtitles | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L5097) | L5097 |
| CLI 字幕选项定义 | [options.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/options.py#L971-L1002) | L971-1002 |
| files_to_move 初始化+字幕注册 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3361-L3389) | L3361-3389 |
