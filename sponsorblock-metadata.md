# SponsorBlock 与元数据后处理关系分析

## 1. 整体架构概览

SponsorBlock 功能在 yt-dlp 中通过 **三个核心后处理器（PostProcessor）** 的协作完成：

1. **SponsorBlockPP** — 从 SponsorBlock API 获取片段标记，写入 `info` 字典
2. **ModifyChaptersPP** — 消费 SponsorBlock 标记，执行章节合并/裁剪/删除
3. **FFmpegMetadataPP** — 将最终处理后的章节信息嵌入视频文件元数据

它们通过 **后处理链（PostProcessor Chain）** 按特定顺序连接，执行时机分布在不同的 `POSTPROCESS_WHEN` 阶段。

---

## 2. 后处理链连接点

### 2.1 阶段定义：POSTPROCESS_WHEN

后处理链的执行时机由 8 个阶段组成，定义在 [yt_dlp/utils/_utils.py](yt_dlp/utils/_utils.py#L2858-L2858)：

```python
POSTPROCESS_WHEN = (
    'pre_process',    # 视频信息提取后
    'after_filter',   # 视频通过筛选后
    'video',          # --format 之后，--print/--output 之前
    'before_dl',      # 下载前
    'post_process',   # 下载后（默认）
    'after_move',     # 文件移动到最终位置后
    'after_video',    # 所有格式下载处理完成后
    'playlist',       # 播放列表结束时
)
```

### 2.2 链的构建：get_postprocessors()

所有后处理器在 [yt_dlp/__init__.py](yt_dlp/__init__.py#L627-L736) 的 `get_postprocessors()` 函数中按 yield 顺序构建。每个后处理器通过 `add_post_processor()` 追加到 `self._pps[when] 列表末尾（[YoutubeDL.py](yt_dlp/YoutubeDL.py#L942-L946)），因此同阶段内的执行顺序与 yield 顺序完全一致。

### 2.2.1 后处理器总表

| 后处理器 | when 阶段 | 说明 |
|---------|----------|------|
| 用户自定义PP | 用户指定 | 通过 `--use-postprocessor` 添加（最先） |
| MetadataParserPP | 用户指定（默认 pre_process） | 元数据解析/替换 |
| **SponsorBlockPP** | **after_filter** | 获取 SponsorBlock 片段 |
| FFmpegSubtitlesConvertorPP | before_dl | 字幕格式转换 |
| FFmpegThumbnailsConvertorPP | before_dl | 缩略图格式转换 |
| FFmpegExtractAudioPP | post_process | 音频提取 |
| FFmpegVideoRemuxerPP | post_process | 视频封装转换 |
| FFmpegVideoConvertorPP | post_process | 视频编码转换 |
| FFmpegEmbedSubtitlePP | post_process | 字幕嵌入 |
| **ModifyChaptersPP** | **post_process** | 章节裁剪（必须在 FFmpegMetadataPP 之前） |
| **FFmpegMetadataPP** | **post_process** | 元数据/章节嵌入 |
| EmbedThumbnailPP | post_process | 缩略图嵌入 |
| FFmpegSplitChaptersPP | post_process | 按章节切分 |
| XAttrMetadataPP | post_process | xattr 属性写入 |
| FFmpegConcatPP | playlist | 播放列表合并 |
| ExecPP | 各阶段 | 外部命令执行（每阶段最后） |

### 2.2.2 post_process 阶段精确顺序

当用户通过 `--parse-metadata "post_process:..."` 或 `--replace-in-metadata "post_process:..."` 指定 MetadataParserPP 在 `post_process` 阶段运行时，该阶段内的完整执行顺序（按 [yt_dlp/__init__.py](yt_dlp/__init__.py#L627-L736) 的 yield 顺序）：

| 序号 | 后处理器 | 对 info 的主要修改 |
|-----|---------|------------------|
| 1 | **MetadataParserPP** | 改写 `info['title']`, `info['artist']`, `info['meta_xxx']` 等顶层字段 |
| 2 | FFmpegExtractAudioPP | 提取音频，修改 `filepath`, `ext` 等 |
| 3 | FFmpegVideoRemuxerPP | 视频重封装 |
| 4 | FFmpegVideoConvertorPP | 视频转码 |
| 5 | FFmpegEmbedSubtitlePP | 字幕嵌入容器 |
| 6 | **ModifyChaptersPP** | 改写 `info['chapters']`（合并/删除/重命名章节），剪切视频文件 |
| 7 | **FFmpegMetadataPP** | 读取 `info['chapters']` 和通用元数据，写入文件 |
| 8 | EmbedThumbnailPP | 缩略图嵌入 |
| 9 | FFmpegSplitChaptersPP | 按章节切分文件 |
| 10 | XAttrMetadataPP | 写入 xattr 属性 |
| 11 | ExecPP | 执行外部命令 |

**关键三者相对顺序：MetadataParserPP → ModifyChaptersPP → FFmpegMetadataPP**

关键代码（第 636-706 行）：

```python
sponsorblock_query = opts.sponsorblock_mark | opts.sponsorblock_remove
if sponsorblock_query:
    yield {
        'key': 'SponsorBlock',
        'categories': sponsorblock_query,
        'api': opts.sponsorblock_api,
        'when': 'after_filter',   # <-- 在筛选后立即获取
    }

# ...

# ModifyChapters must run before FFmpegMetadataPP
if opts.remove_chapters or sponsorblock_query:
    yield {
        'key': 'ModifyChapters',
        'remove_chapters_patterns': opts.remove_chapters,
        'remove_sponsor_segments': opts.sponsorblock_remove,
        'remove_ranges': opts.remove_ranges,
        'sponsorblock_chapter_title': opts.sponsorblock_chapter_title,
        'force_keyframes': opts.force_keyframes_at_cuts,
    }

# FFmpegMetadataPP should be run after FFmpegVideoConvertorPP and FFmpegExtractAudioPP
if opts.addmetadata or opts.addchapters or opts.embed_infojson:
    yield {
        'key': 'FFmpegMetadata',
        'add_chapters': opts.addchapters,
        'add_metadata': opts.addmetadata,
        'add_infojson': opts.embed_infojson,
    }
```

### 2.3 链的执行：run_all_pps() / run_pp()

链的执行逻辑在 [yt_dlp/YoutubeDL.py](yt_dlp/YoutubeDL.py#L3798-L3846)：

```python
def run_pp(self, pp, infodict):
    # 调用单个后处理器的 run() 方法
    files_to_delete, infodict = pp.run(infodict)
    # 处理删除/保留临时文件
    return infodict

def run_all_pps(self, key, info, *, additional_pps=None):
    # additional_pps 是 info['__postprocessors'] 中动态追加的（如 FFmpegMergerPP）
    for pp in (additional_pps or []) + self._pps[key]:
        info = self.run_pp(pp, info)
    return info

def post_process(self, filename, info, files_to_move=None):
    info['filepath'] = filename
    info = self.run_all_pps('post_process', info, additional_pps=info.get('__postprocessors'))
    info = self.run_pp(MoveFilesAfterDownloadPP(self), info)
    return self.run_all_pps('after_move', info)
```

`_pps` 在 [YoutubeDL.__init__](yt_dlp/YoutubeDL.py#L640-L640) 中按阶段初始化：

```python
self._pps = {k: [] for k in POSTPROCESS_WHEN}
```

### 2.4 各阶段实际调用顺序

各阶段在 [YoutubeDL.py](yt_dlp/YoutubeDL.py) 中的实际调用时机和顺序：

| 阶段 | 调用位置 | 触发时机 |
|-----|---------|---------|
| `pre_process` | [process_video_result](yt_dlp/YoutubeDL.py#L3034-L3034) | 提取器返回 info 后，格式筛选前 |
| `after_filter` | [process_video_result](yt_dlp/YoutubeDL.py#L3040-L3040) | 格式筛选通过后，格式选择前 |
| `video` | [process_video_result](yt_dlp/YoutubeDL.py#L3354-L3354) | 格式选择后，文件名准备前 |
| `before_dl` | [process_video_result](yt_dlp/YoutubeDL.py#L3449-L3449) | 实际下载前 |
| `post_process` | [post_process](yt_dlp/YoutubeDL.py#L3843-L3843) | 下载完成后 |
| `after_move` | [post_process](yt_dlp/YoutubeDL.py#L3846-L3846) | 文件移动到最终目录后 |
| `after_video` | process_video_result | 所有格式处理完成后 |
| `playlist` | process_ie_result | 播放列表处理完成后 |

**关键调用链（单视频）：**

```
提取器返回 info_dict
    ↓
pre_process(info_dict)   # 阶段 1
    ↓
_match_entry() 筛选
    ↓
post_extract(info_dict)
    ↓
pre_process(info_dict, 'after_filter')   # 阶段 2
    ↓
格式选择、文件名准备
    ↓
pre_process(info_dict, 'video')   # 阶段 3
    ↓
pre_process(info_dict, 'before_dl')   # 阶段 4
    ↓
下载视频文件
    ↓
post_process(filename, info_dict)
    ├─ run_all_pps('post_process')   # 阶段 5
    ├─ run_pp(MoveFilesAfterDownloadPP)
    └─ run_all_pps('after_move')   # 阶段 6
```

---

## 3. 选项触发链路：从 CLI 到后处理器

SponsorBlock 选项不是直接启用单个后处理器，而是通过 **选项依赖 → 后处理器构建** 的两级触发机制。

### 3.1 选项依赖关系

在 [yt_dlp/__init__.py](yt_dlp/__init__.py#L590-L592) 的 `validate_options()` 中：

```python
if (opts.addmetadata or opts.sponsorblock_mark) and opts.addchapters is None:
    # Add chapters when adding metadata or marking sponsors
    opts.addchapters = True
```

**关键逻辑：`addchapters` 的默认值是 `None`（不是 False）**，这是一个三态设计：
- `None` — 未设置，由其他选项自动决定
- `True` — 显式启用（`--embed-chapters`）
- `False` — 显式禁用（`--no-embed-chapters`）

自动开启的条件是：`addmetadata 或 sponsorblock_mark 为真` **且** `addchapters 为 None`。

此外，在 [yt_dlp/options.py](yt_dlp/options.py) 中，`addchapters` 选项的定义：

```python
# --embed-chapters / --add-chapters
'--embed-chapters', '--add-chapters', action='store_true', dest='addchapters', default=None

# --no-embed-chapters / --no-add-chapters
'--no-embed-chapters', '--no-add-chapters', action='store_false', dest='addchapters'
```

### 3.2 后处理器启用条件汇总

| 后处理器 | 启用条件 | 配置来源 |
|---------|---------|---------|
| **SponsorBlockPP** | `sponsorblock_query` 非空 | `opts.sponsorblock_mark \| opts.sponsorblock_remove` |
| **ModifyChaptersPP** | `remove_chapters` 非空 **或** `sponsorblock_query` 非空 | 章节删除 + SponsorBlock 标记/删除 |
| **FFmpegMetadataPP** | `addmetadata` **或** `addchapters` **或** `embed_infojson` | 元数据嵌入 + 章节嵌入 + info.json |

### 3.3 三个参数在 FFmpegMetadataPP 中的作用

`FFmpegMetadataPP` 构造时接收三个独立开关（[yt_dlp/postprocessor/ffmpeg.py](yt_dlp/postprocessor/ffmpeg.py#L664-L668)）：

```python
def __init__(self, downloader, add_metadata=True, add_chapters=True, add_infojson='if_exists'):
    self._add_metadata = add_metadata
    self._add_chapters = add_chapters
    self._add_infojson = add_infojson
```

在 `run()` 中三者独立判断：
- `self._add_chapters and info.get('chapters')` → 生成章节元数据文件
- `self._add_metadata` → 生成通用 metadata 选项
- `self._add_infojson` → MKV 附加 info.json

因此可能出现 **FFmpegMetadataPP 已启用但 `add_chapters=False`** 的情况（如只嵌入通用元数据，不嵌入章节）。

---

## 4. 四种场景详细对比

### 场景 A：仅标记片段 — `--sponsorblock-mark sponsor,intro`

**选项状态：**
- `sponsorblock_mark = {'sponsor', 'intro'}`
- `sponsorblock_remove = set()`
- `sponsorblock_query = {'sponsor', 'intro'}`
- `addchapters` 初始为 `None` → 因 `sponsorblock_mark` 为真 → `addchapters = True`
- `addmetadata = False`

**启用的后处理器：**

| 后处理器 | when | 是否启用 | 作用 |
|---------|------|---------|------|
| SponsorBlockPP | after_filter | ✅ | 获取片段，写入 `info['sponsorblock_chapters']` |
| ModifyChaptersPP | post_process | ✅ | 合并 sponsor 章节到普通章节，生成标题，**不剪切视频** |
| FFmpegMetadataPP | post_process | ✅ (`add_chapters=True`) | 将合并后的章节嵌入视频文件 |

**ModifyChaptersPP 在无删除时的行为：**

即使没有任何片段需要删除（`cuts` 为空），ModifyChaptersPP 仍然会：
1. 将 `sponsorblock_chapters` 与 `chapters` 合并
2. 通过最小堆算法按时间排序、处理重叠
3. 执行 `_remove_tiny_rename_sponsors()`：
   - 生成 SponsorBlock 章节标题（`[SponsorBlock]: ...`）
   - 合并同类别的相邻章节
4. 将结果写回 `info['chapters']`

然后因为 `cuts` 为空，直接 `return [], info` —— **不调用 ffmpeg 剪切视频**。

**最终效果：** 视频文件中包含 SponsorBlock 章节标记，但视频内容完整未剪切。

---

### 场景 B：仅删除片段 — `--sponsorblock-remove sponsor`

**选项状态：**
- `sponsorblock_mark = set()`
- `sponsorblock_remove = {'sponsor'}`
- `sponsorblock_query = {'sponsor'}`
- `addchapters` 初始为 `None` → 因 `sponsorblock_mark` 为空 → **保持 `None`**
- `addmetadata = False`

**启用的后处理器：**

| 后处理器 | when | 是否启用 | 作用 |
|---------|------|---------|------|
| SponsorBlockPP | after_filter | ✅ | 获取片段，写入 `info['sponsorblock_chapters']` |
| ModifyChaptersPP | post_process | ✅ | 标记要删除的 sponsor 片段，**实际剪切视频** |
| FFmpegMetadataPP | post_process | ❌ | `addchapters=None` 为 falsy，不启用 |

**关键细节：为什么 FFmpegMetadataPP 不启用？**

因为 `sponsorblock_remove` 为真不会触发 `addchapters = True`，只有 `sponsorblock_mark` 和 `addmetadata` 会触发。

`FFmpegMetadataPP` 的启用条件：
```python
if opts.addmetadata or opts.addchapters or opts.embed_infojson:
```
`None` 在布尔上下文中为假，所以条件不成立。

**最终效果：** 视频被剪切，广告片段被移除，但**章节信息不会被写入文件元数据**（视频文件本身没有章节标记）。

---

### 场景 C：标记 + 删除 — `--sponsorblock-mark all --sponsorblock-remove sponsor,intro`

**选项状态：**
- `sponsorblock_mark = {all 类别}`
- `sponsorblock_remove = {'sponsor', 'intro'}`
- `sponsorblock_query = {all 类别}`（并集）
- `addchapters` 初始为 `None` → 因 `sponsorblock_mark` 为真 → `addchapters = True`

**启用的后处理器：**

| 后处理器 | when | 是否启用 | 作用 |
|---------|------|---------|------|
| SponsorBlockPP | after_filter | ✅ | 获取所有类别的片段 |
| ModifyChaptersPP | post_process | ✅ | 删除 sponsor/intro 片段，保留其余作为章节 |
| FFmpegMetadataPP | post_process | ✅ | 将保留的章节嵌入视频文件 |

**数据流转：**
1. SponsorBlockPP 获取所有类别片段
2. ModifyChaptersPP 中，`sponsor` 和 `intro` 类别的章节被标记 `remove=True`
3. `poi_highlight`、`chapter` 等不可删除类别仅作为章节标记
4. 剪切视频后，剩余章节重新编号
5. FFmpegMetadataPP 将最终章节嵌入文件

**最终效果：** 广告片段被剪切，其余 SponsorBlock 标记（如 highlight、chapter）作为章节嵌入文件。

---

### 场景 D：标记但禁用章节嵌入 — `--sponsorblock-mark all --no-embed-chapters`

**选项状态：**
- `sponsorblock_mark = {all 类别}`
- `addchapters = False`（显式设置，**不是 None**）
- 自动开启条件：`sponsorblock_mark 为真 and addchapters is None` → **False**（不满足）

**启用的后处理器：**

| 后处理器 | when | 是否启用 | 作用 |
|---------|------|---------|------|
| SponsorBlockPP | after_filter | ✅ | 获取片段，写入 `info['sponsorblock_chapters']` |
| ModifyChaptersPP | post_process | ✅ | 合并章节到 `info['chapters']`（内存中） |
| FFmpegMetadataPP | post_process | ❌ | `addchapters=False`，不启用 |

**关键细节：ModifyChaptersPP 仍会运行**

ModifyChaptersPP 的启用条件是 `remove_chapters or sponsorblock_query`，与 `addchapters` 无关。只要有 `sponsorblock_query`（即使只有 mark 没有 remove），ModifyChaptersPP 都会被加入链中。

它会将 SponsorBlock 章节合并到 `info['chapters']`，但因为没有后续的 FFmpegMetadataPP，这些章节**只存在于内存的 info 字典中，不会被写入视频文件**。

**最终效果：**
- 内存中 `info['chapters']` 包含 SponsorBlock 章节（可被后续后处理器或 `--print` 使用）
- 视频文件中**没有**章节元数据
- 视频内容完整未剪切

---

## 5. 选项-后处理器矩阵

| 命令选项 | SponsorBlockPP | ModifyChaptersPP | FFmpegMetadataPP<br/>(add_chapters) | FFmpegMetadataPP<br/>(add_metadata) |
|---------|:---:|:---:|:---:|:---:|
| （无选项） | ❌ | ❌ | ❌ | ❌ |
| `--sponsorblock-mark all` | ✅ | ✅ | ✅ | ❌ |
| `--sponsorblock-remove sponsor` | ✅ | ✅ | ❌ | ❌ |
| `--sponsorblock-mark all --sponsorblock-remove sponsor` | ✅ | ✅ | ✅ | ❌ |
| `--embed-metadata` | ❌ | ❌ | ✅ | ✅ |
| `--embed-metadata --sponsorblock-mark all` | ✅ | ✅ | ✅ | ✅ |
| `--embed-chapters` | ❌ | ❌ | ✅ | ❌ |
| `--no-embed-chapters --sponsorblock-mark all` | ✅ | ✅ | ❌ | ❌ |
| `--embed-metadata --no-embed-chapters` | ❌ | ❌ | ❌ | ✅ |
| `--remove-chapters "Intro"` | ❌ | ✅ | ❌ | ❌ |

---

## 6. SponsorBlockPP：片段标记机制

文件：[yt_dlp/postprocessor/sponsorblock.py](yt_dlp/postprocessor/sponsorblock.py)

### 6.1 分类体系

```python
CATEGORIES = {
    'sponsor': 'Sponsor',
    'intro': 'Intermission/Intro Animation',
    'outro': 'Endcards/Credits',
    'selfpromo': 'Unpaid/Self Promotion',
    'preview': 'Preview/Recap',
    'filler': 'Filler Tangent',
    'interaction': 'Interaction Reminder',
    'music_offtopic': 'Non-Music Section',
    'hook': 'Hook/Greetings',
    # 以下为 NON_SKIPPABLE，不可被 --sponsorblock-remove 删除
    'poi_highlight': 'Highlight',      # POI 类别
    'chapter': 'Chapter',
}
```

### 6.2 run() 流程（第 39-47 行）

```python
def run(self, info):
    extractor = info['extractor_key']
    if extractor not in self.EXTRACTORS:  # 目前仅支持 Youtube
        return [], info

    self.to_screen('Fetching SponsorBlock segments')
    # 核心写入：将获取的章节存入 info['sponsorblock_chapters']
    info['sponsorblock_chapters'] = self._get_sponsor_chapters(info, info.get('duration'))
    return [], info   # <-- 不删除任何文件，仅修改 info 字典
```

### 6.3 API 调用：_get_sponsor_segments()（第 94-105 行）

使用 **SHA-256 哈希前缀查询**（隐私保护）：

```python
def _get_sponsor_segments(self, video_id, service):
    video_hash = hashlib.sha256(video_id.encode('ascii')).hexdigest()
    # 仅发送前 4 个哈希字符
    url = f'{self._API_URL}/api/skipSegments/{video_hash[:4]}?' + urllib.parse.urlencode({
        'service': service,
        'categories': json.dumps(self._categories),
        'actionTypes': json.dumps(['skip', 'poi', 'chapter']),
    })
    for d in self._download_json(url) or []:
        if d['videoID'] == video_id:
            return d['segments']
    return []
```

### 6.4 片段过滤与转换：_get_sponsor_chapters()（第 49-92 行）

**过滤逻辑 duration_filter**：
- 跳过 (0,0) 整段视频标记
- 起始 ≤1s 修正为 0
- POI（Point of Interest）类别延长为 1s 以便标记
- 结尾与视频时长差 ≤1s 修正为精确时长
- 时长偏差校验：绝对差 <1s，或相对偏差 <5%

**转换为章节格式 to_chapter**：

```python
def to_chapter(s):
    (start, end), cat = s['segment'], s['category']
    title = s['description'] if cat == 'chapter' else self.CATEGORIES[cat]
    return {
        'start_time': start,
        'end_time': end,
        'category': cat,
        'title': title,
        'type': s['actionType'],    # 'skip' | 'poi' | 'chapter'
        '_categories': [(cat, start, end, title)],  # 内部使用，便于后续合并
    }
```

**关键输出**：`info['sponsorblock_chapters']` — 这是 SponsorBlockPP 与 ModifyChaptersPP 之间的数据契约。

---

## 7. ModifyChaptersPP：元数据改写与视频裁剪

文件：[yt_dlp/postprocessor/modify_chapters.py](yt_dlp/postprocessor/modify_chapters.py)

### 7.1 run() 主流程（第 25-75 行）

```python
def run(self, info):
    self._fixup_chapters(info)

    # Step 1: 标记需要删除的章节（深拷贝避免破坏原始数据）
    chapters, sponsor_chapters = self._mark_chapters_to_remove(
        copy.deepcopy(info.get('chapters')) or [],
        copy.deepcopy(info.get('sponsorblock_chapters')) or [])
    if not chapters and not sponsor_chapters:
        return [], info

    real_duration = self._get_real_video_duration(info['filepath'])
    if not chapters:
        chapters = [{'start_time': 0, 'end_time': info.get('duration') or real_duration, 'title': info['title']}]

    # Step 2: 核心算法 — 合并、裁剪、重新编号
    info['chapters'], cuts = self._remove_marked_arrange_sponsors(chapters + sponsor_chapters)
    if not cuts:
        return [], info
    # ...

    # Step 3: 实际剪切视频文件
    concat_opts = self._make_concat_opts(cuts, real_duration)
    in_out_files = [remove_chapters(info['filepath'], False)]
    in_out_files.extend(remove_chapters(in_file, True) for in_file in self._get_supported_subs(info))

    # 文件替换：原文件 → *.uncut，剪切后 → 原文件名
    for in_file, out_file in in_out_files:
        uncut_file = prepend_extension(in_file, 'uncut')
        os.replace(in_file, uncut_file)
        os.replace(out_file, in_file)

    return files_to_remove, info
```

### 7.2 标记阶段：_mark_chapters_to_remove()（第 77-110 行）

对两种章节来源分别打 `remove=True` 标记：

```python
# 按正则匹配普通章节
for c in chapters:
    if any(regex.search(c['title']) for regex in self._remove_chapters_patterns):
        c['remove'] = True

# 按 SponsorBlock 类别匹配
for c in sponsor_chapters:
    if c['category'] in self._remove_sponsor_segments:
        c['remove'] = True

# 用户手动指定的时间范围
sponsor_chapters.extend({
    'start_time': start,
    'end_time': end,
    'category': 'manually_removed',
    '_categories': [('manually_removed', start, end, 'Manually removed')],
    'remove': True,
} for start, end in self._ranges_to_remove)
```

注意 `__init__`（第 19 行）中的过滤：

```python
self._remove_sponsor_segments = set(remove_sponsor_segments or []) - set(SponsorBlockPP.NON_SKIPPABLE_CATEGORIES.keys())
```

`poi_highlight` 和 `chapter` 类别无法被删除，只能标记。

### 7.3 核心算法：_remove_marked_arrange_sponsors()（第 125-264 行）

使用 **最小堆（priority queue）** 按 `start_time` 处理所有章节，处理 8 种重叠情况：

| 情况 | 当前 | 下一 | 处理方式 |
|-----|------|------|---------|
| 1 | cut | cut | 合并 end_time |
| 2 | cut | sponsor/normal | 裁剪后者开头，重新入堆 |
| 3 | sponsor/normal | cut | 裁剪前者结尾；若 cut 被包含则用 cut_idx 登记 |
| 4 | sponsor | normal | 裁剪 normal 开头，重新入堆 |
| 5 | normal | sponsor | 拆分、合并 _categories |
| 6 | sponsor | sponsor | 合并 _categories |

核心数据结构：
- `'remove' in c` → 表示这是一个需要被剪切掉的片段（cut）
- `'_categories' in c` → 表示这是一个 SponsorBlock 章节（可能包含多个合并后的类别）
- `c['cut_idx']` → 指向第一个落在该章节内的 cut，用于后续计算时长扣减

### 7.4 后处理：_remove_tiny_rename_sponsors()（第 266-311 行）

- **微小章节合并**：时长 < 1s 且由切割产生的（`'_was_cut' in c` 或 `'_categories' in c`）章节被合并到相邻章节
- **类别字段展开**：将内部的 `_categories` 列表展开为：
  ```python
  c.update({
      'category': category,          # 最窄的类别（时长最短）
      'categories': orderedSet(...), # 所有类别
      'name': category_name,
      'category_names': orderedSet(...),
  })
  c['title'] = self._downloader.evaluate_outtmpl(
      self._sponsorblock_chapter_title, c.copy())  # 默认模板: '[SponsorBlock]: %(category_names)l'
  ```
- **同名合并**：相邻且 title 相同的 SponsorBlock 章节被合并

---

## 8. MetadataParserPP：通用元数据改写

文件：[yt_dlp/postprocessor/metadataparser.py](yt_dlp/postprocessor/metadataparser.py)

### 8.1 两种动作详解

MetadataParserPP 支持两种动作，分别由 `--parse-metadata` 和 `--replace-in-metadata` 触发。

#### 8.1.1 INTERPRET 动作（`--parse-metadata FROM:TO`）

**命令格式**：`--parse-metadata "[WHEN:]FROM:TO"`

**两阶段处理流程**：

```
阶段 1: FROM → 模板求值 → 得到字符串 data_to_parse
阶段 2: TO → 编译为正则 → 命名分组匹配 data_to_parse → 写入 info 字段
```

**阶段 1：FROM 模板求值**（[field_to_template](yt_dlp/postprocessor/metadataparser.py#L26-L35) + [evaluate_outtmpl](yt_dlp/postprocessor/metadataparser.py#L69-L69)）

FROM 参数经过 `field_to_template()` 转换后，由 `evaluate_outtmpl()` 求值为字符串：

```python
template = self.field_to_template(inp)  # "title" → "%(title)s"
data_to_parse = self._downloader.evaluate_outtmpl(template, info)
```

- 如果 FROM 是纯字段名（如 `title`），转换为 `%(title)s` 模板
- 如果 FROM 已包含 `%(...)s` 语法（如 `%(title)s - %(artist)s`），直接使用
- 求值结果是一个字符串，例如 `"My Video - My Artist"`

**阶段 2：TO 正则匹配与命名分组写入**（[format_to_regex](yt_dlp/postprocessor/metadataparser.py#L38-L59) + [interpretter](yt_dlp/postprocessor/metadataparser.py#L67-L81)）

TO 参数通过 `format_to_regex()` 转换为正则表达式，其中 `%(...)s` 变为命名捕获组：

```python
out_re = re.compile(self.format_to_regex(out))
match = out_re.search(data_to_parse)
for attribute, value in filter_dict(match.groupdict()).items():
    info[attribute] = value
```

`format_to_regex()` 的转换规则（[metadataparser.py](yt_dlp/postprocessor/metadataparser.py#L38-L59)）：

| TO 参数 | 生成的正则 | 说明 |
|--------|----------|------|
| `%(title)s - %(artist)s` | `(?P<title>.+)\ \-\ (?P<artist>.+)` | 每个 `%(...)s` 变为 `(?P<name>.+)` |
| `%(title)s` | `(?P<title>.+)` | 单字段，匹配整个字符串 |
| `title`（纯字段名） | `(?s)(?P<title>.+)` | 纯字段名自动转为匹配全部的正则 |
| `非模板文本` | 原样保留 | 无 `%(...)s` 则不转换（字面量正则） |

**关键机制**：匹配结果的 `groupdict()` 中，**只有成功匹配的命名组**才会被写入 `info`。`filter_dict()` 过滤掉值为 `None` 的组（即未参与匹配的可选组）。

**完整示例推导**：

```bash
--parse-metadata "%(title)s - %(artist)s:%(artist)s - %(title)s"
```

1. FROM: `%(title)s - %(artist)s` → 求值 → `"My Video - My Artist"`
2. TO: `%(artist)s - %(title)s` → 正则 → `(?P<artist>.+)\ \-\ (?P<title>.+)`
3. 匹配: `match.groupdict() = {'artist': 'My Video', 'title': 'My Artist'}`
4. 写入: `info['artist'] = 'My Video'`, `info['title'] = 'My Artist'`
5. 结果: title 和 artist 被互换了

**单字段提取示例**：

```bash
--parse-metadata "title:%(artist)s - %(title)s"
```

1. FROM: `title` → 转换为 `%(title)s` → 求值 → `"My Video - My Artist"`
2. TO: `%(artist)s - %(title)s` → 正则 → `(?P<artist>.+)\ \-\ (?P<title>.+)`
3. 匹配: 仅当 title 包含 `" - "` 时才成功
4. 写入: 从 title 中提取出 artist 和 title 两个字段

**纯赋值不是直接赋值**：

`--parse-metadata` **没有直接赋值语法**。它的本质是「模板求值 → 正则匹配 → 命名分组写入」，必须经过正则匹配这一步。要实现类似"直接赋值"的效果，需要让 TO 正则匹配整个 FROM 求值结果：

```bash
# ✅ 正确：用纯字段名作为 TO，format_to_regex 会生成匹配全体的 (?P<title>.+)
--parse-metadata "title:artist"

# 等价过程：
#   FROM "title" → "%(title)s" → 求值 → "My Video"
#   TO   "artist" → (?s)(?P<artist>.+) → 匹配 → {'artist': 'My Video'}
#   结果: info['artist'] = 'My Video'（将 title 值复制到 artist）
```

```bash
# ❌ 错误理解：这不是"把 title 设为字面量 '新标题'"
--parse-metadata "title:新标题"

# 实际过程：
#   FROM "title" → "%(title)s" → 求值 → "My Video"（不是字面量 "新标题"！）
#   TO   "新标题" → 正则中无 %(...)s → 当作字面量正则 新标题
#   在 "My Video" 中搜索字面量 "新标题" → 匹配失败
#   结果: 不修改任何字段
```

**正确的赋值方式**：利用 FROM 模板生成字面量，再让 TO 正则捕获它：

```bash
# 将 title 设为字面量 "新标题"
--parse-metadata "新标题:title"

# 过程：
#   FROM "新标题" → 模板中无 %(...)s → 求值后仍为字面量 "新标题"
#   TO   "title" → (?s)(?P<title>.+) → 匹配 → {'title': '新标题'}
#   结果: info['title'] = '新标题'
```

**FROM 中的冒号转义**：`MetadataFromFieldPP.to_action()` 使用 `(?<!\\):` 分割 FROM 和 TO（[metadataparser.py](yt_dlp/postprocessor/metadataparser.py#L109-L109)），因此 FROM 中的冒号需要用 `\:` 转义：

```bash
# FROM 包含冒号时需要转义
--parse-metadata "http\://example.com/%(id)s:url"
```

#### 8.1.2 REPLACE 动作（`--replace-in-metadata FIELDS REGEX REPLACE`）

**命令格式**：`--replace-in-metadata "[WHEN:]FIELDS REGEX REPLACE"`

REPLACE 动作对 **info 字典中已有的字符串字段** 执行正则替换，核心代码（[replacer](yt_dlp/postprocessor/metadataparser.py#L84-L101)）：

```python
def replacer(self, field, search, replace):
    def f(info):
        val = info.get(field)
        if val is None:
            self.to_screen(f'Video does not have a {field}')
            return
        elif not isinstance(val, str):
            self.report_warning(
                f'Cannot replace in field {field} since it is a {type(val).__name__}')
            return
        info[field], n = search_re.subn(replace, val)
        # ...
    search_re = re.compile(search)
    return f
```

**三个关键守卫条件**：

| 条件 | 字段状态 | 行为 |
|-----|---------|------|
| `val is None` | 字段不存在 | 跳过，输出 "Video does not have a {field}" |
| `not isinstance(val, str)` | 字段存在但非字符串（如 list, int） | 跳过并警告，**不会**修改该字段 |
| `isinstance(val, str)` | 字段存在且为字符串 | 执行 `re.subn(search, replace, val)` |

**FIELDS 参数支持逗号分隔的多字段**：在 `__init__.py` 的 `metadataparser_actions()` 中（[yt_dlp/__init__.py](yt_dlp/__init__.py#L429-L429)），逗号分隔的字段名展开为多个独立的 REPLACE 动作：

```python
# --replace-in-metadata "title,description" "old" "new"
# 展开为两个动作：
#   (REPLACE, 'title', 'old', 'new')
#   (REPLACE, 'description', 'old', 'new')
actions = ((MetadataParserPP.Actions.REPLACE, x, *f[1:]) for x in f[0].split(','))
```

**与 INTERPRET 动作的本质区别**：

| 维度 | INTERPRET (`--parse-metadata`) | REPLACE (`--replace-in-metadata`) |
|-----|------|------|
| 输入来源 | FROM 模板求值后的字符串 | info 中指定字段的**现有值** |
| 输出目标 | TO 正则的**命名捕获组**决定写入哪些字段 | 替换结果写回**原字段** |
| 字段要求 | FROM 引用缺失字段时会得到占位值或空串，可能导致 TO 匹配不到预期内容 | 目标字段必须存在**且为字符串** |
| 能否创建新字段 | ✅ TO 中的命名组可创建新字段 | ❌ 只能修改已有字符串字段 |
| 能否修改非字符串字段 | ❌ 模板求值依赖字符串 | ❌ 显式守卫 `not isinstance(val, str)` |
| 能否修改列表/字典字段 | ❌ | ❌ |

**示例**：

```bash
# 在 title 中替换 "官方MV" 为 "音乐视频"
--replace-in-metadata title "官方MV" "音乐视频"

# 同时在 title 和 description 中替换
--replace-in-metadata "title,description" "旧词" "新词"

# 使用正则反向引用
--replace-in-metadata title "(\d{4})-(\d{2})-(\d{2})" "\2/\3/\1"
```

**REPLACE 无法修改章节标题的原因**：

即使指定 `--replace-in-metadata chapters "old" "new"`，也会因为 `info['chapters']` 是一个**列表**（不是字符串）而触发守卫条件，输出警告并跳过。

### 8.2 执行阶段与顺序关系

#### 8.2.1 执行阶段配置

MetadataParserPP 的执行阶段由 `--parse-metadata` 和 `--replace-in-metadata` 的 `[WHEN:]` 前缀决定，定义在 [yt_dlp/options.py](yt_dlp/options.py#L1729-L1739)：

```python
# --parse-metadata [WHEN:]FROM:TO
# --replace-in-metadata [WHEN:]FIELDS REGEX REPLACE
**when_prefix('pre_process')**  # 默认阶段为 pre_process
```

`when_prefix()` 函数（[yt_dlp/options.py](yt_dlp/options.py#L298-L308)）允许用户指定任意 `POSTPROCESS_WHEN` 阶段：

```python
def when_prefix(default):
    return {
        'allowed_keys': '|'.join(map(re.escape, POSTPROCESS_WHEN)),
        'default_key': default,  # 'pre_process'
        # ...
    }
```

在 [yt_dlp/__init__.py](yt_dlp/__init__.py#L630-L635) 的 `get_postprocessors()` 中，按用户指定的阶段生成：

```python
for when, actions in opts.parse_metadata.items():
    yield {
        'key': 'MetadataParser',
        'actions': actions,
        'when': when,  # 用户指定的阶段
    }
```

#### 8.2.2 与 SponsorBlockPP 的顺序关系

SponsorBlockPP 固定在 `after_filter` 阶段运行，而 MetadataParserPP 的阶段由用户指定，因此有以下几种情况：

| MetadataParserPP 阶段 | 与 SponsorBlockPP 的顺序 | 说明 |
|----------------------|------------------------|------|
| `pre_process`（默认） | **MetadataParserPP 在前** | 先改写元数据，再获取 SponsorBlock 片段 |
| `after_filter` | **同阶段，MetadataParserPP 在前** | 同阶段按添加顺序，MetadataParserPP 先被 yield |
| `video` / `before_dl` | **SponsorBlockPP 在前** | 先获取片段，再改写元数据 |
| `post_process` / `after_move` | **SponsorBlockPP 在前** | 先获取片段，下载后再改写元数据 |

**同阶段顺序原因**：在 `get_postprocessors()` 中，MetadataParserPP（第 630 行）的 yield 位于 SponsorBlockPP（第 636 行）之前，因此同阶段时 MetadataParserPP 先执行。

#### 8.2.3 各阶段效果对比

| 阶段 | 可改写的 info 字段 | 与关键处理器的顺序关系 |
|-----|-------------------|-------------------|
| `pre_process`（默认） | `title`, `artist`, `description`, `uploader` 等原始字段 | 在 SponsorBlockPP **之前** |
| `after_filter` | 同上 + 筛选后的字段 | 在 SponsorBlockPP **之前**（同阶段先被 yield） |
| `video` / `before_dl` | 同上 + 格式选择后的字段 | 在 SponsorBlockPP **之后**，在 ModifyChaptersPP **之前** |
| `post_process` | 同上 + `filepath`, `duration` 等下载后字段 | 在 ModifyChaptersPP **之前**，在 FFmpegMetadataPP **之前** |
| `after_move` | 同上 + 最终文件路径 | 在 FFmpegMetadataPP **之后** |

**重要修正**：在 `post_process` 阶段内，MetadataParserPP 在 ModifyChaptersPP **之前**运行（见 [§2.2.2 post_process 阶段精确顺序](#222-post_process-阶段精确顺序)），而不是之后。

### 8.3 post_process 阶段：三者顺序与影响边界

当 MetadataParserPP 被指定在 `post_process` 阶段运行时，三个核心后处理器的执行顺序为：

```
MetadataParserPP → ModifyChaptersPP → FFmpegMetadataPP
```

#### 8.3.1 三者数据读写边界

| 后处理器 | 读取的 info 字段 | 修改的 info 字段 | 修改文件？ |
|---------|----------------|----------------|-----------|
| **MetadataParserPP** | 任意顶层字段（字符串） | 顶层字段（`title`, `artist`, `meta_xxx` 等） | ❌ |
| **ModifyChaptersPP** | `chapters`, `sponsorblock_chapters`, `filepath`, `duration` | `chapters`（整体替换）, `duration` | ✅ 剪切视频 |
| **FFmpegMetadataPP** | `chapters`（只读）, `title`, `artist`, `description` 等 | 无（info 不变） | ✅ 写入元数据 |

#### 8.3.2 对章节标题的影响边界

**MetadataParserPP 改写通用元数据 → 不影响 SponsorBlock 章节标题**

原因：
1. **数据隔离**：SponsorBlock 章节标题模板求值时传入 `c.copy()`（[modify_chapters.py](yt_dlp/postprocessor/modify_chapters.py#L304-L304)），即章节字典自身的拷贝，不包含 `info` 级别的 `title`, `artist` 等字段。
2. **模板字段限制**：章节标题默认模板 `'[SponsorBlock]: %(category_names)l'` 只引用 `category_names` 等章节内部字段。
3. **即使 MetadataParserPP 在 ModifyChaptersPP 之前也没用**：因为两者操作的是 info 字典的不同层级——MetadataParserPP 改顶层，ModifyChaptersPP 改 `chapters` 列表内各章节对象的 `title`。

**ModifyChaptersPP 生成章节标题 → 影响文件中的章节元数据**

- ModifyChaptersPP 生成/修改每个章节的 `title` 字段
- FFmpegMetadataPP 读取 `info['chapters']` 中每个章节的 `title`，写入 FFMETADATA1 文件

#### 8.3.3 对章节列表的影响边界

| 操作 | 能否改变 info['chapters'] 列表 | 说明 |
|-----|:---:|------|
| MetadataParserPP 改 `title` | ❌ | 只能改顶层键，不能遍历修改列表内元素 |
| MetadataParserPP 改 `chapters` 整体 | ⚠️ 理论上可以 | 需模板直接生成整个列表，实际几乎不可行 |
| ModifyChaptersPP | ✅ | 合并、删除、拆分章节，完全重写列表 |
| FFmpegMetadataPP | ❌ | 只读，不改写 info |

#### 8.3.4 对最终文件元数据的影响边界

**通用元数据（title, artist, comment 等）：**
- MetadataParserPP 改写 → ✅ 有效（在 FFmpegMetadataPP 之前）
- ModifyChaptersPP → ❌ 不涉及
- FFmpegMetadataPP → 最终写入文件

**章节元数据：**
- MetadataParserPP → ❌ 不影响章节列表和章节标题
- ModifyChaptersPP → ✅ 决定章节数量、时间、标题
- FFmpegMetadataPP → 最终写入文件

**边界总结表：**

| 改写目标 | MetadataParserPP<br/>能否影响 | ModifyChaptersPP<br/>能否影响 | FFmpegMetadataPP<br/>是否消费 |
|---------|:---:|:---:|:---:|
| 文件 `title` 元数据 | ✅ | ❌ | ✅ 读取 `info['title']` |
| 文件 `artist` 元数据 | ✅ | ❌ | ✅ 读取 `info['artist']` |
| 自定义 `comment` 元数据 | ✅（`meta_comment`） | ❌ | ✅ 读取 `meta_*` |
| 章节数量 | ❌ | ✅ | ✅ 读取 `info['chapters']` |
| 章节时间点 | ❌ | ✅ | ✅ 读取章节 `start_time/end_time` |
| 章节标题文字 | ❌ | ✅ | ✅ 读取章节 `title` |
| 视频文件时长 | ❌（可改 `info['duration']` 但不影响文件） | ✅（剪切改变实际时长） | ❌ |

#### 8.3.5 典型场景验证

**场景 1：post_process 阶段改标题 + SponsorBlock 标记**

```bash
# 正确写法：FROM 模板拼接后缀，TO 用纯字段名捕获
yt-dlp --parse-metadata "post_process:%(title)s [已去广告]:title" \
       --sponsorblock-mark all \
       URL
```

结果：
- ✅ 文件 `title` 元数据变为 "原标题 [已去广告]"（FROM 模板求值后由 TO 正则捕获写入）
- ❌ SponsorBlock 章节标题不变（仍为 "[SponsorBlock]: Sponsor" 等）
- ✅ 章节数量和时间点正确

**场景 2：替换章节标题中的关键词**

MetadataParserPP **无法**直接替换每个章节的 `title` 字段，因为：
- 章节在 `info['chapters']` 列表中，是嵌套结构
- INTERPRET/REPLACE 动作只操作顶层键，不遍历列表
- 没有 "对每个章节应用替换" 的机制

要修改章节标题，需要自定义后处理器或使用 `--exec` 调用外部工具。

### 8.4 章节标题生成机制详解

本节深入说明 SponsorBlock 章节标题如何生成，以及为什么通用元数据改写不影响它。

#### 8.4.1 生成代码

SponsorBlock 章节标题由 ModifyChaptersPP 的 `_remove_tiny_rename_sponsors()` 生成（[yt_dlp/postprocessor/modify_chapters.py](yt_dlp/postprocessor/modify_chapters.py#L294-L304)）：

```python
cats = c.pop('_categories', None)
if cats:
    category, _, _, category_name = min(cats, key=lambda c: c[2] - c[1])
    c.update({
        'category': category,
        'categories': orderedSet(x[0] for x in cats),
        'name': category_name,
        'category_names': orderedSet(x[3] for x in cats),
    })
    # 关键：用 c.copy() 作为模板上下文，不是 info 字典
    c['title'] = self._downloader.evaluate_outtmpl(
        self._sponsorblock_chapter_title, c.copy())
```

默认模板 `DEFAULT_SPONSORBLOCK_CHAPTER_TITLE`（[yt_dlp/postprocessor/modify_chapters.py](yt_dlp/postprocessor/modify_chapters.py#L11-L11)）：

```python
DEFAULT_SPONSORBLOCK_CHAPTER_TITLE = '[SponsorBlock]: %(category_names)l'
```

#### 8.4.2 模板可用字段

**SponsorBlock 章节标题模板仅使用章节自身的字段**，不使用 `info` 字典中的通用元数据字段：

| 可用字段 | 来源 | 说明 |
|---------|------|------|
| `start_time` | 章节 | 章节起始时间 |
| `end_time` | 章节 | 章节结束时间 |
| `category` | 章节 | 主要类别（时长最短的） |
| `categories` | 章节 | 所有类别列表 |
| `name` | 章节 | 主要类别名称 |
| `category_names` | 章节 | 所有类别名称列表 |

**不使用** `info['title']`、`info['artist']`、`info['uploader']` 等通用元数据字段。

#### 8.4.3 解耦结论

通用元数据改写与 SponsorBlock 章节标题完全解耦，因为：
1. **上下文隔离**：模板求值传入 `c.copy()`（章节字典），而非整个 `info` 字典
2. **字段不重叠**：章节模板使用的 `category_names` 等字段与 `info` 顶层字段不在同一层级
3. **来源不同**：`category_names` 来自 SponsorBlock API 返回的类别名称映射，与视频标题无关

**示例验证**：
```bash
# 目标：将 title 改为 "原标题 [无广告]"
# ❌ 错误解法：这不会生效
#   FROM "title" → "%(title)s" → 求值 → "原标题"
#   TO   "%(title)s [无广告]" → 正则 → (?P<title>.+)\ \[无广告\]
#   在 "原标题" 中搜索 → 无匹配（缺少 " [无广告]" 后缀）
yt-dlp --parse-metadata "title:'%(title)s [无广告]'" \
       --sponsorblock-mark sponsor URL

# ✅ 正确解法：用 FROM 模板拼接后缀，TO 用纯字段名捕获全部
#   FROM "%(title)s [无广告]" → 求值 → "原标题 [无广告]"
#   TO   "title" → (?s)(?P<title>.+) → 匹配 → info['title'] = '原标题 [无广告]'
yt-dlp --parse-metadata "%(title)s [无广告]:title" \
       --sponsorblock-mark sponsor URL

# 即使改写了 info['title']，SponsorBlock 章节标题仍为 "[SponsorBlock]: Sponsor"
# 因为章节标题模板使用 c.copy() 上下文，不引用 info['title']
```

### 8.5 元数据改写对文件元数据的影响

FFmpegMetadataPP 的 `_get_metadata_opts()`（[yt_dlp/postprocessor/ffmpeg.py](yt_dlp/postprocessor/ffmpeg.py#L728-L795)）使用修改后的 `info` 字段生成文件元数据。

#### 8.5.1 字段优先级机制

```python
def add(meta_list, info_list=None):
    value = next((
        info[key] for key in [f'{meta_prefix}_', *variadic(info_list or meta_list)]
        if info.get(key) is not None), None)
```

查找顺序（以 `title` 为例）：
1. `info['meta_title']` — 自定义元数据字段（最高优先级）
2. `info['title']` — 主字段
3. `info['track']` — 别名字段

#### 8.5.2 改写不同字段的效果

| 改写目标 | 命令示例 | 实际行为 | 对文件元数据的影响 |
|---------|---------|---------|------------------|
| **复制字段值** | `--parse-metadata "title:artist"` | FROM: `%(title)s`→`"My Video"` → TO: `(?P<artist>.+)`→匹配 → `info['artist']='My Video'` | 文件 `artist` 设为 title 的值 |
| **交换字段** | `--parse-metadata "%(title)s - %(artist)s:%(artist)s - %(title)s"` | FROM 求值→`"My Video - My Artist"` → TO 正则提取两组 → 互写 | 文件 title/artist 互换 |
| **赋值字面量** | `--parse-metadata "新标题:title"` | FROM: 无模板语法→字面量`"新标题"` → TO: `(?P<title>.+)`→匹配 | 文件 `title` 设为 "新标题" |
| **替换字段内容** | `--replace-in-metadata title "原版" "修复版"` | 在 `info['title']` 中正则替换 | 文件 `title` 中 "原版"→"修复版" |
| **自定义元数据** | `--parse-metadata "自定义值:meta_comment"` | FROM→字面量 → TO→`info['meta_comment']='自定义值'` | 文件添加 `comment` 元数据 |

#### 8.5.3 自定义元数据字段的处理

通过 `meta_<key>` 或 `meta<i>_<key>` 语法可以添加自定义元数据（[yt_dlp/postprocessor/ffmpeg.py](yt_dlp/postprocessor/ffmpeg.py#L765-L769)）：

```python
meta_regex = rf'{re.escape(meta_prefix)}(?P<i>\d+)?_(?P<key>.+)'
for key, value in info.items():
    mobj = re.fullmatch(meta_regex, key)
    if value is not None and mobj:
        # meta_xxx → 全局元数据
        # meta<i>_xxx → 第 i 个流的元数据
        metadata[mobj.group('i') or 'common'][mobj.group('key')] = value.replace('\0', '')
```

**示例**：
```bash
# 添加自定义 comment 元数据
# FROM "下载自 YouTube" → 字面量 → TO "meta_comment" → (?P<meta_comment>.+) → 匹配
yt-dlp --parse-metadata "下载自 YouTube:meta_comment" \
       --embed-metadata URL
```

#### 8.5.4 执行顺序对文件元数据的影响

MetadataParserPP 必须在 **`post_process` 阶段结束前**运行才能影响文件元数据。在 `post_process` 阶段内，实际顺序为：

```
post_process 链:
    MetadataParserPP   ← 1. 改写 info['title'], info['artist'] 等（顶层字段）
    FFmpegExtractAudioPP
    FFmpegVideoRemuxerPP
    FFmpegVideoConvertorPP
    FFmpegEmbedSubtitlePP
    ModifyChaptersPP   ← 2. 改写 info['chapters']（章节列表和标题）
    FFmpegMetadataPP   ← 3. 读取 info 并写入文件元数据
    EmbedThumbnailPP
    ...
```

**关键结论：**
- 通用元数据（title, artist 等）：MetadataParserPP 在 FFmpegMetadataPP **之前**，✅ 改写有效
- 章节列表和章节标题：MetadataParserPP 在 ModifyChaptersPP **之前**，但因数据层级不同，❌ 无法影响章节
- 如果 MetadataParserPP 在 `after_move` 阶段运行：FFmpegMetadataPP 已执行完毕，❌ 改写不会写入文件

### 8.6 执行

`run()` 方法依次执行所有 action，仅修改 `info` 字典，不触碰文件：

```python
def run(self, info):
    for f in self._actions:
        f(info)
    return [], info
```

---

## 9. FFmpegMetadataPP：元数据写入文件

文件：[yt_dlp/postprocessor/ffmpeg.py](yt_dlp/postprocessor/ffmpeg.py#L662-L820)

### 9.1 run() 流程（第 678-709 行）

```python
def run(self, info):
    self._fixup_chapters(info)  # 章节规范化
    filename, metadata_filename = info['filepath'], None
    files_to_delete, options = [], []

    # 1. 生成章节元数据文件（FFMETADATA1 格式）
    if self._add_chapters and info.get('chapters'):
        metadata_filename = replace_extension(filename, 'meta')
        options.extend(self._get_chapter_opts(info['chapters'], metadata_filename))
        files_to_delete.append(metadata_filename)

    # 2. 生成通用元数据选项
    if self._add_metadata:
        options.extend(self._get_metadata_opts(info))

    # 3. MKV/MKA 附加 info.json
    if self._add_infojson and info['ext'] in ('mkv', 'mka'):
        options.extend(self._get_infojson_opts(info, infojson_filename))

    # 4. 调用 ffmpeg 重混
    self.run_ffmpeg_multiple_files(
        (filename, metadata_filename), temp_filename,
        itertools.chain(self._options(info['ext']), *options))
    os.replace(temp_filename, filename)
    return [], info
```

### 9.2 章节写入：_get_chapter_opts()（第 711-726 行）

将 `info['chapters']`（其中包含 SponsorBlock 标记的章节）写入 FFMETADATA1 格式：

```python
;FFMETADATA1
[CHAPTER]
TIMEBASE=1/1000
START=0
END=5000
title=Introduction

[CHAPTER]
TIMEBASE=1/1000
START=5000
END=15000
title=[SponsorBlock]: Sponsor
```

然后通过 `-map_metadata 1` 将元数据文件映射到输出文件。

### 9.3 通用元数据：_get_metadata_opts()（第 728-795 行）

字段映射表（部分）：

| info 字段 | FFmpeg metadata |
|-----------|----------------|
| title / track | title |
| upload_date | date |
| description | description, synopsis |
| webpage_url | purl, comment |
| artist / uploader | artist |
| genre / categories / tags | genre |
| album / series | album |
| meta_\<key\> / meta\<i\>_\<key\> | 自定义流元数据 |

---

## 10. 数据流总结

```
用户选项: --sponsorblock-mark all --sponsorblock-remove sponsor,intro
          --parse-metadata "%(title)s [无广告]:title"
          |
          v
validate_options() [yt_dlp/__init__.py]
  ├─ sponsorblock_query = mark | remove
  ├─ addchapters = True (因 sponsorblock_mark 为真且 addchapters is None)
  └─ parse_metadata = { 'pre_process': [ (INTERPRET, '%(title)s [无广告]', 'title') ] }
          |
          v
get_postprocessors()
  ├─ MetadataParserPP (when=pre_process, actions=INTERPRET)
  ├─ SponsorBlockPP (when=after_filter, categories=all)
  ├─ ModifyChaptersPP (when=post_process, remove_sponsor_segments={sponsor,intro})
  └─ FFmpegMetadataPP (when=post_process, add_chapters=True, add_metadata=False)
          |
          v
YouTube 提取器 → info dict { title: '原视频标题', artist: '上传者', ... }
          |
          v
pre_process 阶段: MetadataParserPP.run(info)
  └─ INTERPRET 动作:
       FROM "%(title)s [无广告]" → evaluate_outtmpl → "原视频标题 [无广告]"
       TO   "title" → format_to_regex → (?s)(?P<title>.+)
       匹配 → info['title'] = '原视频标题 [无广告]'    （不影响 SponsorBlock 章节标题）
          |
          v
_match_entry() 筛选
          |
          v
after_filter 阶段: SponsorBlockPP.run(info)
  ├─ API: GET /api/skipSegments/{hash_prefix}
  ├─ 过滤 duration_filter()
  ├─ 转换 to_chapter()
  └─ info['sponsorblock_chapters'] = [ { start, end, category, _categories, ... }, ... ]  ← 数据契约
          |
          v
下载视频文件 → info['filepath']
          |
          v
post_process 阶段: ModifyChaptersPP.run(info)   ← 章节标题仅使用 _categories
  ├─ _mark_chapters_to_remove()
  │     ├─ chapters: 按正则打 remove=True
  │     └─ sponsor_chapters: 按类别打 remove=True (sponsor, intro)
  ├─ _remove_marked_arrange_sponsors()
  │     ├─ 最小堆处理 8 种重叠
  │     ├─ 计算 cuts[] 和 new_chapters[]
  │     └─ info['chapters'] = new_chapters   ← 改写，章节标题由 _categories 生成
  ├─ remove_chapters() 调用 ffmpeg concat 剪切视频
  └─ info['duration'] 更新
          |
          v
post_process 阶段: FFmpegMetadataPP.run(info)
  ├─ _get_chapter_opts(info['chapters']) → *.meta    ← 使用修改后的章节
  ├─ _get_metadata_opts(info) → -metadata title='原视频标题 [无广告]', artist='上传者'
  └─ ffmpeg 重混，章节和元数据嵌入文件
          |
          v
输出文件（章节已嵌入，广告片段已剪切，标题已修改）
```

---

## 11. 关键设计要点

1. **阶段分离**：SponsorBlockPP 在 `after_filter`（下载前）获取数据，ModifyChaptersPP 在 `post_process`（下载后）执行剪切，避免不必要的下载。

2. **数据契约**：SponsorBlockPP 仅写入 `info['sponsorblock_chapters']`，ModifyChaptersPP 消费该字段并改写 `info['chapters']`，两者解耦。

3. **顺序保障**：ModifyChaptersPP 必须在 FFmpegMetadataPP **之前**运行（注释明确标注），因为剪切后的章节才是最终要嵌入的章节。

4. **三态选项**：`addchapters` 使用 `None/True/False` 三态设计，`None` 表示未设置，可被 `sponsorblock_mark` 或 `addmetadata` 自动开启；`False` 表示显式禁用，优先级高于自动开启。

5. **不可删除类别**：`NON_SKIPPABLE_CATEGORIES`（`poi_highlight`、`chapter`）只能用于生成章节标记，不能被视频剪切。

6. **ModifyChaptersPP 的双重作用**：即使没有任何删除操作，只要有 `sponsorblock_query` 就会启用 ModifyChaptersPP，负责将 SponsorBlock 章节合并到普通章节中并生成标题。

7. **仅删除不嵌入**：只使用 `--sponsorblock-remove` 时，视频会被剪切但章节不会嵌入文件；只有 `--sponsorblock-mark` 才会自动触发 `addchapters = True`。

8. **MetadataParserPP 执行阶段灵活性**：可通过 `[WHEN:]` 前缀指定任意 `POSTPROCESS_WHEN` 阶段，默认 `pre_process`；与 SponsorBlockPP、ModifyChaptersPP 的顺序取决于阶段配置。

9. **post_process 阶段精确顺序**：同阶段内按 yield 顺序执行，三者相对顺序为 **MetadataParserPP → ModifyChaptersPP → FFmpegMetadataPP**，由 `get_postprocessors()` 中的 yield 顺序决定。

10. **通用元数据与章节标题解耦**：改写 `info['title']`、`info['artist']` 等通用元数据 **不会影响** SponsorBlock 章节标题。原因有二：一是数据层级不同（顶层字段 vs 章节列表内嵌套字段），二是章节标题模板求值时传入 `c.copy()`（仅章节自身字段）而非整个 `info` 字典。

11. **数据层级隔离**：MetadataParserPP 只能修改 `info` 字典的**顶层字符串字段**，不能遍历 `info['chapters']` 列表逐个修改章节属性。要修改章节标题需通过 `--sponsorblock-chapter-title` 模板或自定义后处理器。

12. **文件元数据改写有效性**：MetadataParserPP 在 `post_process` 阶段内位于 FFmpegMetadataPP **之前**，因此对通用元数据（title, artist, meta_xxx 等）的改写会被 FFmpegMetadataPP 消费并写入文件；但在 `after_move` 阶段运行则无效。

13. **字段优先级机制**：FFmpegMetadataPP 采用 `meta_<key>` > 主字段 > 别名字段的三级优先级；自定义元数据通过 `meta_<key>`（全局）或 `meta<i>_<key>`（指定流）语法添加。

14. **内部字段约定**：
    - `c['_categories']` — SponsorBlock 章节的原始类别列表（用于合并后追踪）
    - `c['remove']` — 标记该时间段需要被剪切
    - `c['cut_idx']` — 指向第一个落在该章节内的 cut 的索引
    - `c['_was_cut']` — 标记该章节是由切割产生的微小片段
