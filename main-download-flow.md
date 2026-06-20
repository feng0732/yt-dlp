# yt-dlp 主调度流程梳理

本文档梳理从 URL 输入到下载完成后处理的完整主调度链路，涵盖任务状态流转、列表/视频分支判断、下载阶段划分以及后处理时机衔接。

---

## 1. 入口与顶层调度

### 1.1 程序入口

入口文件为 [__main__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/__main__.py)，调用 `yt_dlp.main()`。参数解析与实例化后，核心调度类为 `YoutubeDL`（定义于 [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py)）。

### 1.2 顶层下载方法：`download(url_list)`

位置：[YoutubeDL.py#L3694-L3708](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3694-L3708)

```
for url in url_list:
    self.__download_wrapper(self.extract_info)(url, ...)
```

- 对每个 URL 用 `__download_wrapper` 包裹 `extract_info` 调用
- `__download_wrapper` 负责异常捕获、`dump_single_json` 输出等外围逻辑
- 最终返回 `self._download_retcode`

包裹器 [__download_wrapper](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3674-L3692) 处理：
- `CookieLoadError`、`UnavailableVideoError`、`DownloadCancelled` 等异常
- 正常完成时若设置 `dump_single_json` 则输出 JSON

---

## 2. 提取阶段（Extract）

### 2.1 `extract_info` — URL 分发

位置：[YoutubeDL.py#L1671-L1719](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L1671-L1719)

核心职责：遍历所有注册的 `InfoExtractor`，根据 `ie.suitable(url)` 匹配合适的提取器，然后调用 `__extract_info`。

关键检查：
- 下载归档检查 `in_download_archive` — 通过临时 ID 判断是否已下载
- 强制通用提取器 `force_generic_extractor` 支持
- 指定 `ie_key` 时只使用该提取器

### 2.2 `__extract_info` — 实际提取 + 异常处理循环

位置：[YoutubeDL.py#L1856-L1884](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L1856-L1884)

由装饰器 `_handle_extraction_exceptions` 包裹，形成 **重试循环**：

| 异常类型 | 处理方式 |
|---------|---------|
| `ReExtractInfo` | 重新提取（如等待视频发布时间到了之后重试） |
| `GeoRestrictedError` | 报告错误，提示使用 VPN |
| `ExtractorError` | 报告错误，输出格式化 traceback |
| 其他异常 | `ignoreerrors=True` 时吞掉，否则抛出 |

提取流程：
1. `ie.extract(url)` — 具体站点提取器执行解析
2. 若返回 `list`，包装为 `_type='compat_list'` （老格式兼容）
3. `add_default_extra_info` — 注入 `webpage_url`、`extractor`、`extractor_key` 等元数据
4. 若 `process=True`：先 `_wait_for_video`（处理未开播/待发布直播），再进入 `process_ie_result`

### 2.3 任务状态字段 `_type`

提取结果通过 `_type` 字段标识任务类型，驱动后续分支：

| `_type` 值 | 含义 | 处理分支 |
|-----------|------|---------|
| `video` | 单个视频（默认） | 进入 `process_video_result` |
| `url` | URL 转发（需要再次 extract） | 递归 `extract_info` |
| `url_transparent` | 透明 URL（保留外层元数据） | 递归提取后合并元数据 |
| `playlist` | 播放列表 | 进入 `__process_playlist` |
| `multi_video` | 多视频集合（同 playlist 处理） | 进入 `__process_playlist` |
| `compat_list` | 老格式列表（兼容） | 逐个递归 `process_ie_result` |

---

## 3. 列表分支与任务分发：`process_ie_result`

位置：[YoutubeDL.py#L1904-L2036](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L1904-L2036)

这是整个调度流程的 **核心分发器**，根据 `_type` 做分支处理：

### 3.1 `video` 分支（L1939-L1957）

```
add_extra_info → process_video_result → _raise_pending_errors
                → 若有 additional_urls，逐个 extract_info 作为 additional_entries
```

### 3.2 `url` 分支（L1958-L1964）

直接递归调用 `extract_info`，将 `extra_info`（如播放列表上下文）一并传递。

### 3.3 `url_transparent` 分支（L1965-L1995）

"透明" 表示外层页面（如嵌入页面）的元数据需要保留：
1. 先 `process=False` 提取内部 URL（不展开）
2. 将 `ie_result` 中非豁免字段合并到内部结果
3. 若内部结果仍是 `url`，则改为 `url_transparent` 继续递归分发

### 3.4 `playlist` / `multi_video` 分支（L1996-L2015）

调用 `__process_playlist`，前后做递归保护：
- `_playlist_level` 记录嵌套层级
- `_playlist_urls` 集合防止同 URL 无限循环

### 3.5 播放列表处理：`__process_playlist`

位置：[YoutubeDL.py#L2076-L2192](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L2076-L2192)

核心步骤：
1. `PlaylistEntries` 获取条目并做 `requested_items` 过滤
2. `lazy_playlist` 模式下不预解析全部条目（节省内存）
3. 可选写入播放列表级文件：`infojson`、`description`、`thumbnail`
4. 可选 `playlistreverse` / `playlistrandom` 排序
5. 遍历条目，逐个调用 `__process_iterable_entry` → `process_ie_result`
6. 支持 `skip_playlist_after_errors`：失败次数达到阈值时跳过剩余条目
7. 最后执行 `run_all_pps('playlist', ...)` — 播放列表级后处理

每个条目被注入的上下文字段（通过 `collections.ChainMap` 叠加）：
- `playlist` / `playlist_id` / `playlist_title` / `playlist_index` / `playlist_autonumber`
- `playlist_uploader` / `playlist_channel` / `n_entries`

---

## 4. 下载阶段：`process_video_result`

位置：[YoutubeDL.py#L2832-L3152](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L2832-L3152)

### 4.1 格式准备（L2834-L3033）

1. 字段规范化：`id`、数字字段、`duration`、`chapters` 补齐
2. `_sanitize_thumbnails` — 缩略图标准化
3. `process_subtitles` — 处理字幕/自动字幕选择
4. `_get_formats` — 获取格式列表，过滤 DRM 格式、直播格式过滤
5. 每个 format 注入 `http_headers`、补全 `resolution`、`filesize_approx` 等字段
6. `sort_formats` — 按 `_format_sort_fields` 排序
7. `format_id` 去重（同名加 `-i` 后缀；与扩展名冲突加 `f` 前缀）

### 4.2 预处理与过滤（L3034-L3044）

```
pre_process(info_dict)                    # key='pre_process'
_match_entry 过滤                         # 匹配标题/日期等条件
post_extract(info_dict)                   # 执行 __post_extractor 回调
pre_process(info_dict, 'after_filter')    # key='after_filter'
```

### 4.3 格式选择（L3045-L3097）

- `listformats` / `listsubtitles` / `list_thumbnails` — 仅列出，不下载
- `interactive_format_selection`（`-f -`）— 交互式输入格式选择器
- 默认使用 `_default_format_spec` 构造选择器
- `_select_formats` 按选择器表达式选出 `formats_to_download`

### 4.4 实际下载循环（L3098-L3148）

双层循环：`formats_to_download × requested_ranges`

对每个 (format, chapter_range) 组合：
1. 复制 infodict 并注入 format 字段 + 分段字段（`section_start/end`）
2. 调用 **`process_info`** — 单个格式的完整下载流程
3. 捕获 `MaxDownloadsReached` 终止循环

下载完成后：
- `record_download_archive` — 记录下载归档（所有格式都成功时）
- `info_dict['requested_downloads']` — 保存所有下载格式详情
- `run_all_pps('after_video', info_dict)` — 视频级后处理
- `info_dict.update(best_format)` — 兼容旧代码，最佳格式字段上浮

---

## 5. 单格式下载流程：`process_info`

位置：[YoutubeDL.py#L3331-L3673](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3331-L3673)

### 5.1 下载前准备（L3334-L3450）

1. `_match_entry` — 再次过滤（可能被 PP 修改后再次检查）
2. `pre_process(info_dict, 'video')` — 第三次预处理
3. `prepare_filename` — 计算 `full_filename` 和 `temp_filename`
4. `__forced_printings` — 强制输出（`--print`、`--print-to-file`）
5. 检查 `simulate` — 模拟模式直接返回
6. 写入附属文件：
   - `description` / `subtitles` / `thumbnails` / `infojson`
   - 互联网快捷方式：`url` / `webloc` / `desktop`
7. `pre_process(info_dict, 'before_dl', files_to_move)` — 下载前最后一次 PP

### 5.2 下载分支判断

#### 分支 A：`skip_download=True`（L3452-L3457）

不下载实际媒体文件，只：
- 运行 `MoveFilesAfterDownloadPP` 移动附属文件
- 可选写入下载归档

#### 分支 B：多格式合并下载（L3482-L3572）

当 `requested_formats` 存在（如视频+音频分离的 DASH/HLS）：

```
if fd.can_merge_formats:
    # 同时下载所有格式到分文件，再合并
    info_dict['url'] = '\n'.join(各格式URL)
    self.dl(temp_filename, info_dict)
    # FFmpegMergerPP 被加入 __postprocessors
else:
    # 逐个单独下载各格式
    for f in requested_formats:
        self.dl(fname, new_info)
    # 若可用，加入 FFmpegMergerPP 做后续合并
```

合并后处理：`merger` 和 `__files_to_merge` 被放入 `__postprocessors`，等 `post_process` 阶段执行。

#### 分支 C：单文件下载（L3573-L3583）

```
dl_filename = existing_file or temp_filename
if 已存在完整文件: report_file_already_downloaded
else: self.dl(temp_filename, info_dict)
```

### 5.3 核心下载调用：`dl`

位置：[YoutubeDL.py#L3283-L3318](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3283-L3318)

```
fd = get_suitable_downloader(info, params)(self, params)
# 注入 progress_hooks
return fd.download(name, new_info, subtitle)
```

`get_suitable_downloader` 根据 `protocol` 字段选择具体下载器：

| 协议类型 | 下载器类 | 文件位置 |
|---------|---------|---------|
| `http` / `https` | `HttpFD` | [downloader/http.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/http.py) |
| `m3u8` / `m3u8_native` | `HlsFD` | [downloader/hls.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/hls.py) |
| `dash` | `DashSegmentsFD` | [downloader/dash.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/dash.py) |
| `f4m` | `F4mFD` | [downloader/f4m.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/f4m.py) |
| `ism` | `IsmFD` | [downloader/ism.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/ism.py) |
| `rtmp` | `RtmpFD` | [downloader/rtmp.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/rtmp.py) |
| `mhtml` | `MhtmlFD` | [downloader/mhtml.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/mhtml.py) |
| `http_dash_segments` 等分段 | `FragmentFD` 子类 | [downloader/fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/fragment.py) |
| 外部程序 | `ExternalFD` 子类（aria2c、avconv、curl、ffmpeg、wget 等） | [downloader/external.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/external.py) |
| `ffmpeg` 兜底 | `FFmpegFD` | [downloader/common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/common.py) |

返回值：`(success: bool, real_download: bool)`

### 5.4 Fixup 阶段：自动修复（L3599-L3655）

根据 `fixup` 参数（`detect_or_warn`/`warn`/`force`/`ignore`），自动将必要的 FFmpeg 修复 PP 加入 `__postprocessors`：

| 修复器 | 触发条件 |
|--------|---------|
| `FFmpegFixupStretchedPP` | 非 1:1 像素比 |
| `FFmpegFixupM4aPP` | DASH m4a 容器 |
| `FFmpegFixupM3u8PP` | HLS 原生下载 / 直播 |
| `FFmpegFixupDuplicateMoovPP` | DASH 分段 / 直播多周期 |
| `FFmpegFixupTimestampPP` | WebSocket 分片下载 |
| `FFmpegFixupDurationPP` | WebSocket 分片下载 |

### 5.5 后处理调用（L3656-L3667）

```
post_process(dl_filename, info_dict, files_to_move)
→ run_all_pps('post_process', ..., additional_pps=__postprocessors)
→ run_pp(MoveFilesAfterDownloadPP)
→ run_all_pps('after_move', info)
→ post_hooks 回调
→ __write_download_archive = True
```

---

## 6. 后处理（Post-Process）体系

### 6.1 后处理时机枚举

定义于 [utils/_utils.py#L2858](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/utils/_utils.py#L2858)：

```python
POSTPROCESS_WHEN = (
    'pre_process',   # 提取完成后最早的处理（格式选择前）
    'after_filter',  # 条目匹配过滤之后
    'video',         # 视频处理开始（字幕/缩略图写入前）
    'before_dl',     # 实际下载开始前
    'post_process',  # 下载完成后，核心处理（转码、合并、嵌入等）
    'after_move',    # 文件移动到最终目录后
    'after_video',   # 整个视频（含多格式）全部完成后
    'playlist',      # 整个播放列表完成后
)
```

### 6.2 时机注入点全景

| 时机 | 触发位置 | 用途示例 |
|-----|---------|---------|
| `pre_process` | `process_ie_result` L1931 / `process_video_result` L3034 | 元数据解析（MetadataParserPP） |
| `after_filter` | `process_video_result` L3040 | 过滤后二次修改 |
| `video` | `process_info` L3354 | ModifyChapters、SponsorBlock 等 |
| `before_dl` | `process_info` L3349 | 下载前修改文件名、路径 |
| `post_process` | `post_process` L3843 | FFmpegMerger、EmbedThumbnail、FFmpegVideoConvertor、XAttrPP、ExecPP 等核心 PP；额外加 `info['__postprocessors']` 中的（含 merger、各种 fixup） |
| `after_move` | `post_process` L3846 | 文件已在最终目录时的操作 |
| `after_video` | `process_video_result` L3146 | 整个视频完成时的回调 |
| `playlist` | `__process_playlist` L2190 | 播放列表级元数据处理 |

### 6.3 `post_process` 方法 — 下载后处理衔接

位置：[YoutubeDL.py#L3839-L3846](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3839-L3846)

```python
def post_process(self, filename, info, files_to_move=None):
    info['filepath'] = filename
    info['__files_to_move'] = files_to_move or {}
    # 1. 核心后处理（含 merger、fixup、转码、嵌入等）
    info = self.run_all_pps('post_process', info, additional_pps=info.get('__postprocessors'))
    # 2. 移动临时文件到最终目录 + 移动附属文件
    info = self.run_pp(MoveFilesAfterDownloadPP(self), info)
    del info['__files_to_move']
    # 3. 最终目录中的后处理
    return self.run_all_pps('after_move', info)
```

### 6.4 PP 执行机制

- `run_pp(pp, infodict)` — 执行单个 PP，返回 `(files_to_delete, new_infodict)`
  - `keepvideo=True` 时不删除原始文件，保留到 `__files_to_move`
  - `ignoreerrors=True` 时吞掉 `PostProcessingError`
- `run_all_pps(key, info, additional_pps=None)` — 按时机批量执行
  - `additional_pps` 在该时机的注册 PP 之前执行（用于动态注入的 merger/fixup）
  - 非 `video` 时机先执行 `_forceprint` 输出

每个 PP 的 `run` 方法被 `PostProcessorMetaClass.run_wrapper` 包裹（[postprocessor/common.py#L16-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/postprocessor/common.py#L16-L33)），自动发送 progress hook 事件（`started` / `finished`）。

### 6.5 常见后处理器

| 后处理器 | 时机 | 功能 |
|---------|------|------|
| `FFmpegMergerPP` | `post_process`（动态注入） | 合并音视频多格式 |
| `FFmpegVideoConvertorPP` | `post_process` | 格式转码（mp4/webm/mkv 等） |
| `EmbedThumbnailPP` | `post_process` | 嵌入缩略图到媒体文件 |
| `EmbedSubtitlePP`（FFmpeg） | `post_process` | 嵌入字幕 |
| `FFmpegSubtitlesConvertorPP` | `post_process` | 字幕格式转换 |
| `FFmpegThumbnailsConvertorPP` | `post_process` | 缩略图格式转换 |
| `FFmpegFixup*PP` | `post_process`（动态注入） | 容器/时间戳修复 |
| `MetadataParserPP` | `pre_process` | 从文件名/元数据解析字段 |
| `ModifyChaptersPP` | `video` | 增删改章节 |
| `SponsorBlockPP` | `video` / `post_process` | 标记/移除 SponsorBlock 片段 |
| `XAttrPP` | `post_process` | 写入扩展属性 |
| `ExecPP` | 各时机 | 执行外部命令 |
| `MoveFilesAfterDownloadPP` | `post_process` 末尾（硬编码） | 移动文件到最终目录 |

---

## 7. 完整调用链总览

```
YoutubeDL.download(url_list)
│
├─ for each url:
│   └─ __download_wrapper(extract_info)(url)
│       │
│       ├─ extract_info(url)
│       │   ├─ 匹配 IE
│       │   └─ __extract_info(url, ie)   ← 被 _handle_extraction_exceptions 包裹（ReExtractInfo 重试）
│       │       ├─ ie.extract(url)  ──── 返回 ie_result（含 _type 字段）
│       │       ├─ add_default_extra_info
│       │       ├─ _wait_for_video（直播/待发布）
│       │       └─ process_ie_result(ie_result)
│       │           │
│       │           ├─ [_type=video] ──────────────────────────────┐
│       │           ├─ [_type=url] ─→ extract_info(递归) ──────────┤
│       │           ├─ [_type=url_transparent] ─→ 合并后再递归 ────┤
│       │           ├─ [_type=playlist/multi_video]                │
│       │           │   └─ __process_playlist                      │
│       │           │       ├─ 获取 entries（lazy 或 eager）        │
│       │           │       ├─ 写入 playlist infojson/描述/缩略图  │
│       │           │       ├─ for entry in entries:               │
│       │           │       │   └─ __process_iterable_entry        │
│       │           │       │       └─ process_ie_result(递归) ←───┘
│       │           │       └─ run_all_pps('playlist')
│       │           └─ [_type=compat_list] ─→ 逐条递归
│       │
│       └─ （返回值或异常被 __download_wrapper 捕获）
│
└─ return _download_retcode


=== 单个 video 展开 ===

process_video_result(info_dict, download=True)
├─ 字段规范化 + 字幕/缩略图处理
├─ formats 获取、过滤、排序、format_id 去重
├─ pre_process('pre_process')          ← PP 时机 1
├─ _match_entry 过滤
├─ post_extract
├─ pre_process('after_filter')         ← PP 时机 2
├─ 格式选择（_select_formats）
│
├─ [download=True]
│   └─ for fmt × requested_ranges:
│       └─ process_info(new_info)
│           ├─ _match_entry
│           ├─ pre_process('video')    ← PP 时机 3
│           ├─ 计算文件名（full + temp）
│           ├─ __forced_printings（--print 等）
│           ├─ [simulate] → 直接返回
│           ├─ 写入附属文件（sub/thumb/infojson/link）
│           ├─ pre_process('before_dl') ← PP 时机 4
│           │
│           ├─ [skip_download] → MoveFilesAfterDownloadPP + 返回
│           │
│           ├─ [requested_formats（多格式合并）]
│           │   ├─ fd.can_merge_formats ? 同时下载 : 逐个下载
│           │   │   └─ self.dl(filename, info)
│           │   │       └─ get_suitable_downloader(...)
│           │   │           └─ HttpFD / HlsFD / DashSegmentsFD / ...
│           │   └─ 可用时注入 FFmpegMergerPP 到 __postprocessors
│           │
│           ├─ [单文件]
│           │   └─ self.dl(temp_filename, info_dict)
│           │
│           ├─ fixup() — 按需注入 FFmpegFixup*PP
│           │
│           └─ post_process(dl_filename, info, files_to_move)
│               ├─ run_all_pps('post_process') + __postprocessors  ← PP 时机 5（含 merger/fixup/转码/嵌入）
│               ├─ run_pp(MoveFilesAfterDownloadPP)                ← 移动到最终目录
│               └─ run_all_pps('after_move')                       ← PP 时机 6
│
├─ run_all_pps('after_video')          ← PP 时机 7（整个视频完成）
└─ record_download_archive
```

---

## 8. 关键状态/计数器字段

| 字段 | 用途 | 更新位置 |
|-----|------|---------|
| `self._num_videos` | 处理过的视频数（含模拟） | `process_video_result` 入口 L2834 |
| `self._num_downloads` | 实际下载计数 | `process_info` L3356 |
| `self._playlist_level` | 播放列表嵌套层级（防递归） | `process_ie_result` L2006/L2013 |
| `self._playlist_urls` | 已处理播放列表 URL 集合（防循环） | `process_ie_result` L2007/L2015 |
| `self._download_retcode` | 最终返回码 | 各处 `report_error` 时置 1 |
| `info_dict['_type']` | 任务类型驱动分发 | 提取器设置 + 兼容包装 |
| `info_dict['requested_formats']` | 需合并的多格式列表 | 格式选择阶段 |
| `info_dict['requested_downloads']` | 已下载的所有格式详情 | `process_video_result` 末尾 L3145 |
| `info_dict['__postprocessors']` | 动态注入的 PP（merger、fixup） | `process_info` 下载/修复阶段 |
| `info_dict['__files_to_move']` | 需移动的附属文件映射 | 贯穿 pre_process / post_process |
| `info_dict['__write_download_archive']` | 外层格式组（粒度 2）的归档标记（True/False/'ignore'）。不存在于内层子格式（粒度 3） | `process_info` 各分支（见第 10.5 节失败边界） |

---

## 9. 失败状态与返回码传播机制

### 9.1 核心错误方法：`trouble` → `report_error`

错误传播的根方法为 [trouble](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L1068-L1100)，[report_error](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L1155-L1160) 是其封装（加上 `ERROR:` 前缀）。

```
trouble(message, tb=None, is_error=True)
├─ 输出错误信息到 stderr
├─ [verbose] 输出 traceback
├─ [is_error=False] → 仅警告，不影响返回码
├─ [ignoreerrors=False] → 抛出 DownloadError(message, exc_info) ← 进程级中断
└─ [ignoreerrors=True]  → self._download_retcode = 1 ← 继续运行，但标记失败
```

**关键双轨逻辑**：`ignoreerrors` 参数决定 `report_error` 的行为是"抛异常中断"还是"置返回码继续"。返回码只有 `0`（成功）和 `1`（失败）两种，一旦任何 `report_error(is_error=True)` 被调用且 `ignoreerrors=True`，返回码就会变为 1 且无法恢复。

### 9.2 返回码 `_download_retcode` 的生命周期

| 阶段 | 位置 | 操作 |
|------|------|------|
| 初始化 | [L647](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L647) | `self._download_retcode = 0` |
| 置 1 | [L1100](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L1100) | `report_error` 内 `self._download_retcode = 1` |
| 保存/恢复 | [L2281-L2293](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L2281-L2293) | `_check_formats` 测试格式时暂存并恢复，避免测试失败污染返回码 |
| 最终返回 | [L3708](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3708) | `download()` 返回 `self._download_retcode` |

### 9.3 下载器 `dl()` 的 `success` 返回值传播

[FileDownloader.download](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/common.py#L430-L482) 返回 `(success, real_download)`：

| 场景 | success | real_download |
|------|---------|--------------|
| 文件已存在（`continuedl` 或 `nooverwrites`） | `True` | `False` |
| `real_download` 成功 | 子类 `real_download()` 返回值 | `True` |
| 下载失败 | 子类返回 `False` | `True` |

`success` 在 `process_info` 中的传播路径：

```
# 单文件分支 (L3579)
success, real_download = self.dl(temp_filename, info_dict)

# 多格式逐个下载分支 (L3561)
partial_success, real_download = self.dl(fname, new_info)
success = success and partial_success     ← 任何一个失败则整体失败

# 多格式同时下载分支 (L3526)
success, real_download = self.dl(temp_filename, info_dict)
```

**`success` 决定是否进入 fixup + post_process**（[L3597](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3597)）：`if success and full_filename != '-':`。若 `success=False`，则跳过所有后处理和归档写入——这表示下载确实失败了。

### 9.4 异常在各层级的传播

```
异常发生位置                    │ __download_wrapper    │ _handle_extraction_exceptions
──────────────────────────────│──────────────────────│──────────────────────────────
DownloadError(raise)          │ 未捕获 → 向上抛出     │ 未捕获 → 向上抛出
                              │ (process 终止)        │
──────────────────────────────│──────────────────────│──────────────────────────────
report_error(ignoreerrors=T)  │ — (不抛异常)          │ — (不抛异常)
  → _download_retcode=1       │   返回码已标记失败     │   返回码已标记失败
──────────────────────────────│──────────────────────│──────────────────────────────
UnavailableVideoError         │ report_error(e)       │ —
                              │ → retcode=1 继续      │
──────────────────────────────│──────────────────────│──────────────────────────────
DownloadCancelled             │ to_screen(e)          │ —
  break_per_url=False         │ 重新 raise → 终止     │
  break_per_url=True          │ _num_downloads=0      │
                              │ 继续下一个 URL        │
──────────────────────────────│──────────────────────│──────────────────────────────
CookieLoadError               │ 直接 raise            │ 直接 raise
──────────────────────────────│──────────────────────│──────────────────────────────
ReExtractInfo                 │ —                     │ while True 重试
                              │                       │ (expected=True 不警告)
──────────────────────────────│──────────────────────│──────────────────────────────
network_exceptions            │ —                     │ —
  (in process_info)           │ report_error → retcode=1 + return
──────────────────────────────│──────────────────────│──────────────────────────────
OSError                       │ —                     │ —
  (in process_info)           │ raise UnavailableVideoError
                              │ → 被 __download_wrapper 捕获 → report_error
──────────────────────────────│──────────────────────│──────────────────────────────
ContentTooShortError          │ —                     │ —
  (in process_info)           │ report_error → retcode=1 + return
──────────────────────────────│──────────────────────│──────────────────────────────
PostProcessingError           │ —                     │ —
  (in process_info)           │ report_error → retcode=1 + return
  ignoreerrors=True           │ 或 run_pp 内吞掉     │
──────────────────────────────│──────────────────────│──────────────────────────────
MaxDownloadsReached           │ —                     │ —
  (in process_video_result)   │ 捕获后重新 raise
  (in process_info)           │ check_max_downloads() raise
```

### 9.5 `_raise_pending_errors` 机制

某些 PP（如 `pre_process`）不直接抛异常，而是将错误信息写入 `info_dict['__pending_error']`。在 `process_video_result` 和 `process_info` 中调用 `_raise_pending_errors` 统一检查并抛出。这意味着 PP 失败可以被延迟到安全的检查点再中断。

---

## 10. 归档写入时机详解

### 10.1 `__write_download_archive` 三态

`info_dict['__write_download_archive']` 有三种取值：

| 值 | 含义 | 后果 |
|----|------|------|
| `True` | 应当写入归档 | 在 `process_video_result` 的汇总判断中被计为"成功" |
| `False` | 不写入归档 | 在汇总判断中被计为"失败" |
| `'ignore'` | 被过滤/跳过，不参与归档判断 | 不影响汇总判断 |

### 10.2 所有赋值点

| 赋值点 | 代码位置 | 值 | 触发条件 |
|--------|---------|-----|---------|
| `_match_entry` 过滤命中 | [L3341](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3341) | `'ignore'` | 视频/格式被 `--match-title`/`--date` 等过滤掉 |
| `simulate` 模拟 | [L3371](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3371) | `force_write_download_archive` 的值 | `--simulate` 不下载但可选写归档 |
| `skip_download` 模式 | [L3457](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3457) | `force_write_download_archive` 的值 | `--skip-download` 不下载媒体文件 |
| 下载 + 后处理成功 | [L3667](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3667) | `True` | 完整走完下载 → fixup → post_process → post_hooks |
| `force_write_download_archive` | [L3671](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3671) | `True` | `--force-write-download-archive` 全局覆盖 |
| `extract_flat` 模式 | [L1936](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L1936) | 直接调用 `record_download_archive` | 不展开提取，直接记录 |

### 10.3 三层粒度体系与归档判断

在分析归档逻辑之前，必须首先厘清三层粒度，否则会发生"外层格式组"与"内层子格式"的混淆：

| 粒度层 | 名称 | 数据结构 | 对应代码位置 | 说明 |
|-------|------|---------|------------|------|
| 粒度 1（最外层） | **单个视频**（info_dict） | 整个 `info_dict` | `record_download_archive(info_dict)` | 归档的实际写入单位，键为 `extractor_key + video_id` |
| 粒度 2 | **外层格式组**（formats_to_download 元素） | `fmt`（含 `format_id` 可能为 `"137+140"`） | `process_video_result` 中 `itertools.product(formats_to_download, requested_ranges)` 循环，每个元素一次 `process_info` 调用 | 多格式合并时，一个外层格式组 = 一个"需要合并的集合"（如 video-only + audio-only） |
| 粒度 3（最内层） | **内层子格式**（requested_formats 元素） | 单个 format（如 format_id=`137` 的纯视频、format_id=`140` 的纯音频） | `process_info` 中 `for f in info_dict['requested_formats']` 循环 | 真正独立下载的媒体文件，无独立归档标记 |

**关键区分**：
- `downloaded_formats`（[L3127](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3127)）收集的是 **粒度 2（外层格式组）**，每个元素对应一次 `process_info` 调用
- `__write_download_archive` 只存在于 **粒度 2**，不存在于粒度 3 的子格式
- 归档判断完全在 **粒度 1** 和 **粒度 2** 两层进行，**粒度 3（子格式）没有独立的归档话语权**

### 10.4 多格式归档汇总判断（粒度 1 vs 粒度 2）

在 `process_video_result` 的下载循环结束后（[L3140-L3143](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3140-L3143)）：

```python
write_archive = {f.get('__write_download_archive', False) for f in downloaded_formats}
assert write_archive.issubset({True, False, 'ignore'})
if True in write_archive and False not in write_archive:
    self.record_download_archive(info_dict)
```

**判断逻辑（粒度 2 → 粒度 1）**：收集所有外层格式组（粒度 2）的 `__write_download_archive` 值，仅当：
- 至少有一个 `True`（有外层格式组完整成功）
- 且没有任何 `False`（没有外层格式组失败或被提前 return）

才会在粒度 1 写入归档。`'ignore'` 不阻止写入。

**易混淆点澄清**：
- ❌ 错误理解：`formats_to_download` 中每个元素是单个子格式，子格式部分失败单独标记
- ✅ 正确理解：`formats_to_download` 中每个元素是 **外层格式组**，格式组内部包含多个子格式（由 `_merge()` 构造，含 `requested_formats` 列表）
- ✅ 正确理解：粒度 3 子格式失败会 **连带** 整个外层格式组（粒度 2）的 `__write_download_archive` 变为 `False`（或保持默认的 `False`，因为 `success=False` 导致跳过赋值）

### 10.5 子格式失败边界分析（粒度 3 失败如何传播到粒度 2）

外层格式组（粒度 2）对应的 `process_info` 调用内部，可能存在多个内层子格式（粒度 3）的下载。子格式失败如何影响外层格式组的 `__write_download_archive` 值，取决于失败发生的位置：

#### 场景 A：多格式逐个下载分支（`fd is None`，[L3549-L3563](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3549-L3563)）

```python
success = True
downloaded = []
for f in info_dict['requested_formats']:     # 遍历每个子格式（粒度3）
    new_info = dict(info_dict)
    del new_info['requested_formats']
    new_info.update(f)
    fname = prepend_extension(temp_filename, f'f{f["format_id"]}', new_info['ext'])
    if not self._ensure_dir_exists(fname):
        return                                # ⚠️ 提前 return：__write_download_archive 保持 False
    f['filepath'] = fname
    downloaded.append(fname)
    partial_success, real_download = self.dl(fname, new_info)
    info_dict['__real_download'] = info_dict['__real_download'] or real_download
    success = success and partial_success     # 一票否决：任一子格式失败 → success=False
```

失败边界：
1. **`_ensure_dir_exists` 失败** → 整个 `process_info` **立即 return**，`__write_download_archive` 从未被赋值，保持默认 `False`
2. **某个子格式 `partial_success=False`**：循环会继续执行完所有子格式（不会提前 break），但最终 `success=False`
3. 循环结束后进入 `[L3597](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3597)` 判断：`if success and full_filename != '-'` → 由于 `success=False`，整个 fixup + post_process + `__write_download_archive = True` 赋值块 **被完全跳过**
4. 最终外层格式组的 `__write_download_archive` 值为 `False`（默认值）

**关键结论**：在逐个下载分支，任何一个子格式下载失败（`partial_success=False`）→ 外层格式组 `success=False` → 跳过后处理 → `__write_download_archive` 保持 `False`。**即使部分子格式（如视频轨）已经完整下载到磁盘，也不会在外层被记为"成功"**。

#### 场景 B：非 FFmpeg 下载器一步下载多 URL（[L3519-L3527](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3519-L3527)）

```python
if fd != FFmpegFD and temp_filename != '-':
    for f in info_dict['requested_formats']:
        f['filepath'] = fname = prepend_extension(...)
        downloaded.append(fname)
info_dict['url'] = '\n'.join(f['url'] for f in info_dict['requested_formats'])
success, real_download = self.dl(temp_filename, info_dict)  # 一次调用，多 URL 输入
```

- 下载器内部处理多 URL，返回的 `success` 是整体结果
- `success=False` → 外层格式组整体标记为失败，`__write_download_archive` 保持 `False`
- 子格式级别的部分成功在这个分支无法感知

#### 场景 C：FFmpegFD 一步下载+合并（[L3526](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3526)）

- FFmpegFD 内部 `-map` 合并，无显式子格式概念
- 只有整体 `success`，任何失败都导致外层 `__write_download_archive=False`

#### 场景 D：后处理阶段失败（FFmpegMergerPP 合并失败）

```python
if success and full_filename != '-':
    fixup()           # 注入各种 Fixup PP 到 __postprocessors
    try:
        replace_info_dict(self.post_process(dl_filename, info_dict, files_to_move))
        # post_process 内部按顺序执行：
        #   run_all_pps('post_process') → 含 FFmpegMergerPP（多格式合并）
        #   run_pp(MoveFilesAfterDownloadPP)
        #   run_all_pps('after_move')
    except PostProcessingError as err:
        self.report_error(f'Postprocessing: {err}')
        return        # ⚠️ 提前 return：__write_download_archive 赋值被跳过
    try:
        for ph in self._post_hooks:
            ph(info_dict['filepath'])
    except Exception as err:
        self.report_error(f'post hooks: {err}')
        return        # ⚠️ 提前 return
    info_dict['__write_download_archive'] = True   # ← 只有走到这里才是 True
```

**关键边界**：`__write_download_archive = True` 的赋值在 [L3667](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3667)，位于：
- post_process（含 FFmpegMergerPP 合并）成功之后
- post_hooks 成功之后

因此，**即使所有子格式都下载成功（分文件完整存在于磁盘），只要 FFmpegMergerPP 合并失败**，就会 `return`，`__write_download_archive` 保持 `False`。

### 10.6 重试影响分析：子格式文件存在性与续传机制

#### 10.6.1 重试前的文件状态分类

假设外层格式组 `bestvideo+bestaudio`（format_id=`137+140`），重试前子格式文件有三种状态：

| 子格式文件状态 | 磁盘上存在的文件 | 说明 |
|--------------|----------------|------|
| **完全下载成功** | `video.f137.mp4`（无 `.part`） | 下载器 `try_rename` 已将 `.part` 重命名为最终文件名 |
| **部分下载中断** | `video.f137.mp4.part` | 下载中途失败/中断，`.part` 文件仍保留 |
| **从未下载** | 无 | 第一次失败在下载开始前（如 `_ensure_dir_exists`、网络错误等） |

关键区别：`existing_video_file`（粒度 1）只检查**合并后**的最终文件 `video.mkv` / `video.mp4`，不检查任何子格式文件。因此无论子格式文件处于哪种状态，只要合并文件不存在，就会进入下载流程。

#### 10.6.2 下载器级别的续传检测：`FileDownloader.download()`

位置：[common.py#L430-L455](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/common.py#L430-L455)

```python
def download(self, filename, info_dict, subtitle=False):
    nooverwrites_and_exists = (
        not self.params.get('overwrites', True)        # --no-overwrites
        and os.path.exists(filename)                   # filename = 最终文件名（非 .part）
    )
    if not hasattr(filename, 'write'):
        continuedl_and_exists = (
            self.params.get('continuedl', True)        # 默认开启续传
            and os.path.isfile(filename)               # ⚠️ 检查的是最终文件名
            and not self.params.get('nopart', False)   # 未禁用 .part
        )
    if filename != '-' and (nooverwrites_and_exists or continuedl_and_exists):
        self.report_file_already_downloaded(filename)
        return True, False   # success=True, real_download=False
```

**核心发现**：`continuedl_and_exists` 检查的是 `os.path.isfile(filename)`——即**最终文件名**（如 `video.f137.mp4`），而不是 `.part` 临时文件。这意味着：

- **子格式文件完全下载成功**（从 `.part` 已 rename）→ `os.path.isfile('video.f137.mp4')` 为 True → `continuedl` 命中 → **跳过，返回 `(True, False)`** → 零重试成本
- **子格式文件部分下载**（仅 `.part` 存在）→ `os.path.isfile('video.f137.mp4')` 为 False → 不命中 → 进入 `real_download` → **由具体下载器的续传逻辑处理**
- **子格式文件从未下载** → 同上，从头下载

#### 10.6.3 HttpFD 的续传实现

位置：[http.py#L24-L376](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/http.py#L24-L376)

HttpFD 内部维护双层文件名：

| 变量 | 值 | 用途 |
|-----|---|------|
| `ctx.filename` | `video.f137.mp4` | 最终目标文件名 |
| `ctx.tmpfilename` | `video.f137.mp4.part` | 临时下载文件（由 `self.temp_name(filename)` 生成） |

续传流程：

```
real_download(filename='video.f137.mp4', info_dict)
│
├─ ctx.tmpfilename = self.temp_name(filename) = 'video.f137.mp4.part'
│
├─ [continuedl=True] 且 os.path.isfile(ctx.tmpfilename):
│   ├─ ctx.resume_len = os.path.getsize(ctx.tmpfilename)  ← 读取 .part 文件大小
│   └─ ctx.is_resume = True
│
├─ establish_connection():
│   ├─ [ctx.resume_len > 0] → 发送 Range: bytes=resume_len- 请求
│   │   ├─ 服务器返回 Content-Range 且起始匹配 → ctx.open_mode = 'ab'（追加写入）
│   │   ├─ 服务器返回 416 (Range Not Satisfiable):
│   │   │   ├─ 无 Range 重试后 Content-Length ≈ resume_len (±100字节) → 视为已下载完
│   │   │   │   → try_rename(.part → 最终名) → raise SucceedDownload → return True
│   │   │   └─ Content-Length 不匹配 → report_unable_to_resume → 从头下载 (open_mode='wb')
│   │   └─ 服务器不返回 Content-Range → 无法续传 → 从头下载 (open_mode='wb')
│   └─ [ctx.resume_len == 0] → 正常下载 (open_mode='wb')
│
├─ download(): 循环读取数据写入 ctx.tmpfilename
│
└─ 成功后: self.try_rename(ctx.tmpfilename, ctx.filename)
    → .part 文件重命名为最终文件名
```

**关键行为**：
- `.part` 文件存在时，HttpFD 用 HTTP Range 头续传，**只下载剩余部分**
- 服务器不支持 Range 时，放弃续传从头下载（覆盖 `.part`）
- 416 + Content-Length ≈ 已有大小 → 视为已下载完成，直接 rename

#### 10.6.4 FragmentFD（DASH/HLS native）的分片续传

位置：[fragment.py#L26-L319](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/fragment.py#L26-L319)

FragmentFD 使用 **`.ytdl` 簿记文件** 记录下载进度，实现分片级续传：

```
video.f137.mp4           ← 合并后的最终文件（成功后才 rename）
video.f137.mp4.part      ← 临时合并文件（分片逐步追加写入）
video.f137.mp4.part.ytdl ← 簿记文件（JSON，记录 fragment_index）
video.f137.mp4.part-Frag0  ← 单个分片临时文件（下载完即合并后删除）
video.f137.mp4.part-Frag1
...
```

续传流程（`_prepare_frag_download`，[L157-L223](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/fragment.py#L157-L223)）：

```
_prepare_frag_download(ctx)
│
├─ tmpfilename = self.temp_name(ctx['filename']) = 'video.f137.mp4.part'
├─ resume_len = self.filesize_or_none(tmpfilename)  ← .part 文件大小
│
├─ [__do_ytdl_file(ctx)]:
│   ├─ ytdl_file_exists = os.path.isfile('video.f137.mp4.part.ytdl')
│   │
│   ├─ [continuedl=True 且 ytdl_file_exists]:
│   │   ├─ _read_ytdl_file(ctx) → 读取 fragment_index
│   │   ├─ 一致性检查: fragment_index > 0 但 resume_len == 0 → 不一致，从头开始
│   │   └─ ytdl_corrupt → 从头开始
│   │
│   └─ [continuedl=False]:
│       └─ fragment_index = resume_len = 0 → 从头开始
│
├─ [resume_len > 0] → open_mode = 'ab'（追加写入 .part）
│
└─ dest_stream = sanitize_open(tmpfilename, open_mode)
    → 从 fragment_index 处继续下载分片，追加到 .part 文件
```

**分片级续传**：
- 单个分片也支持续传：`_download_fragment` 中检查 `frag_resume_len = self.filesize_or_none(self.temp_name(fragment_filename))`（[L120-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/fragment.py#L120-L122)）
- 成功完成后删除 `.ytdl` 簿记文件，`try_rename(.part → 最终名)`

#### 10.6.5 外部下载器的续传行为

位置：[external.py#L42-L70](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/external.py#L42-L70)

外部下载器的 `real_download` 接收 `filename`（最终文件名），内部用 `self.temp_name(filename)` 得到 `.part` 文件名，将 `.part` 传给 `_call_downloader`。成功后 `try_rename(.part → 最终名)`。

| 下载器 | 续传标志 | 行为 |
|--------|---------|------|
| **curl** | `--continue-at -`（`continuedl=True` 时） | 自动续传 `.part` 文件 |
| **aria2c** | `-c`（硬编码始终开启） | 始终尝试续传 `.part` 文件 |
| **wget** | 无显式续传标志 | wget 默认对已有文件尝试续传 |
| **FFmpegFD** | 无续传 | 不支持续传，每次从头下载 |

**注意**：外部下载器的续传发生在 `.part` 文件级别。如果子格式已完全下载（`.part` 已 rename 为最终名），则 `FileDownloader.download()` 的 `continuedl_and_exists` 检查会先命中，直接返回 `(True, False)`，根本不会调用 `real_download`。

#### 10.6.6 完整重试场景表（含续传与复用细节）

以下表格基于 `continuedl=True`（默认开启续传）和 `overwrites=True`（默认允许覆盖）：

| 失败位置 | 子格式 137 磁盘状态 | 子格式 140 磁盘状态 | 合并文件 | 归档 | 重试时子格式 137 的处理 | 重试时子格式 140 的处理 | 重试成本 |
|---------|-------------------|-------------------|---------|------|----------------------|----------------------|---------|
| `_ensure_dir_exists` | 无 | 无 | 无 | 否 | 从头下载 | 从头下载 | **全量** |
| 第 1 个子格式下载失败（.part 不存在） | 无 | 未尝试 | 无 | 否 | 从头下载 | 从头下载 | **全量** |
| 第 1 个子格式下载中断（.part 存在） | `video.f137.mp4.part` | 未尝试 | 无 | 否 | HttpFD/FragmentFD 续传 `.part` | 从头下载 | **部分** |
| 第 2 个子格式下载失败 | `video.f137.mp4`（完整） | 无 | 无 | 否 | **`continuedl` 跳过**，返回 `(True, False)` | 从头下载 | **低**（仅音频） |
| 第 2 个子格式下载中断 | `video.f137.mp4`（完整） | `video.f140.m4a.part` | 无 | 否 | **`continuedl` 跳过** | HttpFD/FragmentFD 续传 `.part` | **最低**（续传音频） |
| 全部下载成功，FFmpegMergerPP 合并失败 | `video.f137.mp4`（完整） | `video.f140.m4a`（完整） | 无 | 否 | **`continuedl` 跳过** | **`continuedl` 跳过** | **最低**（仅重试合并） |
| FFmpegMergerPP 成功，post_hooks 失败 | — | — | `video.mkv`（完整） | 否 | **`existing_video_file` 检测到** → 跳过整个下载 | 同左 | **仅后处理** |
| 全部成功 | — | — | `video.mkv` | 是 | **归档命中** → 整个视频跳过 | 同左 | **零** |

#### 10.6.7 复用/覆盖条件对重试成本的影响

`overwrites` 和 `continuedl` 两个参数共同决定重试行为：

| 参数组合 | 子格式完整文件存在时 | 子格式 .part 文件存在时 | 合并文件存在时 |
|---------|-------------------|----------------------|-------------|
| `overwrites=True, continuedl=True`（默认） | `continuedl` 跳过，零成本 | 续传 `.part`，部分成本 | `existing_video_file` 跳过，零成本 |
| `overwrites=True, continuedl=False` | **不跳过**，从头覆盖 | **不续传**，从头覆盖 | `existing_video_file` 跳过，零成本 |
| `overwrites=False, continuedl=True` | `nooverwrites` 跳过，零成本 | 续传 `.part`，部分成本 | `existing_video_file` 跳过（`nooverwrites` 更优先），零成本 |
| `overwrites=False, continuedl=False` | `nooverwrites` 跳过，零成本 | **不续传**，从头覆盖 | `existing_video_file` 跳过，零成本 |

**关键差异**：
- `continuedl=True`（默认）是子格式级别复用的关键开关——只有开启时，已完整下载的子格式文件才会被跳过，部分下载的 `.part` 文件才会被续传
- `continuedl=False` 时，子格式文件即使已完整存在也会被重新下载（但合并文件仍被 `existing_video_file` 保护）
- `overwrites=False`（`--no-overwrites`）比 `continuedl` 更严格：只要文件存在就跳过，不区分完整/部分

#### 10.6.8 `existing_file` 对已存在分文件的删除行为

`existing_file` 方法（[L3320-L3328](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3320-L3328)）：

```python
def existing_file(self, filepaths, *, default_overwrite=True):
    existing_files = list(filter(os.path.exists, orderedSet(filepaths)))
    if existing_files and not self.params.get('overwrites', default_overwrite):
        return existing_files[0]
    for file in existing_files:
        self.report_file_delete(file)
        os.remove(file)
    return None
```

`existing_video_file` 只传了 `full_filename` 和 `temp_filename`（合并后的文件），**没有传分文件路径**。因此：
- `overwrites=True`（默认）时：分文件不在检测列表中，不会被主动删除，但下载器写入同名 `.part` 文件时可能覆盖
- `overwrites=False` 时：分文件同样不在检测列表中，不会被主动删除也不会被主动复用

**总结**：子格式级别的部分成功在重试时**可以被利用**，具体机制取决于文件状态和参数：

1. **已完整下载的子格式文件**（无 `.part`）：由 `FileDownloader.download()` 的 `continuedl_and_exists` 检查自动跳过 → 零重试成本
2. **部分下载的 `.part` 文件**：由具体下载器（HttpFD/FragmentFD/外部下载器）的续传逻辑处理 → 部分重试成本
3. **从未下载的子格式**：从头下载 → 全量成本
4. 归档粒度是视频级（粒度 1），不是子格式级（粒度 3），但 `continuedl` 在下载器层面提供了子格式级（粒度 3）的隐式复用能力

### 10.7 `record_download_archive` 实现

位置：[L3876-L3887](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3876-L3887)

```python
def record_download_archive(self, info_dict):
    fn = self.params.get('download_archive')
    if fn is None:
        return
    vid_id = self._make_archive_id(info_dict)
    assert vid_id
    if is_path_like(fn):
        with locked_file(fn, 'a', encoding='utf-8') as archive_file:
            archive_file.write(vid_id + '\n')
    self.archive.add(vid_id)
```

- 同时写入文件（`locked_file` 防并发）和内存集合 `self.archive`
- `vid_id` 由 `_make_archive_id(extractor_key, video_id)` 生成
- 下次 `extract_info` 时 `in_download_archive` 检查内存集合，命中则跳过

---

## 11. 多格式合并在不同下载器和 FFmpeg 条件下的分支逻辑

### 11.1 前置：`get_suitable_downloader` 的决策树

位置：[downloader/__init__.py#L4-L20](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/__init__.py#L4-L20)

当 `requested_formats` 存在时，`info_dict['protocol']` 为各格式协议用 `+` 连接（如 `https+m3u8_native`）。`get_suitable_downloader` 对每个子协议分别匹配下载器，然后做合并判断：

```
protocols = info_dict['protocol'].split('+')    # 如 ['https', 'm3u8_native']
downloaders = [为每个 proto 选下载器]

if 所有子协议都选了 FFmpegFD and FFmpegFD.can_merge_formats():
    → 返回 FFmpegFD（一步到位：ffmpeg 同时下载+合并）
elif 所有子协议都是 DashSegmentsFD 且满足条件:
    → 返回 DashSegmentsFD
elif 只有一个子协议:
    → 返回该下载器
else:
    → 返回 None ← 关键！无合适下载器可一步完成
```

### 11.2 `FFmpegFD.can_merge_formats` 条件

位置：[downloader/external.py#L387-L393](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/external.py#L387-L393)

```python
@classmethod
def can_merge_formats(cls, info_dict, params):
    return (
        info_dict.get('requested_formats')            # 有多格式
        and info_dict.get('protocol')                  # 有协议信息
        and not params.get('allow_unplayable_formats') # 不允许不可播放格式
        and 'no-direct-merge' not in params.get('compat_opts', [])  # 无兼容选项
        and cls.can_download(info_dict)                # ffmpeg 可用 + 协议支持
    )
```

`cls.can_download` 依赖 `cls.supports`（[L106-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/downloader/external.py#L106-L112)），需满足：
- ffmpeg 已安装（`available`）
- 不输出到 stdout 或 stdout 是 `SUPPORTED_FEATURES`
- 协议含 `+` 时需 `MULTIPLE_FORMATS` 特性（FFmpegFD 有此特性）
- 不含 HLS AES 加密参数
- 所有子协议都在 `SUPPORTED_PROTOCOLS` 内

### 11.3 `process_info` 中多格式合并的三大分支

当 `requested_formats` 不为 None 时（[L3482-L3572](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3482-L3572)），根据 `fd`（`get_suitable_downloader` 返回值）和 `merger.available`（FFmpegMergerPP 是否可用）进入不同分支：

#### 分支 1：`fd` 存在（`get_suitable_downloader` 返回了非 None 下载器）

```
if fd is FFmpegFD:   ← 所有子协议都是 FFmpegFD 可处理的
    # ffmpeg 一步完成多格式下载（内含合并逻辑）
    # 不需要预分文件
    # _call_downloader 内部处理 -map 等合并参数

if fd is not FFmpegFD and temp_filename != '-':  ← 非ffmpeg下载器，输出到文件
    for f in requested_formats:
        f['filepath'] = prepend_extension(temp_filename, 'f{format_id}', ext)
        downloaded.append(fname)           # 记录分文件路径
    info_dict['url'] = '\n'.join(各格式URL) # 拼接所有URL
    success, real_download = self.dl(...)  # 用该下载器一次性下载

# 之后判断是否注入 merger：
if downloaded and merger.available and not allow_unplayable_formats:
    __postprocessors.append(merger)
    __files_to_merge = downloaded
    __real_download = True
else:
    for file in downloaded:
        files_to_move[file] = None   # 不合并，分文件各自移动到最终目录
```

#### 分支 2：`fd` 为 None（`get_suitable_downloader` 无法一步完成）+ 无 ffmpeg

```
if not merger.available:      ← ffmpeg 未安装
    if not ignoreerrors:
        report_error → return       # 中止
    report_warning → 继续但不合并

# 逐个格式独立下载
for f in requested_formats:
    new_info = dict(info_dict)      # 复制，去掉 requested_formats
    new_info.update(f)
    fname = prepend_extension(temp_filename, 'f{format_id}', ext)
    f['filepath'] = fname
    downloaded.append(fname)
    partial_success, real_download = self.dl(fname, new_info)
    success = success and partial_success

# 同样判断 merger 可用性
```

#### 分支 3：`fd` 为 None + 输出到 stdout (`temp_filename == '-'`)

```
# 逐个格式流式输出（无法合并到 stdout）
for f in requested_formats:
    fname = '-'                     # stdout
    partial_success, real_download = self.dl(fname, new_info)
    success = success and partial_success

# 无法合并到 stdout，仅给出警告
```

### 11.4 完整决策流程图

```
requested_formats 不为 None
│
├─ 扩展名预处理
│   ├─ merge_output_format 未指定 + webm + EmbedThumbnailPP → 改用 mkv
│   └─ correct_ext 统一文件扩展名
│
├─ existing_video_file 检查（已下载则跳过）
│
├─ fd = get_suitable_downloader(info_dict, params)
│
├─ [fd is not None]
│   │
│   ├─ [fd is FFmpegFD] ─── ffmpeg 一步下载+合并
│   │   └─ self.dl(temp_filename, info_dict)
│   │       → FFmpegFD._call_downloader 内部 -map 合并
│   │       → 返回 (success, real_download)
│   │
│   └─ [fd is not FFmpegFD, temp_filename != '-']
│       ├─ 为每个格式计算分文件路径 f{format_id}.ext
│       ├─ 拼接所有URL: info_dict['url'] = '\n'.join(...)
│       └─ self.dl(temp_filename, info_dict)
│           → 下载器处理多URL输入
│
├─ [fd is None]
│   │
│   ├─ [allow_unplayable_formats] → 警告：不合并以防数据损坏
│   ├─ [not merger.available] → 警告/错误：ffmpeg 未安装
│   │
│   ├─ [temp_filename == '-']
│   │   ├─ FFmpegFD.can_merge_formats → "using a downloader other than ffmpeg"
│   │   ├─ merger.available → "formats are incompatible for simultaneous download"
│   │   └─ else → "ffmpeg is not installed"
│   │   → 逐个格式流式输出到 stdout
│   │
│   └─ [temp_filename != '-']
│       └─ 逐个格式独立下载到分文件
│
├─ 合并判断（通用）
│   ├─ downloaded 非空 + merger.available + not allow_unplayable_formats
│   │   → 注入 FFmpegMergerPP 到 __postprocessors
│   │   → __files_to_merge = downloaded
│   │   → __real_download = True
│   │
│   └─ 否则
│       → files_to_move[file] = None（分文件各自移动）
│
└─ 后续进入 fixup + post_process
    → FFmpegMergerPP.run() 执行 ffmpeg -c copy -map 合并
       输入：info['__files_to_merge']（分文件列表）
       输出：info['filepath']（合并后文件）
       返回：需删除的文件列表 = __files_to_merge
```

### 11.5 FFmpegMergerPP 合并实现

位置：[postprocessor/ffmpeg.py#L822-L847](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L822-L847)

```python
class FFmpegMergerPP(FFmpegPostProcessor):
    def run(self, info):
        filename = info['filepath']
        temp_filename = prepend_extension(filename, 'temp')
        args = ['-c', 'copy']
        for (i, fmt) in enumerate(info['requested_formats']):
            if fmt.get('acodec') != 'none':
                args.extend(['-map', f'{i}:a:0'])
                # m3u8 + aac 需要比特流滤镜修复
                if fmt['protocol'].startswith('m3u8') and self.get_audio_codec(fmt['filepath']) == 'aac':
                    args.extend([f'-bsf:a:{audio_streams}', 'aac_adtstoasc'])
            if fmt.get('vcodec') != 'none':
                args.extend(['-map', f'{i}:v:0'])
        self.run_ffmpeg_multiple_files(info['__files_to_merge'], temp_filename, args)
        os.rename(temp_filename, filename)
        return info['__files_to_merge'], info
```

要点：
- 使用 `-c copy` 不重新编码，仅容器层合并
- `-map` 精确映射每个输入文件的音频/视频流
- 输出先写临时文件，`os.rename` 原子替换
- 返回的 `info['__files_to_merge']` 作为待删除文件列表，由 `run_pp` 根据 `keepvideo` 决定是否删除

### 11.6 各条件组合速查表

| 条件组合 | 下载方式 | 合并方式 | 归档写入（粒度 1） |
|---------|---------|---------|-------------------|
| fd=FFmpegFD, merger 可用 | ffmpeg 一步下载+合并 | ffmpeg 内部 `-map` | 外层格式组粒度 2 成功 → 粒度 1 写入归档 |
| fd≠FFmpegFD, merger 可用, 输出到文件 | 非ffmpeg下载器一步下载 | FFmpegMergerPP 后处理合并 | 外层格式组粒度 2 的下载+合并均成功 → 粒度 1 写入；任何子格式失败或合并失败 → 不写入 |
| fd=None, merger 可用, 输出到文件 | 逐子格式（粒度 3）独立下载 | FFmpegMergerPP 后处理合并 | **粒度 3 部分成功不会被单独计为成功**；所有子格式全部下载成功 + 合并成功 → 粒度 2 标记 True → 粒度 1 写入；任一子格式下载失败 → 粒度 2 保持 False → 不写入 |
| fd=None, merger 不可用, ignoreerrors=True | 逐子格式（粒度 3）独立下载 | **不合并**，分文件各自保留 | 不合并仍走 post_process → 若所有子格式下载均成功（粒度 2 的 `success=True`）→ 粒度 2 标记 True → 粒度 1 写入；任一子格式失败 → 不写入 |
| fd=None, merger 不可用, ignoreerrors=False | 报错中止 | — | — |
| 任何, 输出到 stdout (`-`) | 逐格式流式输出 | **不可合并** | 输出到 stdout 时跳过 `__write_download_archive=True` 赋值（`full_filename != '-'` 条件不满足）→ 粒度 2 保持 False → 不写入 |
| 任何, `allow_unplayable_formats=True` | 视情况 | **强制不合并**（防数据损坏） | 不合并仍走 post_process；下载成功 → 粒度 2 标记 True → 粒度 1 写入 |
