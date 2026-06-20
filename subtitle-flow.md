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

通过 `--convert-subs FORMAT` 命令行选项触发，在 [\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/__init__.py#L644-L649) 中注册为后处理器：

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

---

## 5. 字幕嵌入（Subtitle Embedding）

### 5.1 触发条件

通过 `--embed-subs` 选项触发，在 [\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/__init__.py#L674-L679) 中注册：

```python
if opts.embedsubtitles:
    keep_subs = 'no-keep-subs' not in opts.compat_opts
    yield {
        'key': 'FFmpegEmbedSubtitle',
        'already_have_subtitle': opts.writesubtitles and keep_subs,
    }
```

### 5.2 FFmpegEmbedSubtitlePP

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

## 6. 端到端流程总结

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
│   └─ _merge_subtitles() 合并多个来源                             │
│   → info_dict['subtitles'] + info_dict['automatic_captions']    │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. YoutubeDL.process_info()                                     │
│   ├─ URL 清理 + ext 推断 (L2901-2909)                           │
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
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. FFmpegSubtitlesConvertorPP (when='before_dl')                │
│   ├─ json → 不可转换，跳过                                       │
│   ├─ dfxp/ttml → dfxp2srt() → srt                               │
│   └─ 其他 → ffmpeg -f <target_format> 转换                      │
│   → 更新 sub_info 的 ext/data/filepath                          │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. FFmpegEmbedSubtitlePP                                        │
│   ├─ 检查容器兼容性 (mp4/mkv/webm/...)                          │
│   ├─ 检查字幕格式兼容性 (webm仅vtt, json不可嵌入)               │
│   └─ ffmpeg -map 嵌入字幕轨 + 设置语言 metadata                 │
│   → 嵌入后根据 already_have_subtitle 决定是否删除字幕文件       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 关键代码位置索引

| 功能 | 文件 | 行号 |
|---|---|---|
| 字幕数据结构定义 | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L331-L339) | L331-339 |
| extract_subtitles 入口 | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3885-L3892) | L3885-3892 |
| _merge_subtitles | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3938-L3946) | L3938-3946 |
| M3U8 字幕解析 | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L2284-L2311) | L2284-2311 |
| DASH MPD 字幕解析 | [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/common.py#L3208-L3209) | L3208-3209 |
| process_subtitles | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L3154-L3212) | L3154-3212 |
| URL清理+ext推断 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L2901-L2909) | L2901-2909 |
| _write_subtitles | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/YoutubeDL.py#L4440-L4494) | L4440-4494 |
| subtitles_filename | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L1323-L1324) | L1323-1324 |
| dfxp2srt 转换 | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L3414) | L3414 |
| FFmpegSubtitlesConvertorPP | [ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L939-L1012) | L939-1012 |
| FFmpegEmbedSubtitlePP | [ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L581-L659) | L581-659 |
| 后处理器注册 | [\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/__init__.py#L644-L679) | L644-679 |
| YouTube _SUBTITLE_FORMATS | [youtube/_video.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/youtube/_video.py#L142) | L142 |
| YouTube process_language | [youtube/_video.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/extractor/youtube/_video.py#L4199-L4211) | L4199-4211 |
| MEDIA_EXTENSIONS.subtitles | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/utils/_utils.py#L5097) | L5097 |
| CLI 字幕选项定义 | [options.py](file:///d:/fz/0601-2/solo-dogfeeding/code/91-yt-dlp/yt_dlp/options.py#L971-L1002) | L971-1002 |
