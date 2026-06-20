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

后处理链的执行时机由 8 个阶段组成，定义在 [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/100-yt-dlp/yt_dlp/utils/_utils.py#L2858-L2858)：

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

所有后处理器在 [__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/100-yt-dlp/yt_dlp/__init__.py#L627-L736) 的 `get_postprocessors()` 函数中按 yield 顺序构建：

| 后处理器 | when 阶段 | 说明 |
|---------|----------|------|
| MetadataParserPP | 用户指定（如 pre_process） | 元数据解析/替换 |
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
| ExecPP | 各阶段 | 外部命令执行 |

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

链的执行逻辑在 [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/100-yt-dlp/yt_dlp/YoutubeDL.py#L3798-L3846)：

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

`_pps` 在 [YoutubeDL.__init__](file:///d:/fz/0601-2/solo-dogfeeding/code/100-yt-dlp/yt_dlp/YoutubeDL.py#L640-L640) 中按阶段初始化：

```python
self._pps = {k: [] for k in POSTPROCESS_WHEN}
```

---

## 3. SponsorBlockPP：片段标记机制

文件：[sponsorblock.py](file:///d:/fz/0601-2/solo-dogfeeding/code/100-yt-dlp/yt_dlp/postprocessor/sponsorblock.py)

### 3.1 分类体系

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

### 3.2 run() 流程（第 39-47 行）

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

### 3.3 API 调用：_get_sponsor_segments()（第 94-105 行）

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

### 3.4 片段过滤与转换：_get_sponsor_chapters()（第 49-92 行）

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

## 4. ModifyChaptersPP：元数据改写与视频裁剪

文件：[modify_chapters.py](file:///d:/fz/0601-2/solo-dogfeeding/code/100-yt-dlp/yt_dlp/postprocessor/modify_chapters.py)

### 4.1 run() 主流程（第 25-75 行）

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

### 4.2 标记阶段：_mark_chapters_to_remove()（第 77-110 行）

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

### 4.3 核心算法：_remove_marked_arrange_sponsors()（第 125-264 行）

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

### 4.4 后处理：_remove_tiny_rename_sponsors()（第 266-311 行）

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

## 5. MetadataParserPP：通用元数据改写

文件：[metadataparser.py](file:///d:/fz/0601-2/solo-dogfeeding/code/100-yt-dlp/yt_dlp/postprocessor/metadataparser.py)

### 5.1 两种动作

**INTERPRET** — 从模板解析字段（第 67-81 行）：

```python
# 例：从 "%(title)s - %(artist)s" 解析 title 和 artist
def interpretter(self, inp, out):
    def f(info):
        data_to_parse = self._downloader.evaluate_outtmpl(template, info)
        match = out_re.search(data_to_parse)
        for attribute, value in filter_dict(match.groupdict()).items():
            info[attribute] = value
    # ...
    return f
```

**REPLACE** — 正则替换字段（第 84-101 行）：

```python
def replacer(self, field, search, replace):
    def f(info):
        val = info.get(field)
        info[field], n = search_re.subn(replace, val)
    # ...
    return f
```

### 5.2 执行

`run()` 方法依次执行所有 action，仅修改 `info` 字典，不触碰文件：

```python
def run(self, info):
    for f in self._actions:
        f(info)
    return [], info
```

---

## 6. FFmpegMetadataPP：元数据写入文件

文件：[ffmpeg.py](file:///d:/fz/0601-2/solo-dogfeeding/code/100-yt-dlp/yt_dlp/postprocessor/ffmpeg.py#L662-L820)

### 6.1 run() 流程（第 678-709 行）

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

### 6.2 章节写入：_get_chapter_opts()（第 711-726 行）

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

### 6.3 通用元数据：_get_metadata_opts()（第 728-795 行）

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
| meta_<key> / meta<i>_<key> | 自定义流元数据 |

---

## 7. 数据流总结

```
用户选项: --sponsorblock-mark all --sponsorblock-remove sponsor,intro
          |
          v
get_postprocessors()
  ├─ SponsorBlockPP (when=after_filter)
  └─ ModifyChaptersPP (when=post_process, 必须在 FFmpegMetadataPP 之前)
  └─ FFmpegMetadataPP (when=post_process)
          |
          v
YouTube 提取器 → info dict
          |
          v
after_filter 阶段: SponsorBlockPP.run(info)
  ├─ API: GET /api/skipSegments/{hash_prefix}
  ├─ 过滤 duration_filter()
  ├─ 转换 to_chapter()
  └─ info['sponsorblock_chapters'] = [...]  ← 数据契约
          |
          v
下载视频文件 → info['filepath']
          |
          v
post_process 阶段: ModifyChaptersPP.run(info)
  ├─ _mark_chapters_to_remove()
  │     ├─ chapters: 按正则打 remove=True
  │     └─ sponsor_chapters: 按类别打 remove=True
  ├─ _remove_marked_arrange_sponsors()
  │     ├─ 最小堆处理 8 种重叠
  │     ├─ 计算 cuts[] 和 new_chapters[]
  │     └─ info['chapters'] = new_chapters   ← 改写
  ├─ remove_chapters() 调用 ffmpeg concat 剪切视频
  └─ info['duration'] 更新
          |
          v
post_process 阶段: FFmpegMetadataPP.run(info)
  ├─ _get_chapter_opts(info['chapters']) → *.meta
  ├─ _get_metadata_opts(info) → -metadata 选项
  └─ ffmpeg 重混，章节和元数据嵌入文件
          |
          v
输出文件（章节已嵌入，广告片段已剪切）
```

---

## 8. 关键设计要点

1. **阶段分离**：SponsorBlockPP 在 `after_filter`（下载前）获取数据，ModifyChaptersPP 在 `post_process`（下载后）执行剪切，避免不必要的下载。

2. **数据契约**：SponsorBlockPP 仅写入 `info['sponsorblock_chapters']`，ModifyChaptersPP 消费该字段并改写 `info['chapters']`，两者解耦。

3. **顺序保障**：ModifyChaptersPP 必须在 FFmpegMetadataPP **之前**运行（注释明确标注），因为剪切后的章节才是最终要嵌入的章节。

4. **不可删除类别**：`NON_SKIPPABLE_CATEGORIES`（`poi_highlight`、`chapter`）只能用于生成章节标记，不能被视频剪切。

5. **内部字段约定**：
   - `c['_categories']` — SponsorBlock 章节的原始类别列表（用于合并后追踪）
   - `c['remove']` — 标记该时间段需要被剪切
   - `c['cut_idx']` — 指向第一个落在该章节内的 cut 的索引
   - `c['_was_cut']` — 标记该章节是由切割产生的微小片段
