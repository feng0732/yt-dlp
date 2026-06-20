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
PLAYLIST_ITEMS_RE = r'''(?x)
    (?P<start>[+-]?\d+)?
    (?P<range>[:-]
        (?P<end>[+-]?\d+|inf(?:inite)?)?
        (?::(?P<step>[+-]?\d+))?
    )?'''

@classmethod
def parse_playlist_items(cls, string):
    for segment in string.split(','):
        # 解析为 int 或 slice(start, end, step)
        # 例：'3,5:10' → yield 3, slice(5,10,None)
        yield slice(int_or_none(start), float_or_none(end), int_or_none(step)) if has_range else int(start)
```

用户参数映射（`get_requested_items` L2462 附近）：
- `--playlist-start N --playlist-end M`  → 合成 `f'{N}:{M}'` 再解析
- `--playlist-items '1,3,5:10'`          → 直接解析，此时 start/end 被忽略并告警

### 5.3 __getitem__ — 用户范围到实际索引的转换

```python
def __getitem__(self, idx):
    if isinstance(idx, int):
        idx = slice(idx, idx)                 # 单数字也转成 slice，保持统一

    step = 1 if idx.step is None else idx.step
    # 用户下标从 1 开始，转成 0-based
    if idx.start is None:
        start = 0 if step > 0 else len(self) - 1   # ⚠️ len(self) → 全量消费！
    else:
        start = idx.start - 1 if idx.start >= 0 else len(self) + idx.start
    #
    if idx.stop is None:
        stop = 0 if step < 0 else float('inf')     # 没有上限 → 到无穷（一直取到出错）
    else:
        stop = idx.stop - 1 if idx.stop >= 0 else len(self) + idx.stop
    stop += [-1, 1][step > 0]

    for i in frange(start, stop, step):
        if i < 0: continue
        try:
            entry = self._getter(i)                 # ⭐ 逐次按单个索引访问
        except self.IndexError:
            self.is_exhausted = True; break if step>0 else continue
        yield i + 1, entry                           # 返回 (1-based 下标, entry)
```

**关键观察（⭐）**：`PlaylistEntries` 对底层容器的访问方式是**"单个索引逐个访问"** —— `_getter(i)` → `self._entries[i]`，而**不是**传入一个 slice。

这意味着：
- 对 **PagedList**：`PagedList[i]` 内部会调 `getslice(i, i+1)`，仍然按页号精确跳页，没问题
- 对 **LazyList**：`LazyList[i]` 走单元素分支 → 只会消费到 `i`，顺序推进，但**不能跳**
- 对 **`idx.start` 为负 / `idx.stop` 为负 / 需要 `len(self)`**：先调用 `__len__` → `tuple(self[:])` → **全量消费生成器，全部翻页请求打完**

### 5.4 _getter — 单条获取的异常包装

```python
@functools.cached_property
def _getter(self):
    if isinstance(self._entries, list):
        def get_entry(i): ...  # 直接索引
    else:
        def get_entry(i):
            try:
                # 用 _handle_extraction_exceptions 包一层，单条失败不中断
                return type(self.ydl)._handle_extraction_exceptions(
                    lambda _, i: self._entries[i])(self.ydl, i)
            except (LazyList.IndexError, PagedList.IndexError):
                raise self.IndexError
```

每个单独索引的访问都带异常处理，确保某一页请求失败时可以按 `--ignore-errors` 策略处理。

---

## 6. 范围筛选对翻页请求的实际影响（完整决策表）

综合上面几层，现在可以给出**用户参数 → 实际翻页请求行为**的完整映射。

假设：一个 1000 条、每页 100 条的播放列表，底层 `entries` 为 `LazyList(generator)`（YouTube 的典型情况）。

| 用户参数 | PlaylistEntries 切片 | 底层访问模式 | 实际触发的页 | 说明 |
|---|---|---|---|---|
| （无筛选，全量） | `[1:]` → start=0, stop=∞ | `_getter(0), _getter(1)...` 直到 IndexError | page 0 ~ page 9（全部 10 页） | stop=∞ 不会强制 exhaust，但会一直迭代到尾 |
| `--playlist-start 1 --playlist-end 50` | `[1:50]` → start=0, stop=49 | `_getter(0..49)` → LazyList 消费到 index 49 | page 0（前 50 条都在第一页） | ✅ 只请求第 0 页 |
| `--playlist-start 301 --playlist-end 350` | `[301:350]` → start=300, stop=349 | `_getter(300..349)` → LazyList **顺序消费 0..349** | page 0, 1, 2, 3（前 4 页） | ❌ **虽然只需要第 301-350 条，但因为是生成器顺序推进，前面的 page 0/1/2 也必须全部请求完** |
| `--playlist-items 1,101,201`（离散项） | `int(1), int(101), int(201)` → `slice(1,1), slice(101,101), slice(201,201)` | `_getter(0)`, `_getter(100)`, `_getter(200)` | page 0, 1, 2 | ❌ 同样顺序消费，每跳到一个新索引都补消费中间部分 |
| `--playlist-items 900:1000`（尾部范围） | `[900:1000]` → start=899, stop=999 | `_getter(899..999)` → 消费全部 0..999 | page 0 ~ 9（全部 10 页） | ❌ 生成器必须从头 yield |
| `--playlist-end -10`（负索引终点） | 解析时遇到负 stop → `len(self)+idx.stop` → 调 `__len__` | `__len__` → `tuple(self[:])` → **立即全量消费** | page 0 ~ 9（全部） | ❌ 负索引触发 `len()`，直接全部翻完 |
| `--playlist-items '::-1'`（反向 step） | `start=None, stop=None, step=-1` → 看 L2521-2522：`start=len(self)-1` → 又调 `__len__` | `__len__` → 全量消费 | 全部页 | ❌ 反向遍历必须先知道总长 |

### 如果底层换成 OnDemandPagedList（行为完全不同）

| 用户参数 | 实际触发的页 |
|---|---|
| `--playlist-start 301 --playlist-end 350` | pagenum = 300//100=3 .. 349//100=3 → **只请求 page 3** ✅ |
| `--playlist-items 1,101,201` | page 0, page 1, page 2 → **3 次请求，中间不跨页浪费** ✅ |

**所以**：生成器型 entries（YouTube 式）受限于顺序消费，只有 PagedList 型 entries 能真正跳过中间页。

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
  │     → orderedSet 在 lazy=True 时返回去重生成器（不消费）
  │     → get_requested_items 产出 (1-based_index, entry)
  │
  ├─ lazy = params['lazy_playlist']
  │
  ├─ 非 lazy 分支（默认）：
  │     entries = resolved_entries = list(entries)   # ⚠️⭐ 这里一次性消费
  │     → 触发 get_requested_items 生成器跑完
  │     → 触发所有 _getter(i) → 所有范围内翻页请求打完
  │     n_entries = len(resolved_entries)            # 此时可以显示总数
  │
  ├─ lazy 分支：
  │     resolved_entries, n_entries = [], 'N/A'      # 总长未知
  │     entries 还是生成器，留到 for 循环才消费
  │
  ├─ （写 playlist info.json 等文件）
  │
  ├─ 非 lazy：playlistreverse / playlistrandom 可排序（已有全量 list）
  │   lazy：  这两个选项告警不支持
  │
  └─ for i, (playlist_index, entry) in enumerate(entries):
        │
        ├─ lazy：resolved_entries.append(...)   # 边迭代边追加
        │
        ├─ entry_copy = ChainMap(entry + playlist_info + playlist_index)
        │
        ├─ _match_entry(entry_copy, incomplete=True)    # 日期/标题/观看量 过滤
        │   → 不匹配但没开 break_on_reject → 只是跳过，不 break
        │   → 开了 break_on_reject + reject 命中 → 抛 RejectedVideoReached → 中止
        │
        ├─ （打印 "Downloading item i of N"）
        │
        └─ __process_iterable_entry(entry, download, extra_info)
              └─ process_ie_result(...)
                    └─ （单条展开 / 下载）
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

| 触发条件 | 所在位置 | 后果 |
|---|---|---|
| **`--playlistreverse`** | __process_playlist L2122-2123：`entries.reverse()`（要求非 lazy 并已有 list） + LazyList 反向步进 | 非 lazy：已经 list 但本来就是全量；LazyList 单独使用时 reverse 会 exhaust |
| **`--playlistrandom`** | __process_playlist L2124-2125：`random.shuffle(entries)` | shuffle 需要 list → 全量消费 |
| **负索引** 如 `--playlist-end -5` | PlaylistEntries.__getitem__ L2524, L2530 调 `len(self)` → `__len__` → `tuple(self[:])` | LazyList 全量 exhaust |
| **反向 step** 如 `--playlist-items '::-2'` | 同上，需要先算 `len(self)` 找起点 | LazyList 全量 exhaust |
| **`--playlist-items '[:]'` / 省略 end** | LazyList.__getitem__ L2265 判断 `stop is None and step > 0` → `_exhaust()` | LazyList 全量 exhaust |
| **调用 `len(PlaylistEntries)`** | `__len__` = `len(tuple(self[:]))` | 全量 exhaust |
| **`playlist_count` 推断** 对 InAdvancePagedList pagesize=1 | get_full_count 直接返回 `_pagecount` | 无害，不触发请求（只对该类生效） |
| **`_playlist_infodict` 的 `__last_playlist_index`** | L2071：`max(ie_result.get('requested_entries') or (0, 0))` | 只在已经有 requested_entries 时用，不触发请求 |

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

## 10. 提前终止的完整拦截点图

```
用户开启 --break-on-existing / --break-on-reject / --skip-playlist-after-errors N
         │
         ▼
 ┌─ __process_playlist() ────────────────────────────────────────────────┐
 │                                                                        │
 │  ① get_requested_items()（只在 非lazy）                                 │
 │     _match_entry(entry, incomplete=True, silent=True)                  │
 │     ├─ 命中 archive 且开 break_on_existing → ExistingVideoReached      │
 │     └─ 被 reject 且开 break_on_reject  → RejectedVideoReached          │
 │       → 生成器 return，后面的翻页请求都不发 ✅ （仅非lazy）              │
 │                                                                        │
 │  ② __process_playlist for 循环内（lazy + 非lazy 都有）                  │
 │     _match_entry(entry_copy, incomplete=True)                          │
 │     命中 → resolved_entries[i] = NO_DEFAULT（跳过，不处理）             │
 │     如果 break_on_* → 抛异常 → 被 _handle_extraction_exceptions 捕获   │
 │       → __process_iterable_entry 返回 None → failures += 1             │
 │                                                                        │
 │  ③ failures >= max_failures（--skip-playlist-after-errors N）          │
 │       → break 退出循环                                                 │
 │       → 非lazy：翻页请求已全部打完，只能省后面的下载                    │
 │       → lazy   ：entries 生成器未消费完，✅ 翻页和下载都省              │
 │                                                                        │
 │  ④ MaxDownloadsReached（--max-downloads N）                            │
 │       → process_info 中 check_max_downloads() 抛出                    │
 │       → 向上冒泡，外层 for 循环捕获并 break                             │
 └────────────────────────────────────────────────────────────────────────┘
```

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
| **ChainMap 注入上下文** | playlist 元数据（playlist_index/playlist_title 等）用 `collections.ChainMap` 叠加到每个 entry，避免深拷贝 | `__process_playlist` L2148 |
| **双重循环防死循环** | `_playlist_urls`（playlist 级别）+ `seen_continuations`（YouTube API 级别） | `process_ie_result` / `_entries()` |
| **两处 break 拦截** | 生成器消费期（非 lazy）和 下载循环期（lazy）各有一次 break 机会，lazy 下能省后续翻页请求 | `get_requested_items` + `__process_playlist` for |
