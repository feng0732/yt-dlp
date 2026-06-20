# yt-dlp 章节与缩略图嵌入代码协作分析

## 一、整体架构概览

yt-dlp 中章节（Chapters）与缩略图（Thumbnails）的处理贯穿了 **元数据收集 → 文件下载 → 后处理嵌入** 三个主要阶段。两者看似独立，但在执行顺序、数据共享、后处理链上存在紧密协作关系。

### 核心后处理器清单

| 后处理器类 | 源码位置 | 核心职责 |
|-----------|---------|---------|
| `ModifyChaptersPP` | `yt_dlp/postprocessor/modify_chapters.py` | 移除/切割章节，处理 SponsorBlock 章节合并 |
| `FFmpegMetadataPP` | `yt_dlp/postprocessor/ffmpeg.py#L662-L820` | 嵌入元数据（含章节信息）到视频容器 |
| `FFmpegSplitChaptersPP` | `yt_dlp/postprocessor/ffmpeg.py#L1015-L1059` | 按章节分割视频为独立文件 |
| `EmbedThumbnailPP` | `yt_dlp/postprocessor/embedthumbnail.py` | 将缩略图嵌入到媒体文件 |
| `FFmpegThumbnailsConvertorPP` | `yt_dlp/postprocessor/ffmpeg.py#L1062-L1128` | 缩略图格式转换（WebP→PNG/JPG 等） |
| `SponsorBlockPP` | `yt_dlp/postprocessor/sponsorblock.py` | 从 SponsorBlock API 获取赞助分段 |

---

## 二、元数据收集阶段

### 2.1 缩略图元数据收集

**入口：** InfoExtractor 提取 → `_sanitize_thumbnails` 清洗

#### 提取流程：

1. **Extractor 原始数据**：各站点提取器返回 `thumbnail`（单张）或 `thumbnails`（多张，含 id/width/height/preference/url）
2. **标准化处理** — `yt_dlp/YoutubeDL.py#L2731-L2761`：

```python
def _sanitize_thumbnails(self, info_dict):
    # 1. 单张 thumbnail 转为 thumbnails 列表
    thumbnails = info_dict.get('thumbnails')
    if thumbnails is None:
        thumbnail = info_dict.get('thumbnail')
        if thumbnail:
            info_dict['thumbnails'] = thumbnails = [{'url': thumbnail}]

    # 2. 按优先级排序: preference → width → height → id → url
    self._sort_thumbnails(thumbnails)

    # 3. 补全 id、resolution、url 清洗
    for i, t in enumerate(thumbnails):
        if t.get('id') is None:
            t['id'] = str(i)
        if t.get('width') and t.get('height'):
            t['resolution'] = '%dx%d' % (t['width'], t['height'])
        t['url'] = sanitize_url(t['url'])

    # 4. 可选: 检测缩略图 URL 连通性 (--check-formats)
```

调用时机：
- `process_video_result()` 中：`yt_dlp/YoutubeDL.py#L2887`
- 播放列表提取阶段：`yt_dlp/YoutubeDL.py#L2009`

### 2.2 章节元数据收集

**入口：** InfoExtractor 提取 → `process_video_result` 修正 → 后处理器 `_fixup_chapters` 补全 → **SponsorBlockPP after_filter 阶段注入**

#### 章节数据结构：
```python
chapters = [
    {
        'start_time': 0.0,      # 起始秒数
        'end_time': 120.5,      # 结束秒数
        'title': 'Introduction' # 章节标题
    },
    ...
]
```

#### 收集与修正流程：

1. **Extractor 返回**：各站点提取器返回 `chapters` 列表
2. **自动补全首尾** — `yt_dlp/YoutubeDL.py#L2868-L2874`：
   - 若第一章无 `start_time=0`，自动插入起始章节
   - 遍历检查 `start_time`/`end_time` 连续性
3. **SponsorBlock 分段注入（after_filter 阶段）**：
   - 由 `SponsorBlockPP` 在 **after_filter** 阶段调用 SponsorBlock API
   - 结果存于独立字段 `info['sponsorblock_chapters']`（**不直接写入 chapters**）
   - 源码：`yt_dlp/postprocessor/sponsorblock.py#L39-L47`
4. **后处理阶段补全** — `_fixup_chapters()`：`yt_dlp/postprocessor/ffmpeg.py#L298-L301`
   - 若最后一章缺失 `end_time`，通过 ffprobe 读取实际视频时长填充

---

## 三、文件处理阶段（下载 & 写入）

### 3.1 缩略图下载写入

**时机：** 视频下载之前，在 `process_info()` 中执行
**调用入口：** `yt_dlp/YoutubeDL.py#L3391-L3395` → `_write_thumbnails()`

#### 核心流程 `_write_thumbnails` — `yt_dlp/YoutubeDL.py#L4496-L4547`：

```python
def _write_thumbnails(self, label, info_dict, filename, thumb_filename_base=None):
    # 1. 受参数控制: writethumbnail / write_all_thumbnails
    # 2. 逆序遍历 thumbnails（优先高质量）
    for idx, t in list(enumerate(thumbnails))[::-1]:
        # 3. 检查文件是否已存在
        existing_thumb = self.existing_file((thumb_filename_final, thumb_filename))
        if existing_thumb:
            t['filepath'] = existing_thumb
            continue
        # 4. HTTP 下载缩略图
        uf = self.urlopen(Request(t['url'], headers=t.get('http_headers', {})))
        # 5. 写入本地文件, 将路径写回 t['filepath']
        t['filepath'] = thumb_filename
```

**关键点：** 下载后缩略图路径被写回 `info_dict['thumbnails'][i]['filepath']`，供后续 `EmbedThumbnailPP` 读取。

### 3.2 章节数据在下载阶段的使用：`--download-sections` 参数详解

**实际仅支持的 CLI 选项：`--download-sections REGEX`**（注意：无 `--download-chapter` 独立选项）

源码解析入口：`yt_dlp/__init__.py#L350-L392`，调用 `parse_chapters()` + `download_range_func()`

该参数支持 **两种语义模式**，通过首字符 `*` 区分，可多次混用：

| 模式 | 语法特征 | 示例 | 行为 |
|-----|---------|------|------|
| **章节标题正则** | 不以 `*` 开头 | `--download-sections "Intro"` | 匹配 `chapters` 列表中 title 满足该正则的章节 |
| **时间段** | 以 `*` 开头 | `--download-sections "*00:15-02:30"` | 按秒数精确下载，与 chapters 无关 |

#### 时间段语法（`*start-end`）完整能力：

```
*start-end        常规时间戳，支持 10:15 / 02:30:00 / 纯秒数
*start-           end 省略 = inf（下载到结束）
*-end             start 省略 = 0
*-30              start 为负 = 从倒数 30 秒开始
*10--15           从第 10 秒下载到倒数第 15 秒
*10:15-inf        inf = 无限（实际到视频尾）
*0-30,1:00-1:30   同一条参数内用逗号分隔多个时间段
*from-url         从 URL 中提取 start_time / end_time（仅当 info_dict 有此字段）
```

#### 实现流程：

1. **CLI → 内部结构**（`parse_chapters()` — `yt_dlp/__init__.py#L350-L389`）：
   - 遍历用户传入的每条 `--download-sections` 参数
   - 非 `*` 开头：编译为正则对象，加入 `chapters` 列表
   - `*` 开头：解析为 `[start_seconds, end_seconds]` 元组，加入 `ranges` 列表
   - `*from-url`：单独标记 `from_url=True`

2. **构造回调** — `download_range_func(chapters, ranges, from_url=True)`：
   - 源码：`yt_dlp/utils/_utils.py#L3350-L3389`
   - 该回调在 `process_video_result()` 中被调用，返回实际的 `requested_ranges`

3. **下载阶段消费** — `yt_dlp/YoutubeDL.py#L3098-L3126`：
   - `requested_ranges = download_range_func(info_dict, ydl)` 回调执行
     - 遍历正则匹配 chapters 中的标题，返回 `{start_time, end_time, title, index}`
     - 遍历 ranges 列表中的时间元组（含负数时间戳处理）
     - 若 from_url 且 info_dict 含 start_time/end_time，则使用该段
   - 对 `formats_to_download × requested_ranges` 笛卡尔积
   - 每条产生独立的 `new_info`，带 `section_start` / `section_end` / `section_title` / `section_number`
   - 逐次传入 `process_info(new_info)` 实际下载

---

## 四、后处理触发点与调用链

### 4.1 后处理器注册机制

**注册入口：** `get_postprocessors(opts)` — `yt_dlp/__init__.py#L627-L736`

该函数根据 CLI 选项生成后处理器配置列表，按 **严格顺序** yield，顺序是协作正确性的核心：

```python
def get_postprocessors(opts):
    # SponsorBlock（仅注册有配置时）
    # - when='after_filter'：必须早于下载，才能让 ModifyChapters 读取到 sponsorblock_chapters
    if sponsorblock_query:
        yield {
            'key': 'SponsorBlock',
            'categories': sponsorblock_query,  # mark + remove 的并集
            'api': opts.sponsorblock_api,
            'when': 'after_filter',
        }

    # 1. 缩略图格式转换 (before_dl) - 先于视频下载执行
    if opts.convertthumbnails:
        yield {'key': 'FFmpegThumbnailsConvertor', 'format': ..., 'when': 'before_dl'}

    # 2. 字幕嵌入 (必须在 ModifyChapters 之前，切割时需同步切割字幕)
    if opts.embedsubtitles:
        yield {'key': 'FFmpegEmbedSubtitle', ...}

    # 3. 章节修改/移除 (必须在 FFmpegMetadata 之前)
    if opts.remove_chapters or sponsorblock_query:
        yield {
            'key': 'ModifyChapters',
            'remove_chapters_patterns': opts.remove_chapters,     # 正则匹配 chapters 标题
            'remove_sponsor_segments': opts.sponsorblock_remove,  # 移除的 SponsorBlock 分类
            'remove_ranges': opts.remove_ranges,                  # 手动时间段 (*xx-yy 解析结果)
            'sponsorblock_chapter_title': opts.sponsorblock_chapter_title,
            'force_keyframes': opts.force_keyframes_at_cuts,
        }

    # 4. 元数据嵌入（含章节信息）—— 必须在章节修改之后，嵌入的是最终版本
    if opts.addmetadata or opts.addchapters or opts.embed_infojson:
        yield {
            'key': 'FFmpegMetadata',
            'add_chapters': opts.addchapters,
            'add_metadata': opts.addmetadata,
            'add_infojson': opts.embed_infojson,
        }

    # 5. 缩略图嵌入
    if opts.embedthumbnail:
        yield {
            'key': 'EmbedThumbnail',
            'already_have_thumbnail': opts.writethumbnail,
        }
        # 自动开启 writethumbnail 以确保缩略图文件存在（EmbedThumbnailPP 需要磁盘文件）
        if not opts.writethumbnail:
            opts.writethumbnail = True

    # 6. 按章节分割视频（最后：因会产生多个输出文件，后续 PP 无法再处理）
    if opts.split_chapters:
        yield {'key': 'FFmpegSplitChapters', 'force_keyframes': ...}
```

**后处理器实例化：** `yt_dlp/YoutubeDL.py#L827-L837`

```python
for pp_def_raw in self.params.get('postprocessors', []):
    pp_def = dict(pp_def_raw)
    when = pp_def.pop('when', 'post_process')
    self.add_post_processor(
        get_postprocessor(pp_def.pop('key'))(self, **pp_def),
        when=when)
```

### 4.2 SponsorBlock 两阶段协作：after_filter 取数据 → post_process 消费

这是章节处理中最不直观的跨阶段协作，由两个后处理器接力完成：

```
process_video_result()
  │
  ├─ pre_process('pre_process')
  ├─ post_extract()
  ├─ pre_process('after_filter')          ← ★ 阶段一：SponsorBlockPP 执行
  │    └─ SponsorBlockPP.run(info)
  │         ├─ 仅支持 Youtube 等 EXTRACTORS 中登记的提取器
  │         ├─ SHA256(videoId) → 调用 /api/skipSegments
  │         ├─ 过滤 duration 匹配的分段（避免旧版视频）
  │         ├─ 转为 {start_time, end_time, category, title,
  │         │           _categories: [(cat,s,e,name), ...]} 结构
  │         └─ 写入 info['sponsorblock_chapters'] = [...]
  │                                              ↑
  │                                              │ 字段分离，暂不影响原 chapters
  │
  ├─ 格式选择 → requested_ranges 展开
  ├─ process_info() 批量执行下载
  │
  └─ post_process()
       └─ run_all_pps('post_process')
            └─ ModifyChaptersPP.run(info)    ← ★ 阶段二：消费两段数据
                 ├─ _mark_chapters_to_remove(
                 │      info['chapters'],           # 原站章节 → 按 remove_chapters_patterns 正则标记
                 │      info['sponsorblock_chapters']  # SponsorBlock 分段 → 按 remove_sponsor_segments 分类标记
                 │  )
                 │     若仅 --sponsorblock-mark 而不 remove：
                 │       这些 sponsorblock_chapters 不会被标 remove，
                 │       而是在 _remove_marked_arrange_sponsors 中与普通章节
                 │       重叠合并，最终进入 info['chapters']
                 ├─ _remove_marked_arrange_sponsors(合并后列表)
                 │    └─ heapq 处理 8 种重叠情形，切割正常章节
                 ├─ 更新 info['chapters']（只剩保留的章节，时间轴已重排）
                 └─ ffmpeg 执行切割 + 同步切割已嵌入字幕
```

**SponsorBlock 分类集合关系** — `yt_dlp/postprocessor/sponsorblock.py#L14-L32`：

```
CATEGORIES（所有可查询分类）
├─ 可 skip 的分类：sponsor / intro / outro / selfpromo / preview / filler / interaction / music_offtopic / hook
└─ NON_SKIPPABLE_CATEGORIES（不可用于 remove，仅用于 mark 生成章节）
    ├─ POI_CATEGORIES：poi_highlight（Highlights）
    └─ chapter（直接以 description 为标题的用户自定义章节）
```

**源码证据**（`remove_sponsor_segments` 过滤非跳过分类）— `yt_dlp/postprocessor/modify_chapters.py#L19`：
```python
self._remove_sponsor_segments = (
    set(remove_sponsor_segments or [])
    - set(SponsorBlockPP.NON_SKIPPABLE_CATEGORIES.keys())
)
```

### 4.3 后处理执行链

**总入口：** `post_process()` — `yt_dlp/YoutubeDL.py#L3839-L3846`

```python
def post_process(self, filename, info, files_to_move=None):
    info['filepath'] = filename
    info['__files_to_move'] = files_to_move or {}

    # 第一阶段: post_process（主后处理，含动态追加的 __postprocessors）
    info = self.run_all_pps('post_process', info, additional_pps=info.get('__postprocessors'))

    # 第二阶段: 移动文件到最终目录
    info = self.run_pp(MoveFilesAfterDownloadPP(self), info)
    del info['__files_to_move']

    # 第三阶段: after_move（文件已就位，如写 xattr、执行外部命令）
    return self.run_all_pps('after_move', info)
```

**执行时序图（简化）：**

```
process_video_result()
    │
    ├─ pre_process('pre_process')       # when='pre_process'
    ├─ _match_entry() / post_extract()
    ├─ pre_process('after_filter')      # when='after_filter' ★ SponsorBlockPP
    │
    ├─ 格式选择 / __forced_printings
    └─ 对每个 fmt × range：
         process_info(new_info)
             │
             ├─ pre_process('video')    # when='video'（MetadataParser 等）
             ├─ _write_subtitles()
             ├─ _write_thumbnails()     # 缩略图写入磁盘
             ├─ _write_info_json()
             ├─ pre_process('before_dl')# when='before_dl' ★ ThumbnailsConvertor
             │
             ├─ dl()                    # 视频下载
             ├─ fixup()                 # 动态追加 MergerPP / FixupPP → __postprocessors
             │
             └─ post_process()
                  ├─ run_all_pps('post_process')
                  │   ├─ __postprocessors（MergerPP, Fixup*PP 等）
                  │   ├─ ModifyChaptersPP          ← 先切割视频
                  │   ├─ FFmpegMetadataPP          ← 再嵌入章节+元数据
                  │   ├─ EmbedThumbnailPP          ← 再嵌入缩略图
                  │   └─ FFmpegSplitChaptersPP     ← 最后按章节分割（多文件输出）
                  ├─ MoveFilesAfterDownloadPP
                  └─ run_all_pps('after_move')

所有视频下载完成后：
    └─ run_all_pps('after_video', info)  # when='after_video'
    └─ （playlist 级别）run_all_pps('playlist', info)  # when='playlist' ★ FFmpegConcat
```

### 4.4 阶段划分与 when 参数完整表

`POSTPROCESS_WHEN` 完整定义 — `yt_dlp/utils/_utils.py#L2858`：

```python
POSTPROCESS_WHEN = ('pre_process', 'after_filter', 'video', 'before_dl',
                    'post_process', 'after_move', 'after_video', 'playlist')
```

| when 值 | 触发时机 | 典型后处理器 / 用途 |
|---------|---------|-------------------|
| `pre_process` | `process_video_result` 最开头，格式选择之前 | （预留扩展点） |
| `after_filter` | `post_extract()` 之后，格式筛选/列表打印之前 | **SponsorBlockPP**（提前获取分段） |
| `video` | `process_info()` 中，任何文件写入之前 | MetadataParserPP（解析标题生成元数据） |
| `before_dl` | 缩略图/字幕/infojson 已写入，视频下载之前 | FFmpegThumbnailsConvertorPP、FFmpegSubtitlesConvertorPP |
| `post_process` | 视频下载完成后 | ModifyChaptersPP、FFmpegMetadataPP、EmbedThumbnailPP、FFmpegSplitChaptersPP 等 |
| `after_move` | 文件已移动到最终输出目录 | XAttrMetadataPP、ExecPP（`--exec` after_move 变体） |
| `after_video` | 该视频所有分段（section/range）均已下载完成 | （插件扩展点） |
| `playlist` | 整个播放列表所有视频均已下载完成 | FFmpegConcat（拼接播放列表视频） |

---

## 五、章节相关代码精确执行顺序

### 5.1 章节数据在 info_dict 中的三阶段生命周期

章节数据在 info_dict 中经历 **收集 → 下载筛选 → 后处理修改 → 嵌入** 四步，每一步的代码位置、数据字段和语义都不同：

```
① process_video_result() 中收集与补全
   │
   │  yt_dlp/YoutubeDL.py#L2868-L2880
   │  ├─ chapters[0] 无 start_time=0 → insert 起始章
   │  ├─ 遍历三元组 (prev, current, next_) 补全缺失的 start_time/end_time/title
   │  └─ 此时 info_dict['chapters'] 已就绪（原站数据，SponsorBlock 尚未介入）
   │
② pre_process('after_filter') — SponsorBlock 分段注入
   │
   │  yt_dlp/YoutubeDL.py#L3040
   │  └─ SponsorBlockPP.run() → info_dict['sponsorblock_chapters'] = [...]
   │     ★ 注意：写入的是独立字段 'sponsorblock_chapters'，
   │       不修改 'chapters'，此时 download_range_func 还未执行
   │
③ download_range_func() — 下载前章节筛选（只读 chapters）
   │
   │  yt_dlp/YoutubeDL.py#L3098
   │  ├─ 遍历 info_dict.get('chapters') → 正则匹配标题 → yield 匹配的章节段
   │  ├─ 遍历 self.ranges → yield 时间段（与 chapters 无关）
   │  ├─ from_url → yield info_dict 中 start_time/end_time（与 chapters 无关）
   │  └─ ★ 不读 sponsorblock_chapters！
   │     --download-sections "Sponsor" 不会匹配 SponsorBlock 分段
   │     （因 SponsorBlock 数据在另一个字段）
   │
④ post_process() — 下载后章节修改（读+写 chapters 和 sponsorblock_chapters）
   │
   │  ModifyChaptersPP.run() — yt_dlp/postprocessor/modify_chapters.py#L25-L75
   │  ├─ _fixup_chapters(info)       ← 第1次调用：补全 end_time（基于 ffprobe）
   │  ├─ 读取 info['chapters'] + info['sponsorblock_chapters']
   │  ├─ 合并两段数据 → 重写 info['chapters'] 和 info['duration']
   │  └─ 实际用 ffmpeg 切割视频文件
   │
   │  FFmpegMetadataPP.run() — yt_dlp/postprocessor/ffmpeg.py#L678-L709
   │  ├─ _fixup_chapters(info)       ← 第2次调用：再次补全（ModifyChapters 已更新 chapters）
   │  └─ 将 info['chapters'] 写入 .meta 文件 → ffmpeg 嵌入
   │
   │  FFmpegSplitChaptersPP.run() — yt_dlp/postprocessor/ffmpeg.py#L1043-L1059
   │  ├─ _fixup_chapters(info)       ← 第3次调用：再次补全
   │  └─ 按 info['chapters'] 分割视频 → 产生多个文件
```

### 5.2 下载前章节筛选 vs 下载后章节修改：本质区别

| 维度 | 下载前筛选 (`--download-sections`) | 下载后修改 (`--remove-chapters` / SponsorBlock) |
|-----|----------------------------------|-----------------------------------------------|
| **执行阶段** | `process_video_result` 中 `download_range_func` 回调 | `post_process` 中 `ModifyChaptersPP` |
| **视频文件状态** | 尚未下载，视频不存在 | 已在磁盘，文件已下载完成 |
| **数据来源** | 仅读 `info_dict['chapters']` | 读 `info_dict['chapters']` + `info_dict['sponsorblock_chapters']` |
| **对 chapters 的影响** | **只读不写**：筛选结果存入 `section_start/end`，chapters 不变 | **读写**：重写 `info_dict['chapters']` 和 `info_dict['duration']` |
| **SponsorBlock 可见性** | ❌ 不可见（`sponsorblock_chapters` 不被读取） | ✅ 可见（两个字段均被消费） |
| **输出文件数** | 每段独立下载为单独文件（fmt × range 笛卡尔积） | 切割后原文件保留为 `.uncut`，输出为单个拼接文件 |
| **需要 ffmpeg** | 下载时需要（设置 section_start/section_end 后 ffmpeg 裁剪） | 后处理时需要（concat demuxer 拼接保留段） |
| **重复执行** | 每个 range 单独触发 `process_info()` | 只执行一次 |

**关键设计差异**：`download_range_func` 不读 `sponsorblock_chapters` 是有意为之——下载前筛选仅基于视频元数据中已有的章节信息，而 SponsorBlock 数据虽然已在 after_filter 阶段注入，但与下载范围选择属于不同的关注点。

### 5.3 `_fixup_chapters` 的三次调用与语义差异

`_fixup_chapters` 定义于 `yt_dlp/postprocessor/ffmpeg.py#L298-L301`，逻辑统一：若最后一章缺 `end_time`，用 ffprobe 读取视频文件实际时长填充。但在三个 PP 中的调用语义不同：

| 调用位置 | 代码行 | 此时 info['chapters'] 的状态 | 补全的目的 |
|---------|-------|--------------------------|----------|
| `ModifyChaptersPP.run()` | `modify_chapters.py#L26` | 可能已被 SponsorBlock 合并、但最后一章 end_time 可能为 None | 确保切割前时间轴完整，否则 `_make_concat_opts` 会算错 inpoint/outpoint |
| `FFmpegMetadataPP.run()` | `ffmpeg.py#L679` | ModifyChaptersPP 已重写 chapters，但若 ModifyChapters 未执行（用户未配置移除），end_time 仍可能缺失 | 确保写入 .meta 文件的章节范围覆盖完整视频 |
| `FFmpegSplitChaptersPP.run()` | `ffmpeg.py#L1044` | 同上，或 ModifyChapters 后已补全 | 防御性补全：确保分割时每章有完整时间范围 |

**为什么需要多次调用**：`_fixup_chapters` 是幂等操作，多次调用无副作用。不同 PP 的执行是条件组合的（用户可能只开 `--embed-chapters` 而不配 `--remove-chapters`），每个 PP 必须独立保证数据的完整性。

### 5.4 SponsorBlock 分段进入 info_dict 的精确时机

```
process_video_result() 完整调用链（yt_dlp/YoutubeDL.py）

  L3034  info_dict, _ = self.pre_process(info_dict)          ← when='pre_process'
  L3036  if self._match_entry(...) is not None: return        ← 匹配过滤

  L3039  self.post_extract(info_dict)                         ← 惰性提取器执行

  L3040  info_dict, _ = self.pre_process(info_dict, 'after_filter')  ← ★ 此处执行 SponsorBlockPP
         │
         │  SponsorBlockPP.run(info) 内部流程：
         │    1. 检查 info['extractor_key'] 是否在 EXTRACTORS（目前仅 Youtube）
         │    2. SHA256(info['id']) → 取前 4 字符 → /api/skipSegments/{hash}
         │    3. 按 duration 过滤匹配的分段（避免旧版视频数据）
         │    4. 转换为 [{start_time, end_time, category, title, _categories}] 结构
         │    5. 写入 info['sponsorblock_chapters'] = [...]     ← 数据注入点
         │    6. 返回 [], info（无文件删除）
         │
         │  ★ 此时 sponsorblock_chapters 已在 info_dict 中，
         │    但 download_range_func 尚未执行

  L3043  formats = self._get_formats(info_dict)               ← 格式可能被 PP 修改

  L3098  requested_ranges = download_range_func(info_dict, ydl)
         │  ★ 此函数只读 info_dict['chapters']，不读 sponsorblock_chapters

  L3112  for fmt, chapter in itertools.product(formats_to_download, requested_ranges):
           ... process_info(new_info) ...                      ← 实际下载
```

### 5.5 FFmpegSplitChaptersPP 之后仍有后处理器执行

**`get_postprocessors()` 的完整 yield 顺序**（`yt_dlp/__init__.py#L627-L736`）：

```
  SponsorBlockPP         when='after_filter'
  FFmpegThumbnailsConvertorPP  when='before_dl'
  FFmpegSubtitlesConvertorPP   when='before_dl'
  FFmpegEmbedSubtitlePP        when='post_process'  ← 默认
  ModifyChaptersPP             when='post_process'
  FFmpegMetadataPP             when='post_process'
  EmbedThumbnailPP             when='post_process'
  FFmpegSplitChaptersPP        when='post_process'
  XAttrMetadataPP              when='post_process'  ← ★ 在 SplitChapters 之后！
  FFmpegConcatPP               when='playlist'
  ExecPP                       when=用户指定（默认 after_move）
```

**`XAttrMetadataPP` 在 `FFmpegSplitChaptersPP` 之后执行**，这是有意为之——注释原文（`yt_dlp/__init__.py#L721`）：

> XAttrMetadataPP should be run after post-processors that may change file contents

**但存在一个语义缺陷**：`FFmpegSplitChaptersPP.run()` 返回 `[], info`，**不修改** `info['filepath']`（仍指向原始完整视频文件）。因此：
- `XAttrMetadataPP` 会将 xattr 写到**原始未分割文件**上，而不是分割后的各个章节文件
- `ExecPP(when='post_process')` 同理，`info['filepath']` 指向原始文件
- 分割产生的章节文件路径仅存在于 `chapter['filepath']` 中（`FFmpegSplitChaptersPP._ffmpeg_args_for_chapter` 赋值），但**不回写到 info_dict 的顶层**

这意味着 `--split-chapters` 与 `--xattrs` 同时使用时，xattr 只作用于分割前的完整文件，而非各章节片段。这是当前代码的已知行为。

**ExecPP 的 when 灵活性**：`--exec` 默认 `when='after_move'`，但用户可显式指定 `--exec post_process:CMD` 使其在 post_process 阶段执行。无论哪种，ExecPP 注释声明"must be the last PP of each category"——在 `get_postprocessors` 中它总是某个 when 类别的最后一个 yield。

---

## 六、章节与缩略图的协作细节

### 6.1 执行顺序的依赖关系

在 `get_postprocessors()` 中，后处理器按严格顺序 yield，原因如下：

1. **ModifyChaptersPP → FFmpegMetadataPP**
   若先嵌入章节再移除，移除后的章节信息不会更新到已嵌入的元数据中。故必须先切割移除，再嵌入最终章节。

2. **ModifyChaptersPP 依赖字幕已嵌入**
   切割视频时需同步切割字幕时间轴，字幕必须已在容器内 — `yt_dlp/postprocessor/modify_chapters.py#L112-L123`。
   因此 `FFmpegEmbedSubtitlePP` 的 yield 在 ModifyChaptersPP 之前。

3. **FFmpegSplitChaptersPP 须最后执行**
   分割会生成多个独立文件，一旦分割，后续 PP 无法再对整部视频操作。故放在 post_process 队列末尾。

4. **EmbedThumbnailPP 与格式的隐式协作**
   当输出格式为 webm 且用户启用了缩略图嵌入时，系统自动降级为 mkv（因为 webm 容器不支持嵌入缩略图）— `yt_dlp/YoutubeDL.py#L3485-L3492`：

```python
if (info_dict['ext'] == 'webm'
        and info_dict.get('thumbnails')
        and any(type(pp) == EmbedThumbnailPP for pp in self._pps['post_process'])):
    info_dict['ext'] = 'mkv'
    self.report_warning('webm doesn\'t support embedding a thumbnail, mkv will be used')
```

5. **--embed-thumbnail 自动开启 --write-thumbnail**
   `EmbedThumbnailPP` 依赖磁盘上的缩略图文件。若用户仅指定 `--embed-thumbnail` 而未指定 `--write-thumbnail`，`get_postprocessors()` 会强制设 `opts.writethumbnail = True`，确保 `_write_thumbnails()` 被调用 — `yt_dlp/__init__.py#L707-L715`。

### 6.2 数据共享：info_dict 的传递

所有后处理器通过同一个 `info_dict` 字典共享数据，关键字段：

| 字段 | 写入者 | 读取者 |
|------|--------|--------|
| `chapters` | Extractor → ModifyChaptersPP（切割后重写） | FFmpegMetadataPP、FFmpegSplitChaptersPP、`download_range_func` |
| `sponsorblock_chapters` | SponsorBlockPP（after_filter） | ModifyChaptersPP（post_process） |
| `thumbnails[i].filepath` | `_write_thumbnails()`（process_info） | EmbedThumbnailPP、FFmpegThumbnailsConvertorPP |
| `filepath` | YoutubeDL 下载完成后 | 所有后处理器（输入媒体文件路径） |
| `__files_to_move` | process_info（缩略图/字幕/infojson 路径） | MoveFilesAfterDownloadPP |
| `__postprocessors` | fixup()、Merger 分支（动态追加） | `post_process()` 作为 `additional_pps` 传入 |
| `infojson_filename` | `_write_info_json()`（process_info） | FFmpegMetadataPP `_get_infojson_opts()` |

### 6.3 缩略图内部协作：EmbedThumbnailPP ↔ FFmpegThumbnailsConvertorPP

`EmbedThumbnailPP` 在执行时内部会实例化 `FFmpegThumbnailsConvertorPP` 处理格式兼容问题 — `yt_dlp/postprocessor/embedthumbnail.py#L75-L85`：

```python
convertor = FFmpegThumbnailsConvertorPP(self._downloader)
convertor.fixup_webp(info, idx)  # 修正 WebP 文件扩展名错误

# 非 MKV 容器仅支持 jpg/png, 需先转换为 PNG 再嵌入
if info['ext'] not in ('mkv', 'mka') and thumbnail_ext not in ('jpg', 'jpeg', 'png'):
    thumbnail_filename = convertor.convert_thumbnail(thumbnail_filename, 'png')
```

注意：**两级转换** — 虽然 `--convert-thumbnails` 已在 `before_dl` 阶段执行过 `FFmpegThumbnailsConvertorPP.run()` 全局转换，但 EmbedThumbnailPP 内部仍会再次做 **容器感知** 的针对性转换（用户可能未设置 `--convert-thumbnails`）。

### 6.4 章节内部协作：共享 `_fixup_chapters`

`ModifyChaptersPP`、`FFmpegMetadataPP`、`FFmpegSplitChaptersPP` 均继承自 `FFmpegPostProcessor`，共享父类的 `_fixup_chapters` 方法（`yt_dlp/postprocessor/ffmpeg.py#L298-L301`），确保在各处理器运行前最后一章 `end_time` 已通过 ffprobe 补全：

```python
def _fixup_chapters(self, info):
    last_chapter = traverse_obj(info, ('chapters', -1))
    if last_chapter and not last_chapter.get('end_time'):
        last_chapter['end_time'] = self._get_real_video_duration(info['filepath'])
```

---

## 七、核心后处理器深度解析

### 7.1 FFmpegMetadataPP — 章节与元数据嵌入

**源码位置：** `yt_dlp/postprocessor/ffmpeg.py#L662-L820`

#### 章节嵌入流程：

```python
def run(self, info):
    self._fixup_chapters(info)  # 补全缺失的 end_time

    # 1. 生成 FFMETADATA1 格式的章节元数据文件（仅 add_chapters=True 且有章节）
    if self._add_chapters and info.get('chapters'):
        metadata_filename = replace_extension(filename, 'meta')
        options.extend(self._get_chapter_opts(info['chapters'], metadata_filename))

    # 2. 生成通用元数据选项 (-metadata title=... / -metadata:s:v language=... 等)
    if self._add_metadata:
        options.extend(self._get_metadata_opts(info))

    # 3. 将 info.json 作为附件嵌入 (仅 MKV/MKA 支持)
    if self._add_infojson and info['ext'] in ('mkv', 'mka'):
        options.extend(self._get_infojson_opts(info, infojson_filename))

    # 4. 调用 ffmpeg: ffmpeg -i input -i metadata.meta -map_metadata 1 ... output
    self.run_ffmpeg_multiple_files(
        (filename, metadata_filename), temp_filename,
        itertools.chain(self._options(info['ext']), *options))
```

**章节文件格式**（`_get_chapter_opts` 生成的 .meta 文件）：
```
;FFMETADATA1
[CHAPTER]
TIMEBASE=1/1000
START=0
END=120500
title=Introduction
```

### 7.2 ModifyChaptersPP — 章节移除/切割

**源码位置：** `yt_dlp/postprocessor/modify_chapters.py`

#### 核心算法：

```
run()
 ├─ _fixup_chapters()                           # 补全章节 end_time
 ├─ _mark_chapters_to_remove(chapters,          # 按正则标记普通章节
 │                           sponsor_chapters)  # 按 category 标记 SponsorBlock 分段
 │     ├─ 同时登记 self._ranges_to_remove（手动时间段）为 remove 条目
 │     └─ ⚠️ remove_sponsor_segments 已排除 NON_SKIPPABLE_CATEGORIES
 │
 ├─ _remove_marked_arrange_sponsors(chapters + sponsor_chapters)
 │    └─ 构建最小堆 heapq（按 start_time 排序）处理 8 种重叠情形：
 │        - (cut, cut) 相邻移除范围合并
 │        - (cut, normal/sponsor) 切割后者的开头
 │        - (normal, cut) 切割前者结尾 / 若 cut 完全在内部则登记 cut_idx
 │        - (sponsor, normal) / (normal, sponsor) 相互切割
 │        - (sponsor, sponsor) 重叠区域分类合并
 │        → 最终输出 new_chapters（时间轴已重排，移除段消失）
 │        → 同时输出 cuts（用于 ffmpeg 实际切割视频）
 │
 ├─ _remove_tiny_rename_sponsors()              # 去除 <1s 的微小章节 + 重命名 SponsorBlock 章节
 ├─ _make_concat_opts()                         # 生成 inpoint/outpoint 列表
 └─ remove_chapters()                           # 对视频 + 已嵌入字幕文件调用 ffmpeg concat demuxer 执行切割
```

**协作点：** 修改后直接更新 `info['chapters']` 和 `info['duration']`，供后续 `FFmpegMetadataPP` 读取最新章节数据。

### 7.3 EmbedThumbnailPP — 缩略图嵌入

**源码位置：** `yt_dlp/postprocessor/embedthumbnail.py`

#### 按格式选择嵌入策略（三级降级）：

| 输出格式 | 嵌入工具（优先级从高到低） | 机制 |
|---------|----------------------|------|
| **mp3** | 仅 ffmpeg | 以视频流方式附加封面，设置 ID3v1 + ID3v2.3 |
| **mkv/mka** | 仅 ffmpeg | `-attach` 附件方式，设置 mimetype=image/jpeg，filename=cover.jpg |
| **m4a/mp4/m4v/mov** | ① mutagen → ② AtomicParsley → ③ ffmpeg+ffprobe | 写入 covr atom（三级工具降级） |
| **ogg/opus/flac** | 仅 mutagen | FLAC 用 `add_picture()`；Ogg 用 base64 编码写入 `METADATA_BLOCK_PICTURE` 标签 |

---

## 八、触发条件速查表

| CLI 选项 | 触发的后处理器 / 机制 | when / 执行阶段 |
|---------|---------------------|----------------|
| `--embed-metadata` / `--add-metadata` | FFmpegMetadataPP(add_metadata=True) | post_process |
| `--embed-chapters` / `--add-chapters` | FFmpegMetadataPP(add_chapters=True) | post_process |
| `--embed-info-json` | FFmpegMetadataPP(add_infojson=True) | post_process |
| `--embed-thumbnail` | EmbedThumbnailPP（**自动开启 writethumbnail**） | post_process |
| `--write-thumbnail` | process_info 中 `_write_thumbnails()` | 视频下载前 |
| `--convert-thumbnails FORMAT` | FFmpegThumbnailsConvertorPP | before_dl |
| `--split-chapters` | FFmpegSplitChaptersPP | post_process（队列末尾） |
| `--remove-chapters REGEX\|*xx-yy` | ModifyChaptersPP（正则匹配标题 或 时间段） | post_process |
| `--download-sections REGEX\|*xx-yy\|*from-url` | `download_range_func` 回调展开为多段下载 | process_video_result 中 |
| `--sponsorblock-mark CATS` | SponsorBlockPP（获取分段）+ ModifyChaptersPP（合并为章节） | after_filter → post_process |
| `--sponsorblock-remove CATS` | SponsorBlockPP（获取分段）+ ModifyChaptersPP（标记 remove） | after_filter → post_process |
| `--force-keyframes-at-cuts` | ModifyChaptersPP / FFmpegSplitChaptersPP 切割前先强制关键帧 | 切割前执行 |
| `--xattrs` | XAttrMetadataPP（在 SplitChapters 之后执行） | post_process |
| `--exec [WHEN:]CMD` | ExecPP（默认 when=after_move，可指定任意阶段） | 用户指定 / after_move |

---

## 九、总结

章节与缩略图的协作是 yt-dlp 后处理系统设计的典型体现：

1. **解耦但有序**：各后处理器职责单一，但通过 `get_postprocessors()` 的严格 yield 顺序保证正确性
2. **数据驱动**：全部通过 `info_dict` 共享状态，字段语义清晰（如 `sponsorblock_chapters` 与 `chapters` 分离）
3. **多阶段注入**：通过 8 个 `when` 阶段精确控制执行时机，典型如 SponsorBlockPP 的 `after_filter` 提前取数
4. **动态扩展**：支持 `__postprocessors` 动态追加（MergerPP、FixupPP 等）、两级格式转换（全局 before_dl + 容器感知内联转换）
5. **容错降级**：缩略图嵌入的三级工具降级、章节/缩略图缺失时的优雅跳过
6. **隐式协作**：
   - webm+embed-thumbnail → 自动降容器为 mkv
   - embed-thumbnail 无 write-thumbnail → 自动补开
   - ModifyChapters 后 → FFmpegMetadata 重新嵌入已更新章节
   - --sponsorblock-mark/remove 的跨阶段双 PP 接力
7. **已知行为边界**：
   - `_fixup_chapters` 三次幂等调用，因 PP 条件组合各自独立保证完整性
   - `download_range_func` 不读 `sponsorblock_chapters`，--download-sections 无法匹配 SponsorBlock 分段
   - `FFmpegSplitChaptersPP` 不回写 `info['filepath']`，后续 XAttrPP/ExecPP 作用于原始文件而非章节片段
