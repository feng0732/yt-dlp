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

## 第三部分：冲突处理 — 四层机制协作

### 3.1 四层机制总览

四层机制在下载流程的 **不同阶段** 依次触发，共同决定最终文件是否产生。

```
下载启动
    ↓
[第一层] SameFileError 检测
    时机：download() 开始时
    作用：防止多个 URL 写入同一固定文件名
    ↓
[第二层] download_archive 检测（两次）
    时机 1：extract_info() URL 解析时
    时机 2：_match_entry() 过滤时
    作用：基于视频 ID 的去重（不依赖文件名）
    ↓
[第三层] existing_file + overwrites 检测
    时机：process_info() 下载前
    作用：文件系统级别的存在检测
    ↓
[第四层] MoveFilesAfterDownloadPP 检测
    时机：post_process() 后处理时
    作用：临时文件移动到最终位置时的检测
    ↓
最终文件
```

### 3.2 第一层：SameFileError — 固定文件名冲突

**触发位置**：`yt_dlp/YoutubeDL.py#L3697-L3702`

**触发条件（同时满足）：**
1. `len(url_list) > 1` — 有多个 URL
2. `outtmpl != '-'` — 不是输出到 stdout
3. `'%' not in outtmpl` — 模板中没有变量（固定文件名）
4. `max_downloads != 1` — 不止下载一个

**对最终文件的影响**：
- 直接抛出 `SameFileError` 异常
- 下载流程终止，**不产生任何文件**
- 此检测在文件名清理和路径拼接之前，作用于原始模板字符串

### 3.3 第二层：download_archive — 视频 ID 去重

#### 3.3.1 第一次检查：URL 解析时

位置：`yt_dlp/YoutubeDL.py#L1708-L1713`

```python
temp_id = ie.get_temp_id(url)
if temp_id is not None and self.in_download_archive({'id': temp_id, 'ie_key': key}):
    if self.params.get('break_on_existing', False):
        raise ExistingVideoReached
    break
```

此检查发生在 URL 解析阶段，**尚未生成文件名**。

#### 3.3.2 第二次检查：_match_entry 过滤时

位置：`yt_dlp/YoutubeDL.py#L1645-L1650`

```python
if self.in_download_archive(info_dict):
    reason = '... has already been recorded in the archive'
    break_opt, break_err = 'break_on_existing', ExistingVideoReached
```

此检查在 `process_info` 的最开始，也在文件名生成之前。

#### 3.3.3 in_download_archive 实现

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

**对最终文件的影响**：
- 视频被跳过，**不产生下载文件，也不写入归档**
- `_match_entry` 返回非 None 时，`__write_download_archive` 被设为 `'ignore'`
- 如果 `force_write_download_archive=True`，即使跳过也写入归档

#### 3.3.4 归档写入时机

位置：`yt_dlp/YoutubeDL.py#L3140-L3143`

```python
write_archive = {f.get('__write_download_archive', False) for f in downloaded_formats}
if True in write_archive and False not in write_archive:
    self.record_download_archive(info_dict)
```

`__write_download_archive` 的设置时机：

| 场景 | 值 | 位置 |
|------|-----|------|
| 被 `_match_entry` 过滤 | `'ignore'` | `yt_dlp/YoutubeDL.py#L3341` |
| `simulate` 模式 | `force_write_download_archive` | `yt_dlp/YoutubeDL.py#L3371` |
| `skip_download` 模式 | `force_write_download_archive` | `yt_dlp/YoutubeDL.py#L3457` |
| 正常下载完成 | `True` | `yt_dlp/YoutubeDL.py#L3667` |
| `force_write_download_archive=True` | `True` | `yt_dlp/YoutubeDL.py#L3670-L3671` |

写入条件：集合中含 `True` 且不含 `False`（`'ignore'` 不影响判断）。

### 3.4 第三层：existing_file + overwrites — 文件系统检测

#### 3.4.1 overwrites 参数三态

位置：`yt_dlp/YoutubeDL.py#L291-L293`

| 值 | 视频文件（default_overwrite=False） | 辅助文件（default_overwrite=True） |
|----|--------------------------------------|--------------------------------------|
| `True` | 覆盖 | 覆盖 |
| `None`（默认） | 不覆盖 | 覆盖 |
| `False` | 不覆盖 | 不覆盖 |

**原理**：`existing_file` 的 `default_overwrite` 参数决定 `overwrites=None` 时的行为。视频文件传入 `default_overwrite=False`，辅助文件传入 `default_overwrite=True`。

#### 3.4.2 existing_file 核心逻辑

位置：`yt_dlp/YoutubeDL.py#L3320-L3328`

```python
def existing_file(self, filepaths, *, default_overwrite=True):
    existing_files = list(filter(os.path.exists, orderedSet(filepaths)))
    if existing_files and not self.params.get('overwrites', default_overwrite):
        return existing_files[0]  # 不覆盖：返回已存在文件
    
    for file in existing_files:
        self.report_file_delete(file)
        os.remove(file)  # 覆盖：删除现有文件
    return None
```

**关键**：返回非 None 表示文件已存在且不覆盖；返回 None 表示可以下载。

#### 3.4.3 视频文件检测：existing_video_file

位置：`yt_dlp/YoutubeDL.py#L3463-L3470`

```python
def existing_video_file(*filepaths):
    ext = info_dict.get('ext')
    converted = lambda file: replace_extension(file, final_ext or ext, ext)
    file = self.existing_file(
        itertools.chain(*zip(map(converted, filepaths), filepaths)),
        default_overwrite=False  # 视频文件默认不覆盖
    )
```

同时检查原始扩展名和最终扩展名（如 `.webm` 和 `.mkv`）。

**调用时机**：
- 多格式合并：`yt_dlp/YoutubeDL.py#L3508`
- 单文件下载：`yt_dlp/YoutubeDL.py#L3575`

**对最终文件的影响**：
- 文件已存在且 `overwrites` 不为 True → 跳过下载，报告已存在，使用现有文件
- 文件已存在且 `overwrites=True` → 删除现有文件，重新下载
- 返回 `temp_filename`（`--no-part` 场景）→ 继续下载（断点续传）

#### 3.4.4 辅助文件检测

| 文件类型 | 检测位置 | default_overwrite |
|----------|----------|-------------------|
| 信息 JSON | `yt_dlp/YoutubeDL.py#L4396` | `True`（覆盖） |
| 描述文件 | `yt_dlp/YoutubeDL.py#L4425` | `True`（覆盖） |
| 网络快捷方式 | `yt_dlp/YoutubeDL.py#L3418` | `True`（覆盖） |

**第三层与第二层的关系**：第二层基于视频 ID 去重，不关心文件名；第三层基于文件系统检测，不关心视频 ID。两者独立——即使视频不在归档中，同名文件也可能已存在。

### 3.5 第四层：MoveFilesAfterDownloadPP — 文件移动检测

位置：`yt_dlp/postprocessor/movefilesafterdownload.py#L21-L52`

```python
for oldfile, newfile in info['__files_to_move'].items():
    if os.path.abspath(oldfile) == os.path.abspath(newfile):
        continue  # 源和目标是同一文件，跳过
    if not os.path.exists(oldfile):
        continue  # 源文件不存在，跳过
    if os.path.exists(newfile):
        if self.get_param('overwrites', True):
            os.remove(newfile)  # 覆盖：删除目标
        else:
            continue  # 不覆盖：跳过移动，临时文件留在原地
    shutil.move(oldfile, newfile)
```

**对最终文件的影响**：
- 不覆盖时：临时文件保留在临时目录，**最终位置不产生文件**
- 覆盖时：删除目标文件，移动成功，**最终位置产生文件**
- 源和目标相同时：无需移动，文件已在最终位置

**第四层与第三层的关系**：第三层检测在下载前，决定是否下载；第四层检测在下载后移动时，决定是否将临时文件移动到最终位置。两层的 `overwrites` 逻辑一致但时机不同。

**注意**：`MoveFilesAfterDownloadPP` 中 `get_param('overwrites', True)` 的默认值是 `True`，与 `existing_file` 不同。这意味着在 `overwrites=None`（默认）时，文件移动**允许覆盖**。

### 3.6 四层机制协作时序图

```
YoutubeDL.download(url_list)
├─ 第一层: SameFileError 检测 → 失败则终止，无任何文件
└─ 对每个 URL:
    YoutubeDL.extract_info(url)
    ├─ 第二层(1): in_download_archive(temp_id) → 命中则跳过此 URL
    └─ YoutubeDL.process_ie_result()
        └─ YoutubeDL.process_info(info_dict)
            ├─ 第二层(2): _match_entry() → in_download_archive 命中则跳过
            │   命中时 __write_download_archive = 'ignore'
            ├─ prepare_filename() → full_filename（经过完整清理流程）
            ├─ prepare_filename('temp') → temp_filename
            ├─ 写辅助文件（各自检测 overwrites，default_overwrite=True）
            ├─ 第三层: existing_video_file() → 文件已存在且不覆盖则跳过下载
            │   返回已有文件路径 → report_file_already_downloaded
            │   返回 None → 继续下载到 temp_filename
            └─ YoutubeDL.post_process()
                ├─ 格式转换等后处理
                └─ 第四层: MoveFilesAfterDownloadPP
                    ├─ overwrites=True → 删除目标，移动文件
                    ├─ overwrites=False/None → 目标存在则跳过移动
                    └─ 源=目标 → 无需移动
```

### 3.7 典型场景的最终文件状态

| 场景 | archive | overwrites | 第三层结果 | 第四层结果 | 最终文件状态 |
|------|---------|------------|-----------|-----------|-------------|
| 新视频，无同名文件 | - | 任意 | 无现有文件，下载 | 无冲突，移动 | ✓ 产生新文件，写入 archive |
| 新视频，有同名文件 | - | True | 删除旧文件，下载 | 覆盖，移动 | ✓ 覆盖旧文件，写入 archive |
| 新视频，有同名文件 | - | False | 跳过下载 | 不触发 | ✗ 保留旧文件，不写 archive |
| 新视频，有同名文件 | - | None | 跳过下载（视频 default=False） | 不触发 | ✗ 保留旧视频文件，辅助文件可能被覆盖 |
| 视频已在 archive | 包含 | 任意 | 不到达此层 | 不到达此层 | ✗ 第二层跳过，不写 archive |
| 固定文件名 + 多 URL | - | 任意 | 不到达此层 | 不到达此层 | ✗ 第一层异常，无文件 |

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
