# yt-dlp 输出模板到文件命名：代码路径追踪

> **文档定位**：按代码执行路径，逐环节追踪「输出模板 → 变量展开 → 文件名清理 → 冲突处理 → 最终文件」的完整流程。每个环节附代码位置（仓库相对路径 + 行号）、分支条件、对最终结果的影响。

> **代码行号**：基于仓库根目录，格式为 `文件路径#行号`，如 `yt_dlp/utils/_utils.py#L631`。

---

## 第一部分：整体处理流程

### 1.1 七步代码路径

```
输出模板 (outtmpl)
    ↓
[1] _parse_outtmpl()         → 模板规范化、填充默认值
    ↓                        yt_dlp/YoutubeDL.py#L1201-L1209
[2] _outtmpl_expandpath()    → 环境变量展开（含 %% 保护机制）
    ↓                        yt_dlp/YoutubeDL.py#L1221-L1233
[3] prepare_outtmpl()        → 变量解析、对象遍历、格式转换
    ↓                        yt_dlp/YoutubeDL.py#L1263-L1514
[4] evaluate_outtmpl()       → Python % 格式化最终求值
    ↓                        yt_dlp/YoutubeDL.py#L1516-L1518
[5] _prepare_filename()      → 扩展名替换、长度裁剪
    ↓                        yt_dlp/YoutubeDL.py#L1521-L1546
[6] get_output_path()        → 路径拼接、sanitize_path
    ↓                        yt_dlp/YoutubeDL.py#L1211-L1218
[7] 冲突处理四层机制         → 见第三部分详细说明
    ↓
最终文件
```

### 1.2 关键代码位置速查

| 环节 | 文件路径 | 行号 |
|------|----------|------|
| 模板解析 | `yt_dlp/YoutubeDL.py` | L1201-L1209 |
| 路径展开 | `yt_dlp/YoutubeDL.py` | L1221-L1233 |
| 模板准备核心 | `yt_dlp/YoutubeDL.py` | L1263-L1514 |
| 文件名清理 | `yt_dlp/utils/_utils.py` | L631-L683 |
| 路径清理 | `yt_dlp/utils/_utils.py` | L706-L733 |
| 冲突检测 | `yt_dlp/YoutubeDL.py` | L3320-L3328 |
| 下载归档 | `yt_dlp/YoutubeDL.py` | L3868-L3887 |
| 文件移动 | `yt_dlp/postprocessor/movefilesafterdownload.py` | L21-L52 |
| 默认模板定义 | `yt_dlp/utils/_utils.py` | L2861-L2877 |
| ACCENT_CHARS 映射表 | `yt_dlp/utils/_utils.py` | L99-L101 |

---

## 第二部分：文件名清理 — 四条独立分支

`sanitize_filename` 函数有 **两个独立参数**，组合产生 **四种场景**。每条路径的代码分支完全独立。

### 2.1 函数签名与核心分支点

位置：`yt_dlp/utils/_utils.py#L631-L683`

```python
def sanitize_filename(s, restricted=False, is_id=NO_DEFAULT):
```

**两个参数的独立影响：**

| 参数 | 控制维度 | 影响范围 |
|------|----------|----------|
| `restricted` | 字符集严格度 | 全角/半角、Unicode/ASCII、允许的特殊字符 |
| `is_id` | 是否为标识符 | 首尾清理、重复字符合并、下划线处理 |

**理解 elif 链的关键**：`replace_insane` 函数（L640-L658）是一个 if/elif 链，每个字符只命中**第一个**匹配分支。因此 L645 命中后，L648-L655 不会再执行。不同参数组合会改变哪些分支可达。

### 2.2 分支一：默认模式（restricted=False, is_id=NO_DEFAULT）

**触发条件**：既非受限模式，也不明确指定 `is_id`。这是 yt-dlp 的新规则，也是**最常见的场景**。

**代码路径：**

```
s = '输入字符串'
    ↓
[L661-L662] 跳过 NFKC 规范化（因为 restricted=False）
    ↓
[L663] 时间戳预处理（先于逐字符替换）：
    正则 r'[0-9]+(?::[0-9]+)+' 匹配时间格式如 0:12:34
    命中的 ':' → '_'，所以 0:12:34 → 0_12_34
    此步在 replace_insane 之前，已处理的 ':' 不会再进入逐字符替换
    ↓
[L664] 逐字符调用 replace_insane(char)，if/elif 链的可达分支：
    ├─ [L641] restricted=False → 跳过 ACCENT_CHARS
    ├─ [L643] 仅 '\n' → '\0 '
    ├─ [L645] ★核心分支★ is_id=NO_DEFAULT and not restricted
    │     字符集 '"*:<>?|/\' 全部在此命中：
    │     '/'  → '⧸' (U+29F8)，通过 dict 映射
    │     '\\' → '⧹' (U+29F9)，通过 dict 映射
    │     '"'  → '＂' (U+FF02)，通过 chr(ord+0xfee0)
    │     '*'  → '＊' (U+FF0A)，通过 chr(ord+0xfee0)
    │     ':'  → '：' (U+FF1A)，通过 chr(ord+0xfee0)
    │     '<'  → '＜' (U+FF1C)，通过 chr(ord+0xfee0)
    │     '>'  → '＞' (U+FF1E)，通过 chr(ord+0xfee0)
    │     '?'  → '？' (U+FF1F)，通过 chr(ord+0xfee0)
    │     '|'  → '｜' (U+FF5C)，通过 chr(ord+0xfee0)
    │     ★ 注意：此分支命中后，L648-L655 对以上字符不可达 ★
    ├─ [L648] 仅处理：控制字符(<32, 非\n)、DEL(127)
    │     → 删除（返回空串）
    ├─ [L650] 默认模式下 '"' 已被 L645 截获 → 不可达
    ├─ [L652] 默认模式下 ':' 已被 L645 截获 → 不可达
    ├─ [L654] 默认模式下 '\\/|*<>' 已被 L645 截获 → 不可达
    └─ [L656] restricted=False → 跳过
    ↓
[L665-L668] is_id=NO_DEFAULT 时执行后处理：
    ├─ [L666] 重复替换字符去重（正则匹配 \0. 模式）
    └─ [L667-L668] 首尾清理：去掉首尾的替换字符、空格、_、-
    ↓
[L669] 移除 \0 标记，空则返回 '_'
    ↓
[L671] not is_id → False（因为 is_id=NO_DEFAULT 不等于 False）
    → 跳过 L672-L682 的额外清理
    ↓
返回结果
```

**对最终文件名的影响**：
- 保留 Unicode 字符（中文、重音符号等）
- `"*:<>?|/\\` 全部转全角，视觉上保留原意
- 时间戳格式中的 `:` 先被替换为 `_`（如 `0:12:34` → `0_12_34`）
- 非时间戳的 `:` 转全角 `：`
- 路径分隔符 `/` `\` 替换为 `⧸` `⧹`，避免目录穿越

### 2.3 分支二：受限模式（restricted=True）

**触发条件**：`--restrict-filenames` 参数，或 `params['restrictfilenames']=True`。

**代码路径：**

```
s = '输入字符串'
    ↓
[L661-L662] NFKC 规范化（restricted=True 且 is_id=NO_DEFAULT 或 not is_id）
    ↓
[L663] 时间戳预处理：同上
    ↓
[L664] 逐字符调用 replace_insane(char)，可达分支：
    ├─ [L641] restricted=True → ACCENT_CHARS 重音映射表
    │     映射表定义：yt_dlp/utils/_utils.py#L99-L101
    │     例：Ä→A, é→e（重音字符转 ASCII）
    ├─ [L643] restricted=True → 跳过换行处理
    ├─ [L645] restricted=True → 跳过全角转换
    ├─ [L648] '?'、控制字符(<32)、DEL(127) → 删除
    ├─ [L650] '"' → 删除（restricted=True 分支）
    ├─ [L652] ':' → '\0_\0-'（后续变为 '_-_'）
    ├─ [L654] '\\/|*<>' → '\0_'（后续变为 '_'）
    └─ [L656] 更多字符过滤：
          !&'()[]{}$;`^,#、空格 → 删除或 '_'
          Unicode > 127 → 组合字符(C/M类)删除，其他替换为 '_'
    ↓
[L665-L668] is_id=NO_DEFAULT 时：去重 + 首尾清理
    ↓
[L669] 移除 \0 标记
    ↓
[L671] not is_id → True（因为 is_id=NO_DEFAULT 在布尔上下文为真，
    但代码检查 `not is_id`，NO_DEFAULT 不是 False，所以实际为 False）
    → 实际跳过 L672-L682

    ★ 修正：is_id=NO_DEFAULT 时，`not is_id` 为 False（NO_DEFAULT 是哨兵值，非 False）
    所以 L672-L682 在 is_id=NO_DEFAULT 时不执行
    ↓
返回结果
```

**对最终文件名的影响**：
- 纯 ASCII，无任何 Unicode 字符
- 重音字符通过 ACCENT_CHARS 映射为 ASCII
- 文件名最短，空格和特殊字符都被替换或删除
- 适用于需要兼容旧文件系统的场景

**受限模式 vs 默认模式对照**：

| 字符 | 默认模式 | 受限模式 |
|------|----------|----------|
| `:` | `：`（全角） | `_-_` |
| `"` | `＂`（全角） | 删除 |
| `?` | `？`（全角） | 删除 |
| `*` | `＊`（全角） | `_` |
| `<` | `＜`（全角） | `_` |
| `>` | `＞`（全角） | `_` |
| `\|` | `｜`（全角） | `_` |
| `/` | `⧸` (U+29F8) | `_` |
| `\\` | `⧹` (U+29F9) | `_` |
| `ä` | `ä`（保留） | `a`（映射） |
| `\n` | ` `（空格） | `_` |
| 空格 | 保留 | `_` |

### 2.4 分支三：ID 模式（is_id=True）

**触发条件**：
- 默认（无 `filename-sanitization` compat_opts）：`is_id=NO_DEFAULT`，字段名不影响
- 有 `filename-sanitization` compat_opts 时：字段名匹配正则 `(^|[_.])id(\.|$)`

**判断位置**：`yt_dlp/YoutubeDL.py#L1385-L1388`

```python
is_id = bool(re.search(r'(^|[_.])id(\.|$)', key))
```

**代码路径（与默认/受限模式的差异）**：

```
... 前面的字符替换同上（受 restricted 影响）...
    ↓
[L665] 条件 is_id is NO_DEFAULT → False（因为 is_id=True）
    ↓
跳过 [L666-L668] 的重复替换字符去重和首尾清理
    ↓
[L669] 移除 \0 标记，空则返回 '_'
    ↓
[L671] 条件 not is_id → False（因为 is_id=True）
    ↓
跳过 [L672-L682] 的所有额外清理
    ↓
直接返回结果
```

**对最终文件名的影响**：
- 不合并重复下划线（如 `ab__cd` 保持不变）
- 不清理首尾的 `-`、`.`、`_`
- 不执行首尾替换字符清理（L666-L668 跳过）
- 最大限度保持 ID 的原始格式
- 举例：`_n_cd26wFpw` → `_n_cd26wFpw`（不变）

**注意**：当 `is_id=True` 且 `restricted=True` 时，L661 的 NFKC 规范化**不会执行**（条件 `not is_id` 为 False）。即 ID 字段在受限模式下不做 NFKC 规范化，但 `replace_insane` 中的 ACCENT_CHARS 映射仍生效。

### 2.5 分支四：非 ID 模式（is_id=False）

**触发条件**：`compat_opts` 包含 `filename-sanitization` 且字段名不匹配 ID 正则。

**代码路径（与默认模式的差异）**：

```
... 字符替换同上（受 restricted 影响，但 L645 不可达）...
    ★ 关键差异：is_id=False 时，L645 条件不满足
    ★ 所以 '"*:<>?|/\' 不走全角转换，走后续的 L648-L655
    ↓
[L665] 条件 is_id is NO_DEFAULT → False（因为 is_id=False）
    ↓
跳过 [L666-L668] 的去重和首尾清理
    ↓
[L669] 移除 \0 标记
    ↓
[L671] 条件 not is_id → True
    ↓
执行 [L672-L682] 的所有清理：
    ├─ [L672-L673] '__' → '_'（循环直到无连续下划线）
    ├─ [L674] 首尾 '_' 去除
    ├─ [L676-L677] restricted 时开头 '-_' 移除
    ├─ [L678-L679] 开头 '-' → '_'
    ├─ [L680] 开头 '.' 移除
    └─ [L681-L682] 空则 '_'
    ↓
返回结果
```

**is_id=False 时 L645 不可达的影响**：

| 字符 | is_id=NO_DEFAULT (L645 可达) | is_id=False (L645 不可达) |
|------|------|------|
| `"` | `＂`(全角) → L645 | `'`(单引号) → L650 |
| `*` | `＊`(全角) → L645 | `_` → L654 |
| `:` | `：`(全角) → L645 | ` -` → L652 |
| `<` | `＜`(全角) → L645 | `_` → L654 |
| `>` | `＞`(全角) → L645 | `_` → L654 |
| `?` | `？`(全角) → L645 | 删除 → L648 |
| `\|` | `｜`(全角) → L645 | `_` → L654 |
| `/` | `⧸` → L645 | `_` → L654 |
| `\\` | `⧹` → L645 | `_` → L654 |

### 2.6 prepare_outtmpl 中 sanitize 的决策树

位置：`yt_dlp/YoutubeDL.py#L1390-L1400`

```
调用 prepare_outtmpl(..., sanitize=?)
    ↓
[L1390] sanitize 是 callable? → 弃用警告，继续使用
    ↓
[L1392] sanitize 是 False? → 不做任何清理
    ↓
[L1394] 三个条件同时满足？
    ├─ sys.platform != 'win32'
    ├─ restrictfilenames 未设置
    └─ windowsfilenames=False
    ├─ 是 → 极简清理：仅替换 '/'→'⧸' 和 '\0'→''
    └─ 否 → 完整清理：调用 filename_sanitizer(key, value, restricted=?)
        ↓
        [L1385-L1388] 根据 key 正则判断 is_id：
            无 filename-sanitization compat → is_id=NO_DEFAULT
            有 filename-sanitization compat → is_id=bool(re.match id pattern)
        ↓
        调用 sanitize_filename(value, restricted, is_id)
```

### 2.7 文件名清理的调用时机

位置：`yt_dlp/YoutubeDL.py#L1500-L1508`

仅对以下格式类型执行清理（`sanitize=True` 且 `fmt[-1] in 'csra'`）：
- `c`（首字符）
- `s`（字符串，默认）
- `r`（repr 表示，先转 repr 再清理）
- `a`（ASCII 表示，先转 ascii 再清理）

对 `d/i/f/e/g/l/j/h/q/B/U/D/S` 等格式不执行文件名清理。
`S` 格式内部调用 `filename_sanitizer`，不走外层 `sanitize` 逻辑。

### 2.8 路径清理 sanitize_path

位置：`yt_dlp/utils/_utils.py#L706-L733`

**Windows 特有处理（或 force=True）：**
- 解析 UNC 路径 `\\SERVER\SHARE`
- 解析绝对路径 `C:\path`
- 对每个路径段调用 `_sanitize_path_parts`（L686-L703）：
  - 空串或 `.` → 跳过
  - `..` → 上一级（弹出路径栈最后一个非 `..` 项）
  - 无效字符 `/<>:"\|\\?*` 或结尾空格/点 → 替换为 `#`

**非 Windows：**
- 默认返回原路径（force=False）
- `force=True` 时才执行清理

**调用时机**：`get_output_path`（`yt_dlp/YoutubeDL.py#L1218`）在拼接路径后调用 `sanitize_path(path, force=windowsfilenames)`。

---

## 第三部分：冲突处理与归档写入 — 完整执行时序

### 3.1 核心结论先行

**同名文件且不覆盖时，归档仍然会被写入。**

这是因为：`__write_download_archive` 在 `post_process` 和 `post_hooks` 成功完成后**无条件**被设置为 `True`（`yt_dlp/YoutubeDL.py#L3667`），而即使目标文件已存在且不覆盖，`post_process` 内部的 `MoveFilesAfterDownloadPP` 只会打印 warning 并 `continue`，**不会抛出异常终止流程**。

### 3.2 四层冲突机制总览

四层机制在下载流程的 **不同阶段** 依次触发，但只有前两层能完全跳过后续流程。第三、四层只影响文件操作，不影响归档写入决策。

```
下载启动
    ↓
[第一层] SameFileError 检测
    时机：download() 开始时
    作用：防止多个 URL 写入同一固定文件名
    影响：异常终止，不产生任何文件，不写入归档
    ↓
[第二层] download_archive 检测（两次）
    时机 1：extract_info() URL 解析时
    时机 2：_match_entry() 过滤时
    作用：基于视频 ID 的去重
    影响：跳过下载，__write_download_archive='ignore'，不写入归档
    ↓
[第三层] existing_file + overwrites 检测
    时机：process_info() 下载前
    作用：文件系统级别的存在检测
    影响：文件已存在且不覆盖 → 跳过下载，但继续执行 post_process
    ↓
[第四层] MoveFilesAfterDownloadPP 检测
    时机：post_process() 后处理时
    作用：临时文件移动到最终位置时的检测
    影响：目标已存在且不覆盖 → 跳过移动，但不抛异常
    ↓
__write_download_archive = True （L3667，只要没异常 return 就会执行）
    ↓
最终归档写入决策（L3140-L3143）
```

### 3.3 第一层：SameFileError — 固定文件名冲突

**触发位置**：`yt_dlp/YoutubeDL.py#L3697-L3702`

**触发条件（同时满足）：**
1. `len(url_list) > 1` — 有多个 URL
2. `outtmpl != '-'` — 不是输出到 stdout
3. `'%' not in outtmpl` — 模板中没有变量（固定文件名）
4. `max_downloads != 1` — 不止下载一个

**对最终结果的影响**：
- 直接抛出 `SameFileError` 异常
- 下载流程终止，**不产生任何文件，也不写入归档**
- 此检测在文件名清理和路径拼接之前，作用于原始模板字符串

### 3.4 第二层：download_archive — 视频 ID 去重

#### 3.4.1 第一次检查：URL 解析时

位置：`yt_dlp/YoutubeDL.py#L1708-L1713`

```python
temp_id = ie.get_temp_id(url)
if temp_id is not None and self.in_download_archive({'id': temp_id, 'ie_key': key}):
    if self.params.get('break_on_existing', False):
        raise ExistingVideoReached
    break
```

此检查发生在 URL 解析阶段，**尚未生成文件名**。命中时通过 `break` 跳出循环，整个视频被跳过。

#### 3.4.2 第二次检查：_match_entry 过滤时

位置：`yt_dlp/YoutubeDL.py#L1645-L1650`

在 `process_info` 最开始调用 `_match_entry`。`_match_entry` 内部检查 `in_download_archive`。

位置：`yt_dlp/YoutubeDL.py#L3340-L3342`

```python
if self._match_entry(info_dict) is not None:
    info_dict['__write_download_archive'] = 'ignore'
    return
```

**关键**：此处显式设置 `__write_download_archive = 'ignore'`，然后 `return` 直接退出 `process_info`。

#### 3.4.3 in_download_archive 实现

位置：`yt_dlp/YoutubeDL.py#L3868-L3874`

```python
def in_download_archive(self, info_dict):
    if not self.archive:
        return False
    vid_ids = [self._make_archive_id(info_dict)]
    vid_ids.extend(info_dict.get('_old_archive_ids') or [])
    return any(id_ in self.archive for id_ in vid_ids)
```

**归档 ID 格式**：`make_archive_id(extractor_key, video_id)` → 如 `youtube dQw4w9WgXcQ`

**对最终结果的影响**：
- 第一次检查命中：整个视频被跳过，`process_info` 甚至不会被调用
- 第二次检查命中：`__write_download_archive = 'ignore'`，`process_info` 直接 return
- **两种情况都不会写入归档**

#### 3.4.4 `__write_download_archive` 标记的五种取值时机

此标记是归档写入决策的唯一依据。完整取值时机表：

| 场景 | 值 | 设置位置 | 说明 |
|------|-----|----------|------|
| 被 `_match_entry` 过滤 | `'ignore'` | `yt_dlp/YoutubeDL.py#L3341` | 视频 ID 已在归档中或被过滤条件排除 |
| `simulate` 模式 | `force_write_download_archive` | `yt_dlp/YoutubeDL.py#L3371` | 模拟下载，由参数决定是否归档 |
| `skip_download` 模式 | `force_write_download_archive` | `yt_dlp/YoutubeDL.py#L3457` | 跳过下载，由参数决定是否归档 |
| `process_info` 正常走完 | `True` | `yt_dlp/YoutubeDL.py#L3667` | post_process 和 post_hooks 均未异常 return |
| `force_write_download_archive=True` | `True`（强制覆盖） | `yt_dlp/YoutubeDL.py#L3670-L3671` | 无论前面设置为何值，强制改为 `True` |
| （默认值，未设置） | `False` | `yt_dlp/YoutubeDL.py#L3140` | 通过 `f.get('__write_download_archive', False)` 获取 |

**没有任何地方显式设置为 `False`**——`False` 仅作为 `dict.get()` 的默认值出现。

### 3.5 第三层：existing_file + overwrites — 文件系统检测

#### 3.5.1 overwrites 参数三态

位置：`yt_dlp/YoutubeDL.py#L291-L293`

| 值 | 视频文件（default_overwrite=False） | 辅助文件（default_overwrite=True） |
|----|--------------------------------------|--------------------------------------|
| `True` | 覆盖 | 覆盖 |
| `None`（默认） | 不覆盖 | 覆盖 |
| `False` | 不覆盖 | 不覆盖 |

**原理**：`existing_file` 的 `default_overwrite` 参数决定 `overwrites=None` 时的行为。

#### 3.5.2 existing_file 核心逻辑

位置：`yt_dlp/YoutubeDL.py#L3320-L3328`

```python
def existing_file(self, filepaths, *, default_overwrite=True):
    existing_files = list(filter(os.path.exists, orderedSet(filepaths)))
    if existing_files and not self.params.get('overwrites', default_overwrite):
        return existing_files[0]  # 不覆盖：返回已存在文件路径
    
    for file in existing_files:
        self.report_file_delete(file)
        os.remove(file)  # 覆盖：删除现有文件
    return None  # 可以下载
```

**关键**：返回非 None = 文件已存在且不覆盖；返回 None = 可以下载。

**不覆盖时对后续流程的影响**：返回已存在文件路径，但**不会 return 或 throw**，流程继续向下执行。

#### 3.5.3 视频文件检测：existing_video_file

位置：`yt_dlp/YoutubeDL.py#L3463-L3470`

同时检查原始扩展名和转换后扩展名（如 `.webm` 和 `.mkv`），传入 `default_overwrite=False`。

**单文件下载时的分支**（`yt_dlp/YoutubeDL.py#L3575-L3582`）：

```python
dl_filename = existing_video_file(full_filename, temp_filename)
if dl_filename is None or dl_filename == temp_filename:
    # 无现有文件，或 --no-part 场景（部分下载）
    success, real_download = self.dl(temp_filename, info_dict)
    info_dict['__real_download'] = real_download
else:
    # 文件已存在且不覆盖 → 跳过下载
    self.report_file_already_downloaded(dl_filename)
    # ★ 注意：此处没有设置任何 __write_download_archive 标记 ★
    # ★ 也没有 return，流程继续向下 ★
```

#### 3.5.4 辅助文件检测

| 文件类型 | 检测位置 | default_overwrite |
|----------|----------|-------------------|
| 信息 JSON | `yt_dlp/YoutubeDL.py#L4396` | `True`（覆盖） |
| 描述文件 | `yt_dlp/YoutubeDL.py#L4425` | `True`（覆盖） |
| 网络快捷方式 | `yt_dlp/YoutubeDL.py#L3418` | `True`（覆盖） |

**第三层与第二层的关系**：第二层基于视频 ID 去重（命中则 return 退出）；第三层基于文件系统检测（不覆盖时只是跳过下载，流程继续）。第二层会阻止归档写入，第三层不会。

### 3.6 第四层：MoveFilesAfterDownloadPP — 文件移动检测

位置：`yt_dlp/postprocessor/movefilesafterdownload.py#L21-L52`

```python
for oldfile, newfile in info['__files_to_move'].items():
    if os.path.abspath(oldfile) == os.path.abspath(newfile):
        continue  # 同一文件，跳过
    if not os.path.exists(oldfile):
        continue  # 源不存在，跳过（同名文件不覆盖场景会走到这里）
    if os.path.exists(newfile):
        if self.get_param('overwrites', True):
            self.report_warning(f'Replacing existing file "{newfile}"')
            os.remove(newfile)
        else:
            self.report_warning(
                f'Cannot move file "{oldfile}" out of temporary directory '
                f'since "{newfile}" already exists. ')
            continue  # ★ 不覆盖时只是 continue，不抛异常 ★
    shutil.move(oldfile, newfile)
```

**关键行为**：
- `continue` 而不是抛出异常 → `post_process` 正常返回
- `get_param('overwrites', True)` 的默认值是 `True`（与 `existing_file` 不同）
- 第三层已判定不覆盖且跳过下载时，`oldfile`（临时文件）不存在 → 直接 `continue`

**对最终结果的影响**：
- 不覆盖时：临时文件保留在临时目录（如果有），**最终位置不产生新文件**
- 覆盖时：删除目标文件，移动成功
- 源和目标相同时：无需移动
- **无论何种情况，MoveFilesAfterDownloadPP 都不会导致 post_process 异常退出**

### 3.7 同名文件且不覆盖场景的完整时序追踪

这是最容易产生误解的场景。以下是逐行代码追踪：

```
YoutubeDL.download([url])
  ↓
[第一层] SameFileError：单 URL，跳过
  ↓
extract_info(url)
  ↓
[第二层(1)] in_download_archive(temp_id)：不在归档，继续
  ↓
process_ie_result()
  ↓
process_info(info_dict)
  │
  ├─ [第二层(2)] _match_entry()：不在归档，继续
  │     __write_download_archive 未设置
  │
  ├─ prepare_filename() → full_filename（生成目标文件名）
  ├─ prepare_filename('temp') → temp_filename（生成临时文件名）
  │
  ├─ 写辅助文件（JSON/描述/缩略图/字幕/快捷方式等）
  │     各自检测 overwrites，default_overwrite=True → 辅助文件可能被覆盖
  │
  ├─ pre_process('before_dl')
  │
  ├─ skip_download=False → 进入下载分支
  │     info_dict.setdefault('__postprocessors', [])
  │
  ├─ existing_video_file(full_filename, temp_filename)  [L3575]
  │     │  ← 传入 default_overwrite=False
  │     │
  │     ├─ existing_file() 检查文件系统
  │     │     └─ full_filename 存在且 overwrites 不为 True
  │     │         → 返回 full_filename（非 None）
  │     │
  │     └─ dl_filename = full_filename（非 None，非 temp_filename）
  │
  ├─ if dl_filename is None or dl_filename == temp_filename:  [L3576]
  │     └─ False（dl_filename 是 full_filename，不是 temp）
  │
  ├─ else:  [L3581-L3582]
  │     ├─ report_file_already_downloaded(dl_filename)
  │     └─ ★ 此处没有设置 __write_download_archive ★
  │     └─ ★ 此处没有 return，流程继续 ★
  │
  ├─ dl_filename = dl_filename or temp_filename  [L3584]
  │     → dl_filename = full_filename
  ├─ info_dict['__finaldir'] = ...
  │
  ├─ try 块无异常，success 仍为 True
  │
  ├─ _raise_pending_errors(info_dict) → 无错误
  │
  ├─ if success and full_filename != '-':  [L3597]
  │     │  ← True，进入此块
  │     │
  │     ├─ fixup()
  │     │     fixup_policy='detect_or_warn' 且 __real_download=False
  │     │     → do_fixup=False，不做任何 ffmpeg 修复
  │     │
  │     ├─ post_process(dl_filename, info_dict, files_to_move)  [L3657]
  │     │     │
  │     │     ├─ info['filepath'] = dl_filename  ← full_filename
  │     │     ├─ run_all_pps('post_process', ...)
  │     │     │     → 无额外后处理器（因为没有下载）
  │     │     ├─ run_pp(MoveFilesAfterDownloadPP)
  │     │     │     └─ 遍历 __files_to_move（辅助文件）
  │     │     │         目标已存在且不覆盖 → warning + continue
  │     │     │         ★ 不抛异常 ★
  │     │     └─ run_all_pps('after_move', ...)
  │     │
  │     ├─ try: post_hooks  [L3661-L3665]
  │     │     → 无异常则继续
  │     │
  │     └─ ★ info_dict['__write_download_archive'] = True ★  [L3667]
  │         无条件执行！
  │
  ├─ if force_write_download_archive:  [L3670-L3671]
  │     → 如果为 True，再次强制设为 True
  │
  └─ check_max_downloads()
  │
  └─ process_info 返回，info_dict 中 __write_download_archive = True
  ↓
process_ie_result() 继续执行
  ↓
[归档写入决策]  yt_dlp/YoutubeDL.py#L3140-L3143
  │
  ├─ write_archive = {f.get('__write_download_archive', False)
  │                  for f in downloaded_formats}
  │     → {True}
  │
  └─ if True in write_archive and False not in write_archive:
       → True，条件满足
       → self.record_download_archive(info_dict)  ★ 归档被写入 ★
```

**结论**：同名文件且不覆盖时，只要 post_process 和 post_hooks 没有异常（MoveFilesAfterDownloadPP 只 continue 不抛异常），L3667 就会把 `__write_download_archive` 设为 `True`，归档最终被写入。

### 3.8 归档写入的完整决策逻辑

位置：`yt_dlp/YoutubeDL.py#L3140-L3143`

```python
write_archive = {f.get('__write_download_archive', False) for f in downloaded_formats}
assert write_archive.issubset({True, False, 'ignore'})
if True in write_archive and False not in write_archive:
    self.record_download_archive(info_dict)
```

**三个取值的含义**：

| 值 | 对 write_archive 集合的影响 | 对最终写入决策的影响 |
|----|---------------------------|---------------------|
| `True` | 集合中含 `True` | 满足条件（只要没有 `False`） |
| `False`（默认） | 集合中含 `False` | 条件不满足（`False not in write_archive` 为 False） |
| `'ignore'` | 集合中含 `'ignore'` | 不影响判断（既不是 `True` 也不是 `False`） |

**决策真值表**：

| downloaded_formats 中各格式的标记 | write_archive 集合 | True in? | False not in? | 最终写入？ |
|-----------------------------------|-------------------|----------|---------------|-----------|
| `[True]` | `{True}` | ✓ | ✓ | ✓ 写入 |
| `[False]` | `{False}` | ✗ | ✗ | ✗ 不写 |
| `['ignore']` | `{'ignore'}` | ✗ | ✓ | ✗ 不写 |
| `[True, 'ignore']` | `{True, 'ignore'}` | ✓ | ✓ | ✓ 写入 |
| `[True, False]` | `{True, False}` | ✓ | ✗ | ✗ 不写 |
| `[False, 'ignore']` | `{False, 'ignore'}` | ✗ | ✗ | ✗ 不写 |
| `[True, True]` | `{True}` | ✓ | ✓ | ✓ 写入 |

### 3.9 各冲突层与归档写入的关系总结

| 层次 | 是否阻止归档写入 | 机制 |
|------|-----------------|------|
| 第一层 SameFileError | **是** | 异常终止，整个流程中断 |
| 第二层 download_archive（两次） | **是** | 设置 `__write_download_archive='ignore'` 后 return 或 break |
| 第三层 existing_file（不覆盖） | **否** | 只跳过下载，不设置 False，流程继续 |
| 第四层 MoveFilesAfterDownloadPP（不覆盖） | **否** | 只跳过移动，不抛异常，L3667 仍设为 True |
| `force_write_download_archive=True` | **强制写入** | L3670-L3671 无条件覆盖为 True |

### 3.10 典型场景的最终状态

| 场景 | archive | overwrites | 跳过下载？ | post_process 正常？ | `__write_download_archive` | 归档写入？ | 最终文件状态 |
|------|---------|------------|-----------|-------------------|---------------------------|-----------|-------------|
| 新视频，无同名文件 | - | 任意 | 否 | 是 | `True` | ✓ | ✓ 新文件 |
| 新视频，有同名文件 | - | True | 否（删除旧的重新下载） | 是 | `True` | ✓ | ✓ 覆盖旧文件 |
| 新视频，有同名文件 | - | False | 是 | 是（移动 continue 不抛异常） | `True` | ✓ | ✗ 保留旧文件 |
| 新视频，有同名文件 | - | None | 是（视频 default=False） | 是（移动 continue 不抛异常） | `True` | ✓ | ✗ 保留旧视频，辅助文件可能覆盖 |
| 视频已在 archive | 包含 | 任意 | 是（第二层跳过） | 否（第二层 return） | `'ignore'` | ✗ | ✗ 跳过下载 |
| 固定文件名 + 多 URL | - | 任意 | - | - | - | ✗ | ✗ 异常终止 |
| 任何场景 + force_write | - | - | - | - | `True`（强制） | ✓ | 取决于上述场景 |

**注意**：第三、四层不覆盖场景下，归档仍然会被写入。这意味着后续相同视频 ID 的下载会被第二层直接跳过，即使目标文件从未真正被替换。

---

## 第四部分：变量展开核心逻辑

### 4.1 模板语法解析

位置：`yt_dlp/YoutubeDL.py#L1304-L1313`

内部格式正则：
```
(?P<negate>-)?
(?P<fields>{FIELD_RE})
(?P<maths>(?:{MATH_OPERATORS_RE}{MATH_FIELD_RE})*)
(?:>(?P<strf_format>.+?))?
(?P<remaining>
    (?P<alternate>(?<!\\),[^|&)]+)?
    (?:&(?P<replacement>.*?))?
    (?:\|(?P<default>.*?))?
)$
```

### 4.2 支持的语法元素

| 语法 | 示例 | 说明 |
|------|------|------|
| 对象遍历 | `%(formats.0.id)s` | 点号遍历嵌套对象 |
| 取反 | `-%(view_count)s` | 数值取负 |
| 数学运算 | `%(duration+10)d` | `+` `-` `*` 三种运算 |
| 日期格式化 | `%(upload_date>%Y-%m-%d)s` | `>` 后接 strftime 格式 |
| 替代字段 | `%(artist,uploader)s` | `,` 分隔，按序尝试 |
| 替换格式 | `%(id&https://youtu.be/{})s` | `&` 后接替换模板 |
| 默认值 | `%(uploader|Unknown)s` | `|` 后接默认值 |

### 4.3 格式转换类型

| 字符 | 含义 | 调用外层 sanitize |
|------|------|----------|
| `s` | 字符串（默认） | ✓ |
| `d`, `i` | 整数 | ✗ |
| `f`, `e`, `g` | 浮点数 | ✗ |
| `r` | repr 表示 | ✓（先转 repr 再 sanitize） |
| `a` | ASCII 表示 | ✓（先转 ascii 再 sanitize） |
| `c` | 首字符 | ✓ |
| `l` | 列表格式 | ✗ |
| `j` | JSON 格式 | ✗ |
| `h` | HTML 转义 | ✗ |
| `q` | Shell 引号转义 | ✗ |
| `B` | 字节格式化 | ✗ |
| `U` | Unicode 规范化 | ✗ |
| `D` | 十进制后缀 | ✗ |
| `S` | 文件名清理 | ✗（内部直接调用 filename_sanitizer） |

---

## 第五部分：关键设计要点

### 5.1 _outtmpl_expandpath 的 %% 保护机制

位置：`yt_dlp/YoutubeDL.py#L1221-L1233`

**问题**：`expand_path`（`yt_dlp/utils/_utils.py#L768`）会展开 `%VAR%`（Windows）和 `$VAR`（Unix）形式的环境变量，但模板变量 `%(title)s` 中的 `%` 和元数据中的 `$` 不应被展开。

**解决方案**：
1. 生成 32 位随机字母分隔符 `sep`
2. `%%` → `%{sep}%`，`$$` → `${sep}$`
3. 调用 `expand_path` 展开真正的环境变量
4. 移除分隔符，恢复 `%%` 和 `$$`

### 5.2 文件名中的 \0 标记机制

在 `sanitize_filename` 的 `replace_insane` 中，许多替换结果包含 `\0` 字符：
- `:` → `\0 \0-`（非 restricted，is_id 非 NO_DEFAULT）
- `:` → `\0_\0-`（restricted）
- `\ / | * < >` → `\0_`

**作用**：
1. 后续可以用 `\0.` 正则匹配这些替换（一个 `\0` 加一个替换结果字符）
2. L666 去重连续相同的替换字符
3. L667-L668 从首尾清理替换字符及其周围的空格/下划线/连字符
4. L669 一次性移除所有 `\0`，得到最终的空格/下划线

### 5.3 replace_insane 的 elif 链优先级

L640-L658 是一个 if/elif 链，每个字符只命中第一个匹配分支。不同参数组合改变分支可达性：

| 分支 | 条件 | 默认模式可达 | 受限模式可达 | is_id=False 可达 |
|------|------|:---:|:---:|:---:|
| L641 | `restricted and ACCENT_CHARS` | ✗ | ✓ | ✗ |
| L643 | `not restricted and '\n'` | ✓(仅\n) | ✗ | ✓(仅\n) |
| L645 | `is_id=NO_DEFAULT and not restricted and "特定字符"` | ✓ | ✗ | ✗ |
| L648 | `? / 控制字符 / DEL` | ✓(仅控制/DEL) | ✓ | ✓ |
| L650 | `"` | ✗(被L645截获) | ✓ | ✓ |
| L652 | `:` | ✗(被L645截获) | ✓ | ✓ |
| L654 | `\\/|*<>` | ✗(被L645截获) | ✓ | ✓ |
| L656 | `restricted and 更多字符` | ✗ | ✓ | ✗ |

### 5.4 自动生成字段

位置：`yt_dlp/YoutubeDL.py#L1268-L1278`

| 字段 | 说明 |
|------|------|
| `epoch` | Unix 时间戳，整个过程保持一致 |
| `duration_string` | 时长字符串，文件名中用 `-` 替代 `:` |
| `autonumber` | 自动编号，从 `autonumber_start` 开始 |
| `video_autonumber` | 视频自动编号 |
| `resolution` | 分辨率字符串 |

### 5.5 字段大小兼容性映射

位置：`yt_dlp/YoutubeDL.py#L1282-L1286`

以下字段的 `%(field)s` 自动转换为 `%(field)0Nd`：
- `playlist_index`：位数由 `__last_playlist_index` 决定
- `playlist_autonumber`：位数由 `n_entries` 决定
- `autonumber`：位数由 `autonumber_size` 决定（默认 5）

---

## 第六部分：输入输出示例

### 示例 1：默认模式文件名清理

**输入**：`"Hello: World/Test?"`（标题字段）
**代码路径**：`restricted=False, is_id=NO_DEFAULT`
**输出**：`Hello：World⧸Test？`

转换过程：
- L663 时间戳：`Hello: World/Test?` 中无时间戳格式，不影响
- L645 全角转换：`:` → `：`，`/` → `⧸`，`?` → `？`，`"` → `＂`，`*` → `＊`，`<` → `＜`，`>` → `＞`，`|` → `｜`
- L666-L668 去重 + 首尾清理：无影响
- L669 移除 \0

### 示例 2：受限模式文件名清理

**输入**：`"Hello: World/Test?"`
**代码路径**：`restricted=True, is_id=NO_DEFAULT`
**输出**：`Hello_-_World_Test`

转换过程：
- L641 ACCENT_CHARS：不适用
- L648 `?` → 删除
- L652 `:` → `_-_`
- L654 `/` → `_`
- L656 空格 → `_`
- L666-L668 去重 + 首尾清理
- L669 移除 \0

### 示例 3：ID 字段保持

**输入**：`"_n_cd26wFpw"`
**代码路径**：`is_id=True`
**输出**：`_n_cd26wFpw`（完全不变）

原因：L665 和 L671 条件不满足，跳过所有后处理。

### 示例 4：时间戳处理

**输入**：`"New World record at 0:12:34"`
**代码路径**：`restricted=False, is_id=NO_DEFAULT`
**输出**：`New World record at 0_12_34`

转换过程：L663 先于 replace_insane 执行，`0:12:34` 中的 `:` → `_`。

### 示例 5：冲突处理 — overwrites=None（默认）

**配置**：`overwrites=None`（默认）
**现有视频文件**：`./Video Title [dQw4w9WgXcQ].mp4`
**结果**：
- 第三层：`existing_video_file` 使用 `default_overwrite=False` → 不覆盖 → 跳过下载
- 辅助文件（description 等）：`default_overwrite=True` → 覆盖
- 最终：保留旧视频文件，辅助文件被覆盖

### 示例 6：冲突处理 — overwrites=True

**配置**：`overwrites=True`
**现有文件**：`./Video Title [dQw4w9WgXcQ].mp4`
**结果**：
- 第三层：删除旧文件，下载新文件
- 第四层：移动时无冲突
- 最终：产生新文件，写入归档
