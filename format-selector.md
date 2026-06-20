# yt-dlp 格式选择器代码实现分析

核心代码位于 `yt_dlp/YoutubeDL.py` 的 `build_format_selector(format_spec)` 方法。
该方法将格式选择表达式字符串解析为一棵选择器树，再编译为嵌套函数，最终以生成器模式产出候选格式。

---

## 一、整体架构：解析 → 编译 → 执行

```
用户输入: "bv*[ext=mp4]+ba[ext=m4a]/b[ext=mp4]"
    ↓
1. Tokenize (tokenize 模块 + _remove_unused_ops 清洗)
    ↓
2. Parse (_parse_format_selection 递归下降) → 选择器树 (list of FormatSelector)
    ↓
3. Compile (_build_selector_function 递归编译) → 嵌套闭包函数
    ↓
4. Execute (_select_formats 传入 ctx 字典) → 产出格式列表
```

### 上下文字典 `ctx`

```python
ctx = {
    'formats': [...],                    # 所有可用格式
    'has_merged_format': bool,           # 是否存在音视频合并格式
    'incomplete_formats': bool,          # 是否只有纯音频或纯视频格式
}
```

构建于 `YoutubeDL.py` 的 `_select_formats` 方法。

---

## 二、Token 清洗：`_remove_unused_ops`

将 token 流中的无用运算符（如 `-`）与相邻字符串拼接，保留的结构运算符仅有：

| 运算符 | 含义 |
|--------|------|
| `/` | 候选回退（PICKFIRST） |
| `+` | 合并（MERGE） |
| `,` | 多选（列表） |
| `(` `)` | 分组（GROUP） |
| `[` `]` | 过滤器边界 |

例如 `mp4-baseline-16x9` 中的 `-` 不是结构运算符，会被拼接为一个 NAME token `mp4-baseline-16x9`。

---

## 三、解析：`_parse_format_selection` 递归下降

### 3.1 选择器类型 `FormatSelector`

```python
FormatSelector = namedtuple('FormatSelector', ['type', 'selector', 'filters'])
```

| type | selector | 含义 |
|------|----------|------|
| `SINGLE` | 格式名如 `'best'`, `'mp4'`, `'137'`, `'all'` | 原子选择器 |
| `PICKFIRST` | `(first, second)` 元组 | `/` 回退 |
| `MERGE` | `(selector_1, selector_2)` 元组 | `+` 合并 |
| `GROUP` | 子选择器列表 | `()` 分组 |

### 3.2 运算符优先级（由解析函数的递归结构决定）

解析并非传统运算符优先级文法，而是通过**递归调用入口不同**实现层级：

| 优先级 | 运算符 | 解析入口 | 说明 |
|--------|--------|----------|------|
| 最低 | `,` | 顶层 `_parse_format_selection` | 分隔多个独立选择器 |
| 低 | `/` | 顶层，递归调用 `inside_choice=True` | 候选回退，右结合 |
| 中 | `+` | 递归调用 `inside_merge=True` | 合并，**右侧递归嵌套** |
| 高 | `[...]` | 附加到当前选择器 | 过滤器 |
| 最高 | `()` | 递归调用 `inside_group=True` | 分组 |

**关键机制**：当内层解析遇到不该由自己处理的运算符时，调用 `tokens.restore_last_token()` 回退 token，
交由外层处理。

例如解析 `bv+ba/w`：
1. 顶层遇到 `bv` → 构造 `SINGLE('bv', [])`
2. 遇到 `+` → 递归 `inside_merge=True` 解析右侧
3. 在 merge 上下文中遇到 `ba` → 构造 `SINGLE('ba', [])`
4. 在 merge 上下文中遇到 `/` → **回退 token**，break 退出 merge 递归
5. 回到顶层，`+` 右侧得到 `SINGLE('ba', [])`，构造 `MERGE((bv, ba), [])`
6. 顶层遇到 `/` → 递归 `inside_choice=True` 解析右侧
7. 遇到 `w` → 构造 `SINGLE('w', [])`
8. 最终：`PICKFIRST(MERGE(bv,ba), SINGLE(w), [])`

### 3.3 `+` 的右侧递归嵌套细节

`+` 采用右侧递归嵌套（右结合），而非左结合。解析 `a+b+c` 的过程：

1. 顶层遇到 `a` → `current_selector = SINGLE('a')`
2. 遇到 `+` → `selector_1 = SINGLE('a')`，递归调用 `_parse_format_selection(tokens, inside_merge=True)` 解析右侧
3. 递归内遇到 `b` → `current_selector = SINGLE('b')`
4. 递归内遇到 `+` → `selector_1 = SINGLE('b')`，再次递归调用 `_parse_format_selection(tokens, inside_merge=True)`
5. 第二次递归内遇到 `c` → `current_selector = SINGLE('c')`
6. 第二次递归返回 `[SINGLE('c')]`
7. 第一层递归构造 `MERGE(SINGLE('b'), [SINGLE('c')])`，返回 `[MERGE(...)]`
8. 顶层构造 `MERGE(SINGLE('a'), [MERGE(SINGLE('b'), [SINGLE('c')])])`

最终结构（伪代码）：`MERGE(a, [MERGE(b, [c])])`。

**编译后的执行顺序**：
1. 先递归计算右侧：`_merge(b, c)` → 得到合并结果 `bc`，其 `requested_formats = [b, c]`
2. 再做外层合并：`_merge(a, bc)` → 第一个参数是左侧 `a`，第二个参数是右侧合并结果 `bc`

这个顺序会影响 `_merge` 内部 `formats_info` 的展开顺序：
```python
# _merge(a, bc) 中：
formats_info = a.requested_formats + bc.requested_formats
#              = [a]                    + [b, c]
#              = [a, b, c]
```

即最终合并格式的 `requested_formats` 列表按"从左到右"的顺序展开：**越靠左的选择器，其格式在列表中越靠前**。
这一点很重要，因为多流策略关闭时 `_merge` 会按遍历顺序**保留先遇到的流**，列表顺序直接决定了哪条流会被留下。

### 3.4 递归入口的约束

| 递归参数 | 遇到哪个运算符时回退并退出 |
|-----------|---------------------------|
| `inside_merge=True` | 遇到 `/` 或 `,` → 回退并退出 |
| `inside_choice=True` | 遇到 `,` → 回退并退出 |
| `inside_group=True` | 遇到 `)` → 退出（不回退） |

这意味着 `+` 的绑定力强于 `/`，`/` 的绑定力强于 `,`。

---

## 四、候选回退机制：`/` (PICKFIRST)

```python
def selector_function(ctx):
    for f in fs:
        picked_formats = list(f(ctx))
        if picked_formats:
            return picked_formats
    return []
```

**逻辑**：按从左到右顺序，逐一求值候选分支；第一个产出非空结果的分支直接返回，后续分支不再求值。
这是典型的**短路回退**（short-circuit fallback）。

### 示例

表达式 `bv*[ext=mp4]+ba[ext=m4a]/b[ext=mp4]/bv*+ba/b` 是一个三层嵌套的二选一结构。
虽然代码上是嵌套二叉树，但从效果上看等价于按顺序的回退链：

1. 先尝试 `bv*[ext=mp4]+ba[ext=m4a]` — mp4 视频 + m4a 音频合并
2. 失败则回退到 `b[ext=mp4]` — 预合并的 mp4 格式
3. 再失败则回退到 `bv*+ba` — 任意最佳视频 + 任意最佳音频
4. 最后回退到 `b` — 任意最佳预合并格式

### 右结合性（嵌套二选一结构）

`a/b/c` 被解析为 `PICKFIRST(a, PICKFIRST(b, PICKFIRST(c, ...)))` —— 即每次都是一个二选一，第二个选项本身可以是另一个二选一。

结构示意（嵌套二叉树）：
```
PICKFIRST
├── a
└── PICKFIRST
    ├── b
    └── c
```

执行流程：
1. 先尝试最外层左分支 `a` → 有结果直接返回
2. 失败 → 进入外层右分支（本身又是一个 PICKFIRST）
3. 尝试内层左分支 `b` → 有结果直接返回
4. 失败 → 进入内层右分支 `c`
5. 有结果返回，没有结果则最终返回空

从效果上看等价于"按顺序从左到右逐一尝试，取第一个成功的"，但代码结构上是**逐层嵌套的二选一**，而非扁平的多分支列表。

---

## 五、合并机制：`+` (MERGE)

```python
def selector_function(ctx):
    for pair in itertools.product(selector_1(ctx), selector_2(ctx)):
        yield _merge(pair)
```

**逻辑**：对左右两个选择器产出的格式做笛卡尔积，每一对都合并为一个新格式。
通常左右各只产出一个格式（如 `bv+ba`），此时只有一对合并结果。

### `_merge` 合并细节

1. 展开两边的 `requested_formats`（支持已合并格式的再合并），合并后的顺序为 `format_1.requested_formats + format_2.requested_formats`
2. 根据多流策略过滤：
   - 若不允许同类型多流（默认），按遍历顺序**保留先遇到的视频流和先遇到的音频流**，后续同类型流被丢弃
   - 剔除 `acodec==vcodec=='none'` 的无效格式
3. 计算兼容输出扩展名（基于编解码器和 `merge_output_format` 参数）
4. 构建合并字典，包含 `format_id`、`ext`、`protocol`、`language`、`filesize_approx`、`tbr` 等字段
5. 若仅有一个视频流，继承其视频属性（width, height, fps, vcodec 等）
6. 若仅有一个音频流，继承其音频属性（acodec, abr, asr 等）

---

## 六、多选机制：`,` (列表)

```python
def selector_function(ctx):
    for f in fs:
        yield from f(ctx)
```

**逻辑**：依次执行每个子选择器，将所有产出格式平铺输出。
与 `/` 不同，`,` 不做短路，所有分支的结果都会保留。

### 示例

`bv,ba` → 分别选出最佳纯视频和最佳纯音频，产出两个格式（不合并），用于独立下载。

---

## 七、原子选择器：SINGLE

### 7.1 格式名分类

| 输入 | 匹配方式 | 示例 |
|------|----------|------|
| `all` | 直接关键字匹配，输出全部格式 | `all` |
| `mergeall` | 直接关键字匹配，合并全部有效格式 | `mergeall` |
| `best`/`worst`/`b`/`w` + 可选修饰 | 正则 `r'(?P<bw>best\|worst\|b\|w)(?P<type>video\|audio\|v\|a)?(?P<mod>\*)?(?:\.(?P<n>[1-9]\d*))?$'` | `bv`, `ba*`, `worst.2`, `bv.2` |
| 音频扩展名 | `_format_selection_exts['audio']` | `mp3`, `m4a`, `flac` 等 |
| 视频扩展名 | `_format_selection_exts['video']` | `mp4`, `webm`, `mkv` 等 |
| Storyboard 扩展名 | `_format_selection_exts['storyboards']` | `mhtml` 等 |
| 其他 | 按 `format_id` 精确匹配 | `137`, `251` |

### 7.2 特殊原子：`all`

```python
if format_spec == 'all':
    def selector_function(ctx):
        yield from _check_formats(ctx['formats'][::-1])
```

**行为**：
1. 直接对 `ctx['formats']` 进行倒序（`[::-1]`）——因为 extractor 原本按"最差→最优"排列，倒序后变为"最优→最差"
2. 逐个通过 `_check_formats` 做可用性检查
3. **产出所有可用格式**，数量为全部可用格式的总数

**与过滤器的组合**：`all[ext=mp4]` 先由过滤器筛选出 ext=mp4 的格式，再全部倒序产出。

### 7.3 特殊原子：`mergeall`

```python
elif format_spec == 'mergeall':
    def selector_function(ctx):
        formats = list(_check_formats(
            f for f in ctx['formats']
            if f.get('vcodec') != 'none' or f.get('acodec') != 'none'))
        if not formats:
            return
        merged_format = formats[-1]               # 取最优格式作为起点
        for f in formats[-2::-1]:                 # 剩余格式从次优到最差依次合并
            merged_format = _merge((merged_format, f))
        yield merged_format                       # 最终只产出一个超级合并格式
```

**行为**：
1. 先过滤掉 `acodec==none 且 vcodec==none` 的 storyboard 等无效格式
2. `ctx['formats']` 是按"最差→最优"排序，所以 `formats[-1]` 是最优格式
3. 以最优格式为起点，从次优到最差依次调用 `_merge` 层层合并
4. 最终只产出**一个合并结果**
5. 实际保留的流数量由多流策略决定：
   - **双多流都开启**：`--video-multistreams` 且 `--audio-multistreams` → 包含**所有**视频流和音频流
   - **仅开视频多流**：保留所有视频流，仅保留第一个音频流
   - **仅开音频多流**：保留所有音频流，仅保留第一个视频流
   - **都不开启（默认）**：仅保留先遇到的一个视频流和先遇到的一个音频流

**多流关闭时 `_merge` 的流选择细节**：
```python
# _merge 内部：
formats_info = format_1.requested_formats + format_2.requested_formats  # 按合并顺序排列
get_no_more = {'video': False, 'audio': False}
for fmt_info in formats_info:
    if fmt_info 有视频流且 get_no_more['video']: 丢弃
    elif fmt_info 有视频流且 !get_no_more['video']: 保留, get_no_more['video'] = True
    if fmt_info 有音频流且 get_no_more['audio']: 丢弃
    elif fmt_info 有音频流且 !get_no_more['audio']: 保留, get_no_more['audio'] = True
```

由于 mergeall 按"最优→次优→...→最差"的合并顺序，`formats_info` 中质量较高的流排在前面，因此先遇到的也就是质量较高的视频流和质量较高的音频流会被保留。但代码并未主动"挑选最优"，而是被动地保留遍历顺序中首次出现的流。

**典型用法**：`bv*+mergeall[vcodec=none]` + `--audio-multistreams` = 最佳含视频格式 + 合并所有纯音频格式（多音轨）。

### 7.4 best/worst 系列的过滤逻辑

```
格式:    {bw}{type?}{mod?}{.n?}
           ↑    ↑     ↑    ↑
         best/worst  v/a  *    .1 .2 .3...
```

| 写法 | format_type | format_modified | 过滤条件 | 含义 |
|------|-------------|-----------------|----------|------|
| `b` | None | False | vcodec≠none **AND** acodec≠none | 最佳预合并格式（音视频俱全） |
| `w` | None | False | vcodec≠none **AND** acodec≠none | 最差预合并格式（音视频俱全） |
| `bv` | `'v'` | False | **acodec==none** | 最佳**纯视频**（不含音频流） |
| `ba` | `'a'` | False | **vcodec==none** | 最佳**纯音频**（不含视频流） |
| `wv` | `'v'` | False | **acodec==none** | 最差纯视频 |
| `wa` | `'a'` | False | **vcodec==none** | 最差纯音频 |
| `b*` | None | True | True（不过滤） | 最佳任意格式 |
| `w*` | None | True | True（不过滤） | 最差任意格式 |
| `bv*` | `'v'` | True | **vcodec≠none** | 最佳含视频格式（可含音频） |
| `ba*` | `'a'` | True | **acodec≠none** | 最佳含音频格式（可含视频） |
| `wv*` | `'v'` | True | **vcodec≠none** | 最差含视频格式 |
| `wa*` | `'a'` | True | **acodec≠none** | 最差含音频格式 |

**`bv` vs `bv*` 的本质区别**：
- `bv`（无星号，type=v，modified=False）→ 反向条件：**必须没有音频流**（`acodec == 'none'`），即视频流独立封装的纯视频轨道
- `bv*`（有星号，type=v，modified=True）→ 正向条件：**只要有视频流就行**（`vcodec != 'none'`），包含预合并格式和纯视频轨道

代码中这一分支通过三元选择链实现：
```python
_filter_f = (
    (lambda f: f.get(f'{format_type}codec') != 'none')
    if format_type and format_modified      # bv*, ba* → 正向：有该类编解码器
    else (lambda f: f.get(f'{not_format_type}codec') == 'none')
    if format_type                         # bv, ba → 反向：没有对立类编解码器
    else (lambda f: ...)                    # b, w, b*, w* 的其余分支
)
```

所有过滤器都附加一个**基础条件**：`vcodec≠none OR acodec≠none`，排除 storyboard 等无效格式。

### 7.5 序号修饰符 `.n`：`bv.2`、`ba.3` 等

正则中的 `(?:\.(?P<n>[1-9]\d*))?` 捕获点号后的整数：

```python
format_idx = int_or_none(mobj.group('n'), default=1)
# ...
yield matches[format_idx - 1]     # 取排序后第 format_idx 个（1-based → 0-based）
```

**`bv.2` 的完整流程**：
1. 解析：`bv` → format_type=`'v'`, modified=False；`.2` → format_idx=2
2. 过滤条件：`acodec==none`（纯视频）
3. 候选倒序排列（`format_reverse=True`，best 从优到差）
4. `matches[1]`（0-based 索引 1 = 1-based 第 2 个）→ 产出**第二佳纯视频格式**

**组合示例**：
| 写法 | 含义 |
|------|------|
| `bv` | 第 1 佳纯视频 |
| `bv.2` | 第 2 佳纯视频 |
| `bv.3` | 第 3 佳纯视频 |
| `ba.2` | 第 2 佳纯音频 |
| `worst.2` | 第 2 差预合并格式 |
| `bv*.5` | 第 5 佳含视频格式 |

### 7.6 排序与选取

```python
format_reverse = True   # best → 倒序（最优在前）
format_reverse = False  # worst → 正序（最差在前）
format_idx = 1          # 默认取第1个
# bv.2 → format_idx = 2, 取排序后第2个
```

排序依赖 `ctx['formats']` 的原始顺序（由 extractor 和 `--format-sort` / `-S` 参数预先排好），
`best` 取倒序后第一个（即质量最高），`worst` 取正序第一个（即质量最低）。

### 7.7 回退策略

当主过滤无匹配时，有两条回退路径：

| 条件 | 触发场景 | 回退行为 |
|------|----------|----------|
| `format_fallback=True` 且 `incomplete_formats=True` | `b`/`w` 在纯音频/纯视频站点 | 放宽为 `vcodec≠none OR acodec≠none` |
| `seperate_fallback` 存在且 `has_merged_format=False` | 扩展名选择器（如 `mp4`）无匹配 | 对视频扩展名，回退为只要求 vcodec≠none |

`format_fallback` 仅在**没有指定 type 也没有指定 `*` 修饰**时成立（即只对 `b`/`w`）。
`bv`、`bv*`、`ba` 等带 type 或带 `*` 的写法**不会触发此回退**。

### 7.8 格式检查 `_check_formats`

在最终选取前，过滤不可用格式（DRM 保护或需测试的格式会实际探测可用性）。

---

## 八、过滤器：`[...]`

### 8.1 两种过滤器语法

**数值比较**（优先匹配）：

```
key OP value
OP: <, <=, >, >=, =, !=
value: 数字 + 可选单位 (K, M, G, KiB, MB 等)
```

**字符串匹配**（数值不匹配时尝试）：

```
key [!]OP value
OP:  =   (精确匹配)
     ^=  (前缀匹配)
     $=  (后缀匹配)
     *=  (包含匹配)
     ~=  (正则匹配)
!: 否定
```

### 8.2 none_inclusive `?` 修饰符

```
height<=?480  →  若 height 字段不存在，也算匹配通过
```

在代码中体现为：当 `actual_value is None` 时，若有 `?` 修饰则返回 True（通过），否则返回 False（拒绝）。

### 8.3 过滤器应用方式

```python
def final_selector(ctx):
    ctx_copy = dict(ctx)
    for _filter in filters:
        ctx_copy['formats'] = list(filter(_filter, ctx_copy['formats']))
    return selector_function(ctx_copy)
```

多个 `[...]` 过滤器以 **AND** 逻辑串联——依次过滤 `ctx['formats']`，
只有通过所有过滤器的格式才进入选择器的候选集。

---

## 九、完整选择流程（从调用到结果）

1. **格式预排序**：extractor 提取格式后，由 `--format-sort` / `-S` 参数进行排序（最差到最优）
2. **构建选择器**：`build_format_selector(format_spec)` 解析并编译为闭包函数
3. **执行选择**：`_select_formats(formats, selector)` 构造 ctx 并调用选择器函数
4. **遍历选择器树**：
   - 顶层是 `,` 分隔的列表 → 逐一执行，平铺产出
   - 遇到 `/` → 短路回退，取第一个非空结果
   - 遇到 `+` → 笛卡尔积合并
   - 遇到 `[...]` → 先过滤再选取
   - 遇到原子 → 过滤候选 → 排序 → 按 idx 取值
5. **结果处理**：返回格式列表，若为空则报错（除非 `--ignore-no-formats-error`）

### 默认格式规格

默认格式规格由 `_default_format_spec(info_dict)` 方法决定，涉及三层条件判断：

**第一层：`prefer_best` 判定（优先使用预合并格式）**

满足以下任一条件时，`prefer_best = True`：
1. **输出到 stdout**：`self.params['outtmpl']['default'] == '-'` — 流式输出时不便合并多格式
2. **直播且非从头开始**：`info_dict.get('is_live') and not self.params.get('live_from_start')` — 直播流实时推送，合并操作无法进行
3. **ffmpeg 不可用且当前未设置 prefer_best**：后续逻辑补充的条件

**第二层：ffmpeg 可用性检查与告警**

若 `prefer_best` 为 False 且 ffmpeg 不可用（`can_merge() == False`）：
1. 强制设置 `prefer_best = True` — 因为没有 ffmpeg 就无法做格式合并
2. 调用 `_get_formats(info_dict)` 预取格式列表
3. 分别用 `'b/bv+ba'` 和 `'bv*+ba/b'` 两套规格执行选择
4. 若两套规格的选择结果**不一致**（即有 ffmpeg 时能选出更好的组合），则输出告警：
   > "ffmpeg not found. The downloaded format may not be the best available. Installing ffmpeg is strongly recommended"

**第三层：最终规格选定**

| 条件 | 默认规格 | 说明 |
|------|----------|------|
| `prefer_best == True` | `best/bestvideo+bestaudio` | stdout / 直播 / 无 ffmpeg |
| `compat == True` | `bestvideo+bestaudio/best` | 多音频流或 format-spec 兼容模式 |
| 默认 | `bestvideo*+bestaudio/best` | 常规情况 |

`compat` 触发条件：`--audio-multistreams` 开启，或 `compat_opts` 包含 `'format-spec'`。

---

## 十、表达式求值示例

### `bv*[ext=mp4]+ba[ext=m4a]/b[ext=mp4]/bv*+ba/b`

```
解析树（嵌套二选一结构）:
PICKFIRST
├── MERGE(SINGLE("bv*")[ext=mp4], SINGLE("ba")[ext=m4a])
└── PICKFIRST
    ├── SINGLE("b")[ext=mp4]
    └── PICKFIRST
        ├── MERGE(SINGLE("bv*"), SINGLE("ba"))
        └── SINGLE("b")
```

执行流程（逐层短路回退）：
1. 最外层左分支：尝试 mp4 最佳含视频 + m4a 最佳纯音频 → 有结果则返回
2. 失败 → 进入第一层右分支（本身是 PICKFIRST）
3. 第二层左分支：尝试 mp4 最佳预合并 → 有结果则返回
4. 失败 → 进入第二层右分支（本身是 PICKFIRST）
5. 第三层左分支：尝试任意最佳含视频 + 任意最佳纯音频 → 有结果则返回
6. 失败 → 进入第三层右分支：尝试任意最佳预合并 → 有结果则返回
7. 全部失败 → 返回空列表

### `bv,bv.2,ba`

```
解析树: [SINGLE("bv"), SINGLE("bv.2"), SINGLE("ba")]
```

执行流程：
1. `bv` → 过滤条件 `acodec==none`（纯视频），倒序取第 1 个 → 产出最佳纯视频
2. `bv.2` → 过滤条件 `acodec==none`（纯视频），倒序取第 2 个 → 产出第二佳纯视频
3. `ba` → 过滤条件 `vcodec==none`（纯音频），倒序取第 1 个 → 产出最佳纯音频

共产出 3 个格式，用于独立下载。

### `bv*+ba+ba.2`（配合 `--audio-multistreams`）

```
解析树: MERGE(
    SINGLE("bv*"),
    MERGE(SINGLE("ba"), SINGLE("ba.2"))
)
```

执行流程：
1. 先递归计算右侧 `ba+ba.2` → 合并结果 `ba_merged`，`requested_formats = [ba, ba.2]`
2. 再做外层合并 `_merge(bv*, ba_merged)` → 左侧 `bv*` 在前，右侧合并结果在后
3. 最终 `requested_formats = [bv*, ba, ba.2]`（从左到右展开）
4. 若开启多音频流，合并结果保留两个音频轨道，顺序为 `ba` 在前、`ba.2` 在后

### `all[height<=480]`

```
解析树: SINGLE("all") with filter height<=480
```

执行流程：
1. 过滤器先在上下文中过滤，只保留 height≤480 的格式
2. `all` 将过滤后剩余的格式全部倒序（从优到差）产出
3. 若需探测可用性，逐个通过 `_check_formats`

### `mergeall`（默认不启用多流）

```
解析树: SINGLE("mergeall")
```

执行流程：
1. 过滤掉 vcodec 和 acodec 均为 none 的无效格式
2. 以最优格式（formats[-1]）为基底
3. 从次优到最差依次与当前结果 `_merge`，由于默认不允许多流，`_merge` 会按遍历顺序**保留先遇到的视频流和先遇到的音频流**，后续同类型流被丢弃
4. 最终产出一个合并格式，其实际保留的是**遍历顺序中首次出现的视频流和首次出现的音频流**，由于 mergeall 从最优开始合并，实际保留的是质量较高的视频流和质量较高的音频流

### `bestvideo[height<=?480]+bestaudio/worst`

```
解析树:
PICKFIRST
├── MERGE
│   ├── SINGLE("bestvideo") with filter height<=?480
│   └── SINGLE("bestaudio")
└── SINGLE("worst")
```

执行流程：
1. 在高度 ≤480（或高度未知）的纯视频中选最佳，与最佳纯音频合并
2. 失败 → 取最差预合并格式
