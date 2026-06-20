# yt-dlp 列表与分页抓取流程梳理

## 1. 总体架构概览

yt-dlp 的列表/分页抓取是一个**分层递归**的流程，核心围绕以下几个关键类和方法展开：

```
用户 URL
   │
   ▼
YoutubeDL.download()          ── 入口，遍历 URL 列表
   │
   ▼
YoutubeDL.extract_info()      ── 匹配合适的 Extractor
   │
   ▼
InfoExtractor.extract()       ── 调用 _real_extract()
   │
   ▼
InfoExtractor._real_extract() ── 各平台自定义实现，返回 info_dict
   │  返回 { _type: 'playlist'|'video'|'url'|..., entries: ... }
   ▼
YoutubeDL.process_ie_result() ── **核心调度器**，按 _type 分派
   │
   ├─ _type='video'          → process_video_result()  → 下载
   ├─ _type='url'            → 递归 extract_info()
   ├─ _type='url_transparent'→ 递归 + 合并元数据
   ├─ _type='playlist'       → __process_playlist()
   └─ _type='multi_video'    → __process_playlist()
```

核心代码位置（相对仓库根）：
- `yt_dlp/YoutubeDL.py` — 主流程调度：`download`, `extract_info`, `process_ie_result`, `__process_playlist`, `process_video_result`, `process_info`
- `yt_dlp/extractor/common.py` — Extractor 基类：`extract`, `_real_extract`
- `yt_dlp/utils/_utils.py` — 列表核心类：`PlaylistEntries`, `LazyList`, `PagedList`（含 `OnDemandPagedList`/`InAdvancePagedList`）
- `yt_dlp/extractor/youtube/_tab.py` — YouTube 分页：`YoutubeTabIE._entries`, `_extract_entries`, `_extract_video`

---

## 2. 主入口调用链

### 2.1 download() → extract_info()

**位置**：`YoutubeDL.download`（YoutubeDL.py L3694 附近）

```python
def download(self, url_list):
    for url in url_list:
        self.__download_wrapper(self.extract_info)(
            url, force_generic_extractor=self.params.get('force_generic_extractor', False))
```

`__download_wrapper` 是异常/进度包装器。

### 2.2 extract_info() — Extractor 匹配

**位置**：`YoutubeDL.extract_info`（YoutubeDL.py L1671 附近）

- 遍历所有已注册 Extractor，调用 `ie.suitable(url)` 匹配
- 成功后调用 `__extract_info(url, ie, download, extra_info, process)`

### 2.3 __extract_info() — 调用 Extractor

**位置**：`YoutubeDL.__extract_info`（YoutubeDL.py L1856 附近）

```python
@_handle_extraction_exceptions
def __extract_info(self, url, ie, download, extra_info, process):
    ie_result = ie.extract(url)                                    # ① Extractor 提取
    self.add_default_extra_info(ie_result, ie, url)               # ② 补字段
    if process:
        return self.process_ie_result(ie_result, download, extra_info)  # ③ 核心调度
```

`_handle_extraction_exceptions` 装饰器处理 GeoRestrictedError / ExtractorError / ReExtractInfo 等。

### 2.4 InfoExtractor.extract()

**位置**：`InfoExtractor.extract`（extractor/common.py L755 附近）

```python
def extract(self, url):
    for _ in range(2):
        try:
            self.initialize()
            ie_result = self._real_extract(url)   # 子类重写
            return ie_result
        except GeoRestrictedError:
            if self.__maybe_fake_ip_and_retry(...): continue
            raise
```

真正的提取逻辑在子类 `_real_extract(url)`。

---

## 3. process_ie_result() — 结果类型分派核心

**位置**：`YoutubeDL.process_ie_result`（YoutubeDL.py L1904 附近）

整个架构最关键的调度枢纽。按 `ie_result['_type']` 分派：

| _type 值 | 处理方式 |
|----------|---------|
| `'video'` (默认) | 调 `process_video_result()` → 进入下载 |
| `'url'` | **递归** `extract_info(url, ie_key)`，跳转提取 |
| `'url_transparent'` | 递归提取 + 合并当前元数据 |
| `'playlist'` / `'multi_video'` | 调 `__process_playlist()` 遍历条目 |
| `'compat_list'` | 兼容旧格式，逐条目递归 |

### 3.1 _type='url' 的展开

Playlist 中的每个视频条目典型结构：

```python
{
    '_type': 'url',
    'ie_key': 'Youtube',       # 指定使用哪个 Extractor
    'url': 'https://www.youtube.com/watch?v=xxxx',
    'id': 'xxxx', 'title': '...',
    # ... 浅层元数据，不含 formats
}
```

`process_ie_result` 看到后递归 `extract_info(url, ie_key)`，触发单个视频的完整提取。

### 3.2 extract_flat 的作用

`--flat-playlist`（`extract_flat=True`）时，对 `url` / `url_transparent` **不展开**，直接保留浅层条目，避免为每个视频再发 HTTP 请求。

---

## 4. 分页状态底层模型：LazyList / PagedList

### 4.1 LazyList — 惰性求值列表

**位置**：`LazyList`（utils/_utils.py L2217 附近）

```python
class LazyList(collections.abc.Sequence):
    def __init__(self, iterable, *, reverse=False, _cache=None):
        self._iterable = iter(iterable)   # 生成器，不立即消费
        self._cache = []                  # 已消费元素的缓存
        self._reversed = reverse
```

#### __getitem__ 的"只消费必要部分"逻辑

```python
def __getitem__(self, idx):
    start, stop, step = ...  # 解析 idx
    # 关键判断：负数索引、反向步进或无边界的正向，都必须全量消费
    if ((start or 0) < 0 or (stop or 0) < 0
            or (start is None and step < 0)
            or (stop is None and step > 0)):
        self._exhaust()          # ⚠️ 全量消费：会触发所有翻页请求
        return self._cache[idx]
    # 正向索引、有明确 stop 的情况：只消费到 max(start, stop) 为止
    n = max(start or 0, stop or 0) - len(self._cache) + 1
    if n > 0:
        self._cache.extend(itertools.islice(self._iterable, n))  # ✅ 按需消费
    return self._cache[idx]
```

**结论 1**：LazyList 要想"不翻多余的页"，必须满足 3 个条件：
- `start` ≥ 0（非负起点）
- `stop`  ≥ 0（非负终点，且不能省略/None）
- `step`  > 0（正向步进）

只要出现任何**负数索引、`[:]` 全切片、反向遍历**，LazyList 就会直接 `_exhaust()`，生成器全部跑完 → 所有翻页请求全部触发。

**结论 2**：对 LazyList 做**单个元素访问** `LazyList[i]`（`PlaylistEntries` 的默认工作方式）：
- 访问 i=0 → 消费到 `max(0,0)-len(cache)+1 = 1` 个 → 取第 0 个
- 访问 i=49 → 消费到 `max(49,49)-len(cache)+1 = 50-len(cache)` 个
- 即：**必须顺序消费到索引 i，不能跳过前面的元素**

> 对包装在 LazyList 中的**生成器型 entries**（如 YouTube `_entries()` 产出的生成器），这点非常关键——要拿到第 50 条，必须先让生成器 yield 出前 50 条，期间如果翻页到第 1 页，该请求还是要发的。

---

### 4.2 PagedList 族 — 按页码直接跳转

**位置**：`PagedList` 及其子类（utils/_utils.py L2306 附近）

两个子类：

| 类 | 总页数是否已知 | 翻页方式 | 典型场景 |
|---|---|---|---|
| `OnDemandPagedList` | 未知，直到某页 `< pagesize` 才停 | 用 `pagenum = idx // pagesize` 直接请求对应页 | 不知道总数的列表 API |
| `InAdvancePagedList` | 构造时已知 `pagecount` | 在 `[0, pagecount)` 范围内按页请求 | 总数已知的搜索结果 |

#### OnDemandPagedList._getslice — 真正的"跳过中间页"

```python
def _getslice(self, start, end):
    for pagenum in itertools.count(start // self._pagesize):
        firstid    = pagenum * self._pagesize
        nextfirstid = pagenum * self._pagesize + self._pagesize
        if start >= nextfirstid:
            continue                  # ✅ 本页完全落在 start 之前 → 跳过！不调 pagefunc
        # 计算本页内切片 startv:endv
        startv = (start % self._pagesize) if firstid <= start < nextfirstid else 0
        endv   = (((end-1) % self._pagesize)+1) if (end and firstid <= end <= nextfirstid) else None
        page_results = self.getpage(pagenum)        # 只调需要的页
        yield from page_results[startv:endv]
        if len(page_results) + startv < self._pagesize:  # 不满一页 → 到尾
            break
        if end == nextfirstid:                       # 到 end 了 → 停
            break
```

**结论 3**：当 `_entries` 底层是 `OnDemandPagedList` 时，切片 `[M:N]` 只请求 `[M//pagesize .. N//pagesize]` 范围内的页，**中间不需要的页全部跳过**。

例：pagesize=100，用户 `--playlist-items 150,450`（要第 150 和第 450 条）：
- 条目 150 → i=149 → pagenum = 149//100 = 1 → 请求 page 1
- 条目 450 → i=449 → pagenum = 449//100 = 4 → 请求 page 4
- **page 2、page 3 完全不请求**。

> 这也是 `LazyList` 和 `PagedList` 最大的行为差异：LazyList 是**顺序消费生成器**，而 PagedList 是**按页号随机访问**。YouTube 用的是生成器模式 → 包装在 LazyList 中 → 必须顺序 yield。

---

## 5. PlaylistEntries — 用户筛选与底层容器的桥接

**位置**：`PlaylistEntries`（utils/_utils.py L2416 附近）

`PlaylistEntries` 是 `__process_playlist` 直接操作的"门面"，统一屏蔽 `list` / 生成器 / `LazyList` / `PagedList` 四种底层来源。

### 5.1 构造时的类型分派

```python
def __init__(self, ydl, info_dict):
    entries = info_dict.get('entries')
    if isinstance(entries, list):
        self.is_exhausted = True               # list = 已在内存，不用翻页
    elif isinstance(entries, (list, PagedList, LazyList)):
        self._entries = entries                # 保持原样
    else:
        self._entries = LazyList(entries)      # ⭐ 生成器自动包成 LazyList
```

**关键点**：Extractor 返回 `entries=(generator)` 时，被自动包装成 **`LazyList(generator)`**。

### 5.2 parse_playlist_items() — start/end/items 解析

```python
PLAYLIST_ITEMS_RE = re.compile(r'''(?x)
    (?P<start>[+-]?\d+)?
    (?P<range>[:-]
        (?P<end>[+-]?\d+|inf(?:inite)?)?   # end 支持数字 / 负号 / inf / infinite
        (?::(?P<step>[+-]?\d+))?
    )?''')

@classmethod
def parse_playlist_items(cls, string):
    for segment in string.split(','):
        if not segment:
            raise ValueError('There is two or more consecutive commas')
        mobj = cls.PLAYLIST_ITEMS_RE.fullmatch(segment)
        if not mobj:
            raise ValueError(f'{segment!r} is not a valid specification')
        start, end, step, has_range = mobj.group('start', 'end', 'step', 'range')
        if int_or_none(step) == 0:
            raise ValueError(f'Step in {segment!r} cannot be zero')
        # 返回值保持用户原始 1-based 语义：start/end 不做 0-based 转换
        # 例：'3,5:10' → yield 3, slice(5, 10, None)
        #     '::-1'   → yield slice(None, None, -1)
        #     '1:inf:2' → yield slice(1, float('inf'), 2)
        if has_range:
            yield slice(int_or_none(start), float_or_none(end), int_or_none(step))
        else:
            yield int(start)
```

**⚠️ `--playlist-end -1` 的特殊兼容陷阱**（`get_requested_items` L2462 附近）：
```python
playlist_start = self.ydl.params.get('playliststart', 1)
playlist_end   = self.ydl.params.get('playlistend')
if playlist_end in (-1, None):      # -1 被历史兼容为"不设终点"，不是"倒数第 1 条"！
    playlist_end = ''
if not playlist_items:
    playlist_items = f'{playlist_start}:{playlist_end}'
```
- `--playlist-end -1` 实际等价于**不指定 playlist_end**（全量到尾），不是负索引。
- 要表达"倒数第 N 条"必须用 `--playlist-items '1:-5'`（通过 items 语法走负号解析），不能用 `--playlist-start/end`。

用户参数映射：
- `--playlist-start N --playlist-end M`  → 合成字符串 `f'{N}:{M}'` 再调用 parse_playlist_items
- `--playlist-items '1,3,5:10'`          → 直接解析；若同时指定了 start/end 会告警并忽略 start/end

**合法的 items 写法示例**（测试用例真实行为，以 10 条 `[1..10]` 为例）：

| items 字符串 | parse 结果（slice/int） | 真实选到的下标（1-based） | 说明 |
|---|---|---|---|
| `':'` / `'::1'` | `slice(None, None, 1)` | 1,2,3,4,5,6,7,8,9,10 | 全量 |
| `'::-1'` | `slice(None, None, -1)` | 10,9,8,7,6,5,4,3,2,1 | 反向全量，会触发 `len(self)` |
| `':6'` | `slice(None, 6, None)` | 1,2,3,4,5,6 | 前 6 条 |
| `':-6'` | `slice(None, -6, None)` | 1,2,3,4,5 | 等价于 Python `[:-5]`（注意偏移 +1），触发 `len(self)` |
| `'-1:6:-2'` | `slice(-1, 6, -2)` | 10,8,6 | 反向步长 2，从倒数第 1 条到第 6 条，触发 `len(self)` |
| `'9:-6:-2'` | `slice(9, -6, -2)` | 9,7,5 | 反向步长 2，触发 `len(self)` |
| `'1:inf:2'` / `'1:infinite:2'` | `slice(1, float('inf'), 2)` | 1,3,5,7,9 | 无限终点，正向步长 2，不触发 `len(self)` |
| `'-2:inf'` | `slice(-2, float('inf'), None)` | 9,10 | 倒数第 2 条到末尾，先触发 `len(self)` 算起点 |
| `':inf:-1'` | `slice(None, float('inf'), -1)` | 空 | 反向 + 终点=inf 永远不进入 frange |
| `'0-2:2'` | `slice(0, 2, 2)` | 2 | start=0 → `0-1=-1`，i=-1 被 `if i<0: continue` 跳过；下一个 i=-1+2=1 → `_getter(1)` → yield `(2, entry)` |
| `'1-:2'` | `slice(1, None, 2)` | 1,3,5,7,9 | 省略 end 的正步长区间写法 |
| `'0--2:2'` | `slice(0, -2, 2)` | 2,4,6,8 | 触发 `len(self)`，等价 `[1:-1:2]` |
| `'0'` | `0` | 空 | int(0) 经 `-1` 得 i=-1，被 `if i<0: continue` 跳过 |
| `'20'` | `20` | 空 | 超出范围 → `_getter` 抛 IndexError → break |

---

### 5.3 __getitem__ — 用户范围到实际索引的转换

`PlaylistEntries` 对用户的 1-based 输入到生成器产出 `(1-based_index, entry)` 做了 4 步转换：

```python
def __getitem__(self, idx):
    # ── Step 1：单 int 统一转成 slice，后续只用一套边界处理 ──
    if isinstance(idx, int):
        idx = slice(idx, idx)             # int(3) → slice(3, 3)
                                          # 这样 int 和 slice 走相同的 start/stop 计算

    # ── Step 2：step 默认值 ──
    step = 1 if idx.step is None else idx.step

    # ── Step 3：算 start（用户 1-based → 0-based） ──
    if idx.start is None:
        # 起点缺省：正向从头开始，反向从尾开始
        start = 0 if step > 0 else len(self) - 1   # ⚠️ 反向起点调 len(self) → 全量消费！
    else:
        # 正数：减 1 变 0-based；负数：len(self) + idx.start
        start = idx.start - 1 if idx.start >= 0 else len(self) + idx.start  # ⚠️ 负起点也调 len

    # ── Step 4：算 stop（用户 1-based → 0-based → frange 开区间） ──
    if idx.stop is None:
        # 终点缺省：反向到 0，正向到无穷
        stop = 0 if step < 0 else float('inf')
    else:
        # 正数：减 1 变 0-based；负数：len(self) + idx.stop
        stop = idx.stop - 1 if idx.stop >= 0 else len(self) + idx.stop   # ⚠️ 负终点也调 len
    # ⭐ 关键修正：用户想要的是"包含 stop"的闭区间，但 frange 是开区间 (<)
    #   step>0 → stop += 1 让 frange 包含用户的 stop
    #   step<0 → stop -= 1 让 frange 往负方向多走一格（stop 是较小的 0-based 索引）
    stop += [-1, 1][step > 0]

    # ── Step 5：frange 产出 0-based i，跳过负数，逐一取条目 ──
    for i in frange(start, stop, step):
        # frange 行为：while sign*start < sign*stop: yield start; start += step
        if i < 0:
            continue                        # 用户写 0 或很小的负数时跳过
        try:
            entry = self._getter(i)         # ⭐ 单个索引访问 → LazyList[i] / PagedList[i]
        except self.IndexError:
            self.is_exhausted = True
            if step > 0: break               # 正向：到尾就退出
            else:     continue              # 反向：越界跳过，继续尝试下一个 i
        yield i + 1, entry                   # ⭐ 对外再转回 1-based 下标
```

**边界示例（对照测试用例）**：
- 用户 `'3'`（单元素）→ int(3) → slice(3,3) → start=3-1=2, stop=3-1=2, step=1 → stop+=1=3 → frange(2,3,1) → `i=2` → yield `(3, entry)` ✅
- 用户 `'2-4'` → slice(2,4) → start=1, stop=4-1=3, step=1 → stop+=1=4 → frange(1,4,1) → `i=1,2,3` → yield `(2, entry), (3, entry), (4, entry)` 即 `[2,3,4]` ✅（与测试一致）
- 用户 `':-6'`（10 条列表）→ slice(None, -6) → start=0, stop=len+(-6)=4, step=1 → stop+=1=5 → frange(0,5,1) → `i=0..4` → yield `1..5` ✅（等价 Python `INDICES[:-5]`，比用户写的 -6 多 +1）
- 用户 `'::-1'`（反向全量）→ slice(None, None, -1) → `start=len(self)-1` ⚠️ 先全量消费 → start=9, stop=0, step=-1 → stop+=(-1)=-1 → frange(9,-1,-1) → `i=9,8..0` → yield `10,9..1` ✅

**关键观察（⭐）**：`PlaylistEntries` 对底层容器的访问方式是**"单个索引逐个访问"** —— 每次循环调 `_getter(i)` → 最终是 `self._entries[i]`（单个 int 索引），而**不是**一次性传入 slice。

这意味着：
- 对 **PagedList**：`PagedList[i]` 内部会走 `__getitem__` → 若传入 int 则调 `_getslice(i, i+1)`，仍然按页号精确跳页，没问题
- 对 **LazyList**：`LazyList[i]` 走单元素分支 → 只会消费到 `i`，顺序推进，但**不能跳过前面的元素**
- 对 **`idx.start` 为负 / `idx.stop` 为负 / step<0 且 start=None**：先调用 `len(self)` → `tuple(self[:])` → **全量消费生成器，全部翻页请求打完**

### 5.4 _getter — 单条获取的异常包装

```python
@functools.cached_property
def _getter(self):
    if isinstance(self._entries, list):
        # ── list 分支：直接索引，处理 MissingEntry ──
        def get_entry(i):
            try:
                entry = self._entries[i]
            except IndexError:
                entry = self.MissingEntry
                if not self.is_incomplete:
                    raise self.IndexError      # 越界 → 抛 IndexError 停止迭代
            if entry is self.MissingEntry:
                raise EntryNotInPlaylist(f'Entry {i + 1} cannot be found')
            return entry
    else:
        # ── 非 list 分支（LazyList / PagedList / 生成器）──
        def get_entry(i):
            try:
                # ⭐ 调用形式解析：
                #   type(self.ydl)                         → YoutubeDL 类
                #   ._handle_extraction_exceptions(...)    → 拿到装饰后的 wrapper 函数
                #   (self.ydl, i)                          → 传 self 作为 wrapper 的第一个参数 + i
                # 装饰器内部：wrapper(self, *args, **kwargs) → func(self, *args, **kwargs)
                # 也就是：func(self.ydl, i) → self._entries[i]
                #
                # LazyList.IndexError / PagedList.IndexError 在装饰器白名单内 → 原样 re-raise
                # ExtractorError（翻页 HTTP 失败等） → 装饰器吞掉 + report_error + 返回 None
                # 但 None 返回后会被外层 except 过滤，所以实际上只有白名单异常能出去
                return type(self.ydl)._handle_extraction_exceptions(
                    lambda _, i: self._entries[i])(self.ydl, i)
            except (LazyList.IndexError, PagedList.IndexError):
                # 白名单 re-raise 出来的，转成 PlaylistEntries.IndexError
                raise self.IndexError
            # ⚠️ 注意：如果 _handle_extraction_exceptions 吞掉了 ExtractorError 返回 None
            # 此时 get_entry(i) 返回 None → __getitem__ 会 yield (i+1, None)
            # 后续 __process_playlist 的 for 循环中 `if not entry: continue` 会跳过
            # 这和 IndexError 不同：IndexError 停止整个 __getitem__ 迭代，None 只是跳过
    return get_entry
```

**装饰器对 `_getter` 调用的影响**：

| 底层 `self._entries[i]` 产生的异常 | 装饰器处理 | `get_entry(i)` 的行为 | `__getitem__` 的行为 |
|---|---|---|---|
| `LazyList.IndexError` / `PagedList.IndexError` | 白名单 re-raise | `except` 转成 `self.IndexError` | 正向：`break` 停止迭代；反向：`continue` |
| `ExtractorError`（翻页失败） | `report_error` + 返回 None | 返回 None | `yield (i+1, None)` → for 循环 `if not entry: continue` |
| `DownloadCancelled` 子类 | 白名单 re-raise | 冒泡出 `get_entry` | 冒泡出 `__getitem__` → 冒泡出 `get_requested_items` 生成器 |
| 其它 `Exception` | `ignoreerrors=True` → 吞 + None；`False` → re-raise | 同上 | 同上 |

---

## 6. 范围筛选对翻页请求的实际影响（完整决策表）

综合上面几层，现在给出**用户参数 → 实际翻页请求行为**的完整映射。

假设：一个 1000 条、每页 100 条的播放列表，底层 `entries` 为 `LazyList(generator)`（YouTube 的典型情况）。

| 用户参数 | 内部 slice | 触发 `len(self)` 全量消费？ | 实际消费到的最大 0-based 索引 | 实际触发的页 | 说明 |
|---|---|---|---|---|---|
| （无筛选，全量） | `playliststart=1, playlistend=''` → parse `f'1:'` → `slice(1, None)` → start=0, stop=∞ | 否 | 999（迭代到 IndexError） | page 0 ~ 9（全部 10 页） | stop=∞ 不强制 exhaust，但会一直迭代直到耗尽 |
| `--playlist-start 1 --playlist-end 50` | `slice(1, 50)` → start=0, stop=50（经过 step>0 偏移） | 否 | 49 | page 0（前 50 条都在第一页） | ✅ 只请求第 0 页 |
| `--playlist-end 50` | 同上 | 否 | 49 | page 0 | ✅ 同 start=1 end=50 |
| `--playlist-end -1`（⚠️ 历史兼容） | `-1` 被转成空字符串 → `slice(1, None)` 等价全量 | 否 | 999（迭代到 IndexError） | page 0 ~ 9（全部） | ❗ `-1` **不是负索引**，被兼容为"不设终点" |
| `--playlist-items :-10`（负索引终点） | `slice(None, -10)` → `stop = len(self) + (-10)` 触发 len | ✅ 是 | 999（len 时全量） | page 0 ~ 9（全部） | ❌ 负 stop 触发 `len()` 直接全翻，翻完后再 slice 取前 990 条 |
| `--playlist-start 301 --playlist-end 350` | `slice(301, 350)` → start=300, stop=350 | 否 | 349 | page 0, 1, 2, 3（前 4 页） | ❌ **虽然只需要 301-350，但生成器必须顺序 yield 到 index 349，前面的 page 0/1/2 也要请求** |
| `--playlist-items 1,101,201`（离散项） | int(1) → slice(1,1); int(101)→slice(101,101); int(201)→slice(201,201) | 否 | 200（到第 3 段为止） | page 0, 1, 2 | ❌ 顺序消费，每段单独推进 LazyList，无法"跳回" |
| `--playlist-items 900:1000`（尾部范围） | `slice(900, 1000)` → start=899, stop=1000 | 否 | 999 | page 0 ~ 9（全部 10 页） | ❌ 生成器必须从头 yield 到 999 |
| `--playlist-items 0` | int(0) → slice(0,0) → start=-1 | 否 | -1 被 `if i<0: continue` 跳过 | 无请求 | 空结果，消费 0 条 |
| `--playlist-items 2000`（越界） | int(2000) → start=1999 → `_getter(1999)` 抛 IndexError → break | 否 | 999（消费到耗尽） | page 0 ~ 9（全部） | ❌ 生成器顺序推进到耗尽才知道越界，全部翻完 |
| `--playlist-items :::-1`（反向 step） | start 缺省且 step<0 → `start=len(self)-1` 触发 len | ✅ 是 | 999（len 时全量） | 全部页 | ❌ 反向必须先知道总长 |
| `--playlist-items 1:inf:2`（无限终点） | `slice(1, inf, 2)` → start=0, stop=∞, step=2 | 否 | 998（偶数索引） | page 0 ~ 9（全部 10 页） | ⚠️ inf 不触发 len，但仍会迭代到耗尽；如果步长大实际请求的页数相同 |
| `--playlist-items -2:inf`（负起点 + 无限终点） | start=-2 → `start = len(self) + (-2)` 触发 len | ✅ 是 | 999（len 时全量） | 全部页 | ❌ 负起点触发 len |

### 如果底层换成 OnDemandPagedList（行为完全不同）

| 用户参数 | 实际触发的页 |
|---|---|
| `--playlist-start 301 --playlist-end 350` | pagenum = 300//100=3 .. 349//100=3 → **只请求 page 3** ✅ |
| `--playlist-items 1,101,201` | int(1)→getslice(0,1)→page 0; int(101)→getslice(100,101)→page 1; int(201)→getslice(200,201)→page 2 → **3 次请求，不跨页浪费** ✅ |
| `--playlist-items 900:1000` | getslice(899,1000) → pagenum 8, 9 → **只请求 page 8, 9** ✅ |
| `--playlist-items :::-1` | 调 len() → 对 InAdvancePagedList 无害（已知 pagecount） |

**核心差异**：生成器型 entries（YouTube 式）受限于顺序消费，只有 PagedList 型 entries 能真正跳过中间页。

---

## 7. __process_playlist() — 消费时机与懒加载限制

**位置**：`YoutubeDL.__process_playlist`（YoutubeDL.py L2076 附近）

### 7.1 完整流程

```
__process_playlist(ie_result, download)
  │
  ├─ all_entries = PlaylistEntries(self, ie_result)
  │     → 生成器自动包装为 LazyList
  │
  ├─ entries = orderedSet(all_entries.get_requested_items(), lazy=True)
  │     → orderedSet(lazy=True) 返回去重生成器（不消费）
  │     → get_requested_items 产出 (1-based_index, entry)
  │
  ├─ lazy = params['lazy_playlist']
  │
  ├─ 非 lazy 分支（默认）：
  │     entries = resolved_entries = list(entries)   # ⚠️⭐ 这里一次性消费
  │     → get_requested_items 生成器跑完全程
  │     → 所有 _getter(i) 被调完 → 筛选范围内所有翻页请求全部发出
  │     n_entries = len(resolved_entries)            # 此时已知精确总数
  │
  ├─ lazy 分支：
  │     resolved_entries, n_entries = [], 'N/A'      # 总长未知，先占位
  │     ie_result['requested_entries'] = None        # 清空 infodict 中的浅引用
  │     ie_result['entries'] = None
  │     entries 还是生成器，留到下面 for 循环才消费
  │
  ├─ （写 playlist info.json / description / thumbnails）
  │
  ├─ ⚠️ playlistreverse / playlistrandom 分支：
  │     if lazy:
  │         report_warning("... not supported with lazy_playlist")
  │         # 不做任何操作，不会触发 exhaust
  │     elif playlistreverse:
  │         entries.reverse()                         # entries 已是 list，不触发
  │     elif playlistrandom:
  │         random.shuffle(entries)                   # entries 已是 list，不触发
  │
  └─ for i, (playlist_index, entry) in enumerate(entries):
        │
        ├─ lazy：resolved_entries.append(...)         # 边迭代边累计
        │
        ├─ entry_copy = ChainMap(entry + playlist_info + playlist_index)
        │
        ├─ _match_entry(entry_copy, incomplete=True) is not None
        │   → 被 match_filter/date/等过滤（非 break 情形）
        │   → resolved_entries[i] = NO_DEFAULT; continue
        │
        ├─ （打印 "Downloading item i of N"）
        │
        └─ __process_iterable_entry(entry, download, extra_info)
              └─ process_ie_result(...)                # 单条展开 / 下载
```

### 7.2 消费时机的 4 个关键分叉点

#### 分叉 1：orderedSet 的 lazy 参数

```python
entries = orderedSet(all_entries.get_requested_items(), lazy=True)
#                                    ^^^^^^^^^^^^ 始终为 True
```

`orderedSet(iterable, lazy=True)` 返回的是**去重生成器**（`_iter()`），不是 list。这一步本身不消费任何元素。

> 注意：即使非 lazy 模式，这里也传 `lazy=True`；真正的消费点在下一步 `list(entries)`。

#### 分叉 2：非 lazy = `list(entries)` 一次性消费（默认行为）

```python
entries = resolved_entries = list(entries)   # 非 lazy 的情况
```

这行代码的效果：
- `get_requested_items` 生成器被**完整遍历**
- 每个 `(index, entry)` 被取出 → `PlaylistEntries.__getitem__` → `_getter(i)` → 底层 `LazyList[i]`
- 对 YouTube 这种生成器型 entries：生成器**顺序推进**到用户范围的最大索引 → **该范围内所有页请求全部发出**

> 结论：默认（非 lazy）下，**在开始下载第一个视频之前**，播放列表范围内的所有"翻页"请求已经全部跑完并缓存到内存。用户看到的 "Downloading item X of Y" 的 Y 是精确的，因为此时已经知道总条数。

#### 分叉 3：get_requested_items 中的"预检查提前终止"（只在非 lazy 下生效）

```python
def get_requested_items(self):
    for index in self.parse_playlist_items(playlist_items):
        for i, entry in self[index]:
            yield i, entry
            if not entry: continue
            try:
                if not self.ydl.params.get('lazy_playlist'):   # ⭐ 只在非 lazy
                    self.ydl._match_entry(entry, incomplete=True, silent=True)
            except (ExistingVideoReached, RejectedVideoReached):
                return   # ⚡ 生成器提前 return，不再产生后续元素
```

- **非 lazy**：在 `list(entries)` 消费生成器的过程中，如果遇到
  - `--break-on-existing`（开启 `break_on_existing`）+ 命中 archive → 抛 `ExistingVideoReached` → 生成器结束
  - `--break-on-reject`（开启 `break_on_reject`）+ 被 match_filter/date 等 reject → 抛 `RejectedVideoReached` → 生成器结束
  
  **效果**：后面的条目不再 yield，连翻页请求都不会发。典型场景：按时间排序的频道视频，命中前一天就停，不用翻完整个频道。

- **lazy**：这层预检查整个被跳过（`if not lazy_playlist`），要到外层 for 循环才检查。

#### 分叉 4：for 循环中的提前终止（两种模式都有，但 lazy 更有价值）

```python
for i, (playlist_index, entry) in enumerate(entries):
    ...
    entry_result = self.__process_iterable_entry(...)
    if not entry_result:
        failures += 1
    if failures >= max_failures:        # --skip-playlist-after-errors N
        report_error(); break
    ...
```

- **非 lazy**：此时所有翻页请求已经打完，break 只是停止下载，不能减少翻页请求。
- **lazy**：break 时 entries 生成器没有消费完，后面的页请求**确实不会发**。这是 lazy 模式的核心价值。

---

### 7.3 强制全量消费的触发条件汇总

以下任一场景都会导致"用户以为选了范围，实际还是翻完全部/大部"：

| 触发条件 | 所在位置 | 真实行为 | 是否会额外触发全翻 |
|---|---|---|---|
| **负索引 start/stop** 如 `--playlist-items ':-5'` / `'2:-3'` | PlaylistEntries.__getitem__：`len(self) + idx.start/stop` | `len(self)` → `tuple(self[:])` 立即触发 LazyList 全量 exhaust | ❌ 是 |
| **反向 step** 如 `--playlist-items '::-1'` / `'5:1:-2'` | PlaylistEntries.__getitem__：start 缺省且 step<0 → `start = len(self) - 1` 调 `len(self)` | 同上，负起点也会先全量 | ❌ 是 |
| **内部全切片 `self[:]` / `--playlist-items ':'` 省略 stop** | LazyList.__getitem__：`stop is None and step > 0` 判断 → `_exhaust()` | LazyList 判定无法找到终点 → 全量消费 | ❌ 是 |
| **显式调用 `len(PlaylistEntries)`** | `__len__` = `len(tuple(self[:]))` | 直接全切片 self[:] → 全量 | ❌ 是 |
| **`--playlistreverse`（非 lazy，默认）** | __process_playlist L2122：`entries.reverse()` | 此时 entries 已经是 `list(entries)` 消费完的结果 → reverse 只是对内存 list 反转，**不额外触发翻页** | ⚠️ 翻页请求已在 `list(entries)` 时完成，不是 reverse 导致的 |
| **`--playlistreverse`（lazy 模式）** | __process_playlist L2120-2121：`report_warning` + 不做任何操作 | **不会触发任何翻页**，只是告警然后用原顺序继续 | ✅ 不会额外翻页 |
| **`--playlistrandom`（非 lazy）** | __process_playlist L2124：`random.shuffle(entries)` | 同上，shuffle 的是已经消费完的 list | ⚠️ 同上，翻页请求已在 `list(entries)` 时完成 |
| **`playlist_count` 推断**（InAdvancePagedList pagesize=1） | `get_full_count()` 直接返回 `_pagecount` | 不触发任何请求 | ✅ 无害（仅该类） |
| **`_playlist_infodict` 的 `__last_playlist_index`** | `max(ie_result.get('requested_entries') or (0,0))` | 只在已有 `requested_entries` 被回填后使用 | ✅ 不触发 |

> **关键澄清**：`--playlistreverse / --playlistrandom` **本身并不导致翻页**——真正触发全量翻页的是默认的非 lazy 模式（`list(entries)` 那行代码），reverse/shuffle 只是在结果上的后处理。如果用户同时开了 lazy 模式又加 reverse，根本不会 reverse（只会告警），也不会多翻页。

---

## 8. 单条任务的完整传递链

### 8.1 一个条目的典型生命周期（YouTube playlist → 单个视频下载）

```
① Extractor 产出 entry:        {_type:'url', ie_key:'Youtube', url:'...watch?v=xxxx', id, title, ...}
        │
        ▼
② process_ie_result 识别 _type='url' → extract_info(url, ie_key='Youtube')
        │  YoutubeDL.py L1958-1964
        ▼
③ YoutubeIE._real_extract() 真正解析视频 watch 页面
   → 返回 {_type:'video', formats:[...], subtitles, chapters, ...}
        │
        ▼
④ process_ie_result 识别 _type='video' → process_video_result(info_dict, download=True)
        │
        ├─ 4a. 字段校验/规范化、章节修正、缩略图补全
        ├─ 4b. 字幕筛选 process_subtitles()
        ├─ 4c. formats 清洗（URL、DRM、live 过滤）+ 排序 + 填充排序字段
        ├─ 4d. build_format_selector() → _select_formats() → formats_to_download
        │
        └─ 4e. for fmt, chapter in product(...):
                 new_info = info_dict + fmt + section_start/end
                 process_info(new_info)        # ⭐ 下载真正启动
                        │
                        ▼
⑤ process_info(info_dict)
   YoutubeDL.py L3331 附近
        │
        ├─ 5a. prepare_filename() 计算输出路径
        ├─ 5b. 写字幕/缩略图/info.json/快捷方式
        ├─ 5c. pre_process(before_dl) 钩子
        ├─ 5d. 跳过下载检测（existing_file / skip_download）
        │
        └─ 5e. fd = get_suitable_downloader(info_dict)
                 → 根据 protocol 返回：
                      HttpFD / HlsFD / DashFD / F4mFD / IsmFD /
                      RtmpFD / WebSocketFD / FragmentFD ...
                 fd.download(filename, info_dict)   # 字节级下载
        │
        ▼
⑥ post_process() → ffmpeg 合并/转码/嵌入缩略图/MoveFiles 等
```

### 8.2 process_info() 的断点：下载启动点

`get_suitable_downloader` 按 `info_dict['protocol']`（或 URL 推断 protocol）选择下载器：

- `http` / `https` → `HttpFD`（单请求或分片 Range 下载）
- `m3u8` / `m3u8_native` → `HlsFD`（基于 ffmpeg 或原生解析 TS）
- `dash` → `DashFD`（按 MPD manifest 分片下载）
- `f4m` → `F4mFD`（Adobe HDS）
- `rtmp*` → `RtmpFD`（调外部 rtmpdump）
- 等等...

下载器全部在 `yt_dlp/downloader/` 下。

---

## 9. YouTube 特定分页的推进细节

### 9.1 YoutubePlaylistIE 只是包装器

```python
# YoutubePlaylistIE._real_extract
def _real_extract(self, url):
    playlist_id = self._match_id(url)
    url = update_url_query('https://www.youtube.com/playlist',
                           parse_qs(url) or {'list': playlist_id})
    return self.url_result(url, ie=YoutubeTabIE.ie_key(), video_id=playlist_id)
```

它只是规范化 URL，然后返回 `_type='url'`，把实际工作交给 **YoutubeTabIE**。

### 9.2 YoutubeTabIE._entries() — 真正的循环翻页

**位置**：`YoutubeTabBaseInfoExtractor._entries`（extractor/youtube/_tab.py L560 附近）

```python
def _entries(self, tab, item_id, ytcfg, delegated_session_id, visitor_data):
    continuation_list = [None]                      # 用 list 实现闭包内回传
    extract_entries = lambda x: self._extract_entries(x, continuation_list)

    # ── 第一页：从初始响应 tab['content'] 提取 ──
    tab_content = try_get(tab, lambda x: x['content'], dict)
    parent_renderer = try_get(tab_content,
        lambda x: x['sectionListRenderer'], dict) or ...
    yield from extract_entries(parent_renderer)     # ⭐ 产出第一批条目
    continuation = continuation_list[0]             # 取第一页的 continuation token

    # ── 后续页：continuation API 循环 ──
    seen_continuations = set()                      # 防止 feed 循环
    for page_num in itertools.count(1):
        if not continuation: break
        token = continuation.get('continuation')
        if token in seen_continuations: break        # 检测到循环，退出
        seen_continuations.add(token)

        # ⭐ 发 API 请求，获取下一页 JSON
        headers = self.generate_api_headers(ytcfg=ytcfg, ...)
        response = self._extract_response(
            item_id=f'{item_id} page {page_num}',
            query=continuation, headers=headers, ytcfg=ytcfg,
            check_get_keys=('continuationContents',
                            'onResponseReceivedActions', ...))

        if not response: break
        continuation = None

        # 解析响应里的 appendContinuationItemsAction
        continuation_items = traverse_obj(response,
            (('onResponseReceivedActions',
              'onResponseReceivedEndpoints'), ...,
             'appendContinuationItemsAction', 'continuationItems'),
            'continuationContents', ...)

        continuation_item = traverse_obj(continuation_items, 0, ...)

        # ── 识别 renderer 类型，产出条目 ──
        known_renderers = {
            'playlistVideoRenderer':      (_playlist_entries,           'contents'),
            'gridVideoRenderer':          (_grid_entries,               'items'),
            'playlistVideoListContinuation': (_playlist_entries,        None),
            'gridContinuation':           (_grid_entries,               None),
            'sectionListContinuation':    (extract_entries,             None),
            # ... 10+ 种类型
        }
        for key in continuation_item:
            if key in known_renderers:
                func, parent_key = known_renderers[key]
                video_items_renderer = (
                    {parent_key: continuation_items} if parent_key
                    else continuation_items)
                continuation_list = [None]
                yield from func(video_items_renderer)    # ⭐ 产出本页条目
                continuation = (continuation_list[0]
                    or self._extract_continuation(video_items_renderer))

        continuation = continuation or self._extract_continuation(
            {'contents': [continuation_item]})

        if not continuation and not video_items_renderer:
            break
```

**关键点**：
- `continuation_list = [None]` 用 list 是因为**不可变对象无法在闭包中被赋值**，list 可以就地修改 `[0]`
- `_extract_entries()` 递归解析 renderer 树的同时把 token 回写到 `continuation_list[0]`
- `seen_continuations` 集合防止 YouTube 某些 feed 返回相同 token 导致**死循环**
- 循环结束条件有 4 个：① token 为空；② 检测到重复 token；③ API 空响应；④ 无 renderer 且无新 token

### 9.3 _extract_video() 只产出"引用条目"

```python
def _extract_video(self, renderer):
    # 从 renderer 提取 video_id, title, duration, channel 等浅层信息
    return {
        '_type': 'url',                        # 不是 video，是引用
        'ie_key': YoutubeIE.ie_key(),
        'id': video_id,
        'url': f'https://www.youtube.com/watch?v={video_id}',
        'title': title,
        'duration': duration,
        'thumbnails': self._extract_thumbnails(renderer, 'thumbnail'),
        ...
    }
```

> 没有 formats！没有实际的视频 URL！只有一个"待展开"的引用。展开要等 `process_ie_result` 递归时才做，那时才会请求 watch 页面、获取 formats。这就是为什么 `--flat-playlist` 可以快速列出 1000 个视频而不发 1000 次 watch 页面请求。

### 9.4 内联播放列表的特殊分页

watch 页面右侧的"正在播放"列表（inline playlist）使用 `next` API（不是 continuation），以最后一个 videoId + index 为锚点翻页（参见 `_extract_inline_playlist`）。

---

## 10. 提前终止的完整拦截点图与异常传播路径

### 10.1 异常继承链（先明确类层次，再谈装饰器行为）

```
Exception
  └─ YoutubeDLError (utils/_utils.py L968)
       ├─ ExtractorError (L980)
       │    └─ GeoRestrictedError (L1036)
       ├─ DownloadCancelled (L1102)        ← ⭐ 关键中间类
       │    ├─ ExistingVideoReached (L1107)   --break-on-existing 触发
       │    ├─ RejectedVideoReached (L1112)   --break-match-filter 触发
       │    └─ MaxDownloadsReached (L1117)    --max-downloads 触发
       ├─ ReExtractInfo (L1122)            → 重试循环
       └─ CookieLoadError
```

**关键事实**：`ExistingVideoReached`、`RejectedVideoReached`、`MaxDownloadsReached` **都继承自 `DownloadCancelled`**，不是直接继承 `Exception`。

### 10.2 _handle_extraction_exceptions 装饰器的真实行为

```python
def _handle_extraction_exceptions(func):
    @functools.wraps(func)
    def wrapper(self, *args, **kwargs):
        while True:
            try:
                return func(self, *args, **kwargs)
            except (CookieLoadError, DownloadCancelled,
                    LazyList.IndexError, PagedList.IndexError):
                raise                     # ⭐ 白名单：原样 re-raise，不 report_error
            except ReExtractInfo as e:
                ...; continue             # 重试：重新跑 while
            except GeoRestrictedError as e:
                self.report_error(msg)    # 吞掉 + report_error + break（函数返回 None）
            except ExtractorError as e:
                self.report_error(...)    # 吞掉 + report_error + break（函数返回 None）
            except Exception as e:
                if self.params.get('ignoreerrors'):
                    self.report_error(...)    # ⚠️ ignoreerrors 才吞，否则 re-raise
                else:
                    raise
            break
    return wrapper
```

**分类汇总**：

| 异常类型 | 继承链 | 装饰器匹配的 except 分支 | 行为 | 函数返回 |
|---|---|---|---|---|
| `LazyList.IndexError` | `IndexError` | 白名单 `except` | re-raise | — |
| `PagedList.IndexError` | `IndexError` | 白名单 `except` | re-raise | — |
| `DownloadCancelled` | `YoutubeDLError` | 白名单 `except` | re-raise | — |
| **`ExistingVideoReached`** | `DownloadCancelled` → `YoutubeDLError` | **白名单 `except`**（因为 `DownloadCancelled` 在白名单，子类也匹配） | **re-raise，不吞** | — |
| **`RejectedVideoReached`** | `DownloadCancelled` → `YoutubeDLError` | **白名单 `except`** | **re-raise，不吞** | — |
| **`MaxDownloadsReached`** | `DownloadCancelled` → `YoutubeDLError` | **白名单 `except`** | **re-raise，不吞** | — |
| `CookieLoadError` | `YoutubeDLError` | 白名单 `except` | re-raise | — |
| `ReExtractInfo` | `YoutubeDLError` | 专用 `except ReExtractInfo` | continue 重试 | — |
| `GeoRestrictedError` | `ExtractorError` → `YoutubeDLError` | `except ExtractorError`（父类先匹配） | report_error + break | None |
| `ExtractorError`（其它） | `YoutubeDLError` | `except ExtractorError` | report_error + break | None |
| 其它普通 `Exception` | — | `except Exception` | `ignoreerrors=True` → 吞 + None；`False` → re-raise | 视参数 |

**易错点**：`ExistingVideoReached` / `RejectedVideoReached` 不是普通 Exception 路径；它们继承 `DownloadCancelled`，会命中白名单并被 re-raise，`--ignore-errors` 对它们无效。

### 10.3 三层终止拦截点（按代码执行顺序）

```
用户开 --break-on-existing / --break-on-reject / --skip-playlist-after-errors N
                │
                ▼
 ┌─ __process_playlist() ──────────────────────────────────────────────────────┐
 │                                                                           │
 │  ① get_requested_items() 内的预检查（仅非 lazy 生效）                              │
 │  ┌───────────────────────────────────────────────────────────────┐        │
 │  │ for index in parse_playlist_items(playlist_items):                │        │
 │  │     for i, entry in self[index]:                              │        │
 │  │         yield i, entry                                        │        │
 │  │         if not entry: continue                                │        │
 │  │         if not lazy_playlist:                                  │        │
 │  │             try:                                              │        │
 │  │                 _match_entry(entry, incomplete=True, silent=True)│        │
 │  │             except (ExistingVideoReached,                      │        │
 │  │                     RejectedVideoReached):                        │        │
 │  │                 return  ← ⭐ 直接 return，结束整个生成器           │        │
 │  └───────────────────────────────────────────────────────────────┘        │
 │     ⚠️ 这段 try/except 独立于 _handle_extraction_exceptions                  │
 │        不经过装饰器，直接在生成器内部捕获 → 生成器静默结束                       │
 │     效果：后面条目不再 yield，后续翻页请求不发 ✅                             │
 │                                                                           │
 │  ② __process_playlist for 循环内（lazy + 非 lazy 都有）                           │
 │  ┌───────────────────────────────────────────────────────────────┐        │
 │  │ for i, (playlist_index, entry) in enumerate(entries):        │        │
 │  │     _match_entry(entry_copy, incomplete=True)                    │        │
 │  │     ├─ 返回 None → 不匹配 → resolved_entries[i] = NO_DEFAULT   │        │
 │  │     │  continue（不终止，继续下一条）                            │        │
 │  │     │                                                         │        │
 │  │     └─ 返回非 None 且 break_on_* 开启 → 抛异常                  │        │
 │  │        ExistingVideoReached / RejectedVideoReached              │        │
 │  │        ↑ 这些异常在 for 循环体里直接抛出（不在 __process_iterable_entry 内）│
 │  │        ↑ 没有被 _handle_extraction_exceptions 包裹               │        │
 │  │        ↑ 直接冒泡出 for 循环，终止整个 __process_playlist           │        │
 │  │                                                               │        │
 │  │     实际代码路径：                                              │        │
 │  │     if self._match_entry(entry_copy, incomplete=True) is not None:│      │
 │  │         resolved_entries[i] = (playlist_index, NO_DEFAULT)     │        │
 │  │         continue   ← ⭐ 注意！代码里是 continue，不是 break！     │        │
 │  │         _match_entry 内部如果开了 break_on_* 会直接 raise，       │        │
 │  │         异常冒泡出 for 循环，不是走 continue 这条线                 │        │
 │  │                                                               │        │
 │  │     entry_result = __process_iterable_entry(entry, ...)        │        │
 │  │     ↑ 此方法带 @_handle_extraction_exceptions                     │        │
 │  │     → 如果 __process_iterable_entry 内部产生 ExistingVideoReached    │        │
 │  │       （例如 extract_info 中 extract_info 的 archive 检查）         │        │
 │  │       → 装饰器白名单 re-raise → 冒泡出 for 循环                    │        │
 │  │     → 其它 ExtractorError → 装饰器吞掉返回 None → failures += 1    │        │
 │  │                                                               │        │
 │  │     if not entry_result: failures += 1                            │        │
 │  │     if failures >= max_failures: break 退出循环                    │        │
 │  └───────────────────────────────────────────────────────────────┘        │
 │                                                                           │
 │  ③ failures >= max_failures（--skip-playlist-after-errors N）                       │
 │     → report_error + break                                                    │
 │     → 非lazy：翻页请求已全部打完，只能省后续下载                                  │
 │     → lazy  ：生成器未消费完，✅ 翻页和下载都省                                 │
 │                                                                           │
 │  ④ MaxDownloadsReached（--max-downloads N）                                      │
 │     → process_info 中 check_max_downloads() 抛出                                    │
 │     → 继承 DownloadCancelled → 在 __process_iterable_entry 的装饰器白名单内     │
 │     → re-raise → 冒泡出 for 循环 → 终止 __process_playlist                      │
 │     → 不管 ignoreerrors 设置如何，MaxDownloadsReached 永远终止下载               │
 └───────────────────────────────────────────────────────────────────────────────┘
```

### 10.4 _match_entry() 的完整逻辑

```python
def _match_entry(self, info_dict, incomplete=False, silent=False):
    # ── 第一阶段：archive 检查 ──
    if self.in_download_archive(info_dict):
        reason = ''.join((
            format_field(info_dict, 'id', f'{self._format_screen("%s", self.Styles.ID)}: '),
            format_field(info_dict, 'title', f'{self._format_screen("%s", self.Styles.EMPHASIS)} '),
            'has already been recorded in the archive'))
        break_opt, break_err = 'break_on_existing', ExistingVideoReached
    else:
        # ── 第二阶段：match_filter / date / view_count / age 等检查 ──
        try:
            reason = check_filter()          # 内部调用 match_filter(info_dict)
        except DownloadCancelled as e:
            # match_filter 可以主动 raise DownloadCancelled(msg) 来中止
            reason, break_opt, break_err = e.msg, 'match_filter', type(e)
        else:
            break_opt, break_err = 'break_on_reject', RejectedVideoReached

    # ── 第三阶段：决定是否抛异常 ──
    if reason is not None:
        if not silent:
            self.to_screen('[download] ' + reason)
        if self.params.get(break_opt, False):    # ⭐ 只有用户显式开启 break_on_*
            raise break_err()                     #    才抛异常
    return reason    # 返回值非 None = "被过滤"，None = "通过"
```

关键点：
- 默认不开 `break_on_existing` / `break_on_reject`，`_match_entry` **只返回 reason**（非 None 表示"被过滤"）但**不抛异常** → for 循环里只是 `resolved_entries[i] = NO_DEFAULT; continue` 跳过，不会中断整个播放列表
- 只有显式开了相应的 `break_on_*` 参数才会抛 `ExistingVideoReached` / `RejectedVideoReached`
- `match_filter` 可以主动 `raise DownloadCancelled(msg)` → 此时 `break_opt = 'match_filter'`，`self.params.get('match_filter', False)` 返回 match_filter callable 本身（真值），所以 `raise break_err()` 必定执行 → 即 `raise type(e)()` → 抛出 `DownloadCancelled` 子类实例 → 终止下载
- `ExistingVideoReached` / `RejectedVideoReached` 继承自 `DownloadCancelled`，它们在 `_handle_extraction_exceptions` 的白名单里 → **永远 re-raise**，不受 `--ignore-errors` 影响

### 10.5 break_on_existing 的两层拦截对比

| 拦截层 | 代码位置 | 生效模式 | 异常如何抛出 | 异常如何被处理 | 能否省翻页请求 |
|---|---|---|---|---|---|
| ① 预检查 | `get_requested_items` 内独立 try/except | 仅非 lazy | `_match_entry` 直接 raise → 被生成器内 `except (ExistingVideoReached, RejectedVideoReached): return` 捕获 | 生成器静默结束（不冒泡到外层） | ✅ 生成器 return，后续翻页不发 |
| ② for 循环内 | `_match_entry(entry_copy)` 直接在循环体调用 | lazy + 非 lazy | `_match_entry` 直接 raise → 冒泡出 for 循环（没有 try 包裹） | 终止 `__process_playlist`（非 lazy 时翻页已打完 ❌；lazy 时 ✅ 可以省） | 非 lazy ❌ / lazy ✅ |
| ③ for 循环内（间接） | `__process_iterable_entry` 内的 `extract_info` → archive 检查 | lazy + 非 lazy | `extract_info` 中 L1712-1713：`raise ExistingVideoReached` → 装饰器白名单 re-raise | 同 ②，冒泡终止 | 非 lazy ❌ / lazy ✅ |
| ④ failures 计数 | `if not entry_result: failures += 1` | 都生效 | 不抛异常，只计失败数 | 到 `failures >= max_failures` 时 break | 非 lazy ❌ / lazy ✅ |

> **预检查存在的理由**：非 lazy 模式下 `list(entries)` 会先把范围内所有翻页请求打完，必须在生成器消费期提供一次提前终止机会。预检查的 `except (ExistingVideoReached, RejectedVideoReached): return` 直接在生成器内部静默结束，**不经过任何装饰器**，是最干净的终止方式——连 `__process_playlist` 都不会收到异常，只是发现生成器结束了。

---

## 11. 关键设计模式总结

| 机制 | 作用 | 关键位置 |
|------|------|----------|
| **生成器 + LazyList 顺序推进** | 分页请求在迭代时才发生，支持遍历中途终止 | `LazyList.__getitem__` + `_entries()` generator |
| **PagedList 按页随机访问** | 通过页号直接请求，跳过不需要的页 | `OnDemandPagedList._getslice` |
| **PlaylistEntries 门面层** | 统一 list/生成器/LazyList/PagedList 的差异，统一解析 items 语法 | `PlaylistEntries` |
| **分派递归** | `_type` 字段驱动 `process_ie_result` 递归处理嵌套/引用结构 | `process_ie_result` 的 _type 分支 |
| **闭包 list 回传** | YouTube 用 `continuation_list = [None]` 在 renderer 解析函数中回传翻页 token | `_extract_entries(continuation_list)` |
| **引用条目** | playlist 内的视频以 `_type='url'` 浅层返回，递归时才深度展开，`--flat-playlist` 据此止步 | `_extract_video()` + `ie_key` |
| **ChainMap 注入上下文** | playlist 元数据（playlist_index/playlist_title 等）用 `collections.ChainMap` 叠加到每个 entry，避免深拷贝 | `__process_playlist` |
| **双重循环防死循环** | `_playlist_urls`（playlist 级别）+ `seen_continuations`（YouTube API 级别） | `process_ie_result` / `_entries()` |
| **DownloadCancelled 继承链** | `ExistingVideoReached`/`RejectedVideoReached`/`MaxDownloadsReached` 继承 `DownloadCancelled`，在装饰器白名单内永远 re-raise，不受 `--ignore-errors` 影响 | `_handle_extraction_exceptions` 白名单 `except` |
| **预检查 + 循环内双拦截** | 生成器消费期（非 lazy 预检查）和下载循环期（for 内 continue/raise）各有一次终止机会 | `get_requested_items` 内 try/except + `__process_playlist` for 循环 |
