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
| `info_dict['__write_download_archive']` | 是否写入归档（True/False/'ignore'） | process_info 各分支 |

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

### 10.3 多格式归档汇总判断

在 `process_video_result` 的下载循环结束后（[L3140-L3143](file:///d:/fz/0601-2/solo-dogfeeding/code/83-yt-dlp/yt_dlp/YoutubeDL.py#L3140-L3143)）：

```python
write_archive = {f.get('__write_download_archive', False) for f in downloaded_formats}
assert write_archive.issubset({True, False, 'ignore'})
if True in write_archive and False not in write_archive:
    self.record_download_archive(info_dict)
```

**判断逻辑**：收集所有格式（每个 `process_info` 调用一个）的 `__write_download_archive` 值，仅当：
- 至少有一个 `True`（有格式成功下载完成）
- 且没有任何 `False`（没有格式失败或被提前 return）

才会写入归档。`'ignore'` 不阻止写入。

**这意味着**：如果一个视频选择了 2 个格式，格式 1 下载成功（`True`）但格式 2 下载失败（`False`），则 **不会** 写入归档——重试时两个格式都会重新下载。

### 10.4 `record_download_archive` 实现

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

| 条件组合 | 下载方式 | 合并方式 | 归档写入 |
|---------|---------|---------|---------|
| fd=FFmpegFD, merger 可用 | ffmpeg 一步下载+合并 | ffmpeg 内部 `-map` | 下载成功后 `__write_download_archive=True` |
| fd≠FFmpegFD, merger 可用, 输出到文件 | 非ffmpeg下载器一步下载 | FFmpegMergerPP 后处理合并 | 合并成功后写入 |
| fd=None, merger 可用, 输出到文件 | 逐格式独立下载 | FFmpegMergerPP 后处理合并 | 合并成功后写入 |
| fd=None, merger 不可用, ignoreerrors=True | 逐格式独立下载 | **不合并**，分文件各自保留 | 部分成功可写入（取决于各格式 `__write_download_archive`） |
| fd=None, merger 不可用, ignoreerrors=False | 报错中止 | — | — |
| 任何, 输出到 stdout (`-`) | 逐格式流式输出 | **不可合并** | 取决于各格式成功与否 |
| 任何, `allow_unplayable_formats=True` | 视情况 | **强制不合并**（防数据损坏） | 下载成功可写入 |
