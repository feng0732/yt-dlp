# yt-dlp 输出模板到文件命名处理路径详解

## 整体处理流程概览

从输出模板字符串到最终文件名的完整处理路径如下：

```
输出模板字符串 (outtmpl)
    ↓
1. _parse_outtmpl()  —— 模板解析与默认值填充
    ↓
2. _outtmpl_expandpath()  —— 环境变量与波浪号展开
    ↓
3. prepare_outtmpl()  —— 模板变量解析、对象遍历、格式转换
    ↓
4. evaluate_outtmpl()  —— 模板最终求值
    ↓
5. _prepare_filename()  —— 文件名特殊处理（扩展名、长度裁剪）
    ↓
6. get_output_path()  —— 路径拼接与路径清理
    ↓
7. existing_file() / download_archive  —— 冲突检测与处理
    ↓
最终文件名
```

---

## 一、输出模板解析与默认值

### 1.1 模板类型与默认值

定义在 [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/_utils.py#L2861-L2877)：

```python
DEFAULT_OUTTMPL = {
    'default': '%(title)s [%(id)s].%(ext)s',
    'chapter': '%(title)s - %(section_number)03d %(section_title)s [%(id)s].%(ext)s',
}
OUTTMPL_TYPES = {
    'chapter': None,
    'subtitle': None,
    'thumbnail': None,
    'description': 'description',
    'annotation': 'annotations.xml',
    'infojson': 'info.json',
    'link': None,
    'pl_video': None,
    'pl_thumbnail': None,
    'pl_description': 'description',
    'pl_infojson': 'info.json',
}
```

### 1.2 解析函数 `_parse_outtmpl`

位置：[YoutubeDL.py#L1201-L1209](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1201-L1209)

处理逻辑：
1. 如果 `outtmpl` 不是字典，包装为 `{'default': outtmpl}`
2. 对 `DEFAULT_OUTTMPL` 中的每个键，如果用户未指定则使用默认值
3. 如果启用了 `restrictfilenames`，对默认模板进行简单的空格替换：
   - `' - '` → `' '`
   - `' '` → `'-'`

---

## 二、环境变量与路径展开

### 2.1 `_outtmpl_expandpath` 方法

位置：[YoutubeDL.py#L1221-L1233](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1221-L1233)

**关键设计：使用随机分隔符保护模板变量不被展开**

处理步骤：
1. 生成 32 位随机字母字符串作为分隔符 `sep`
2. 将模板中的 `%%` 替换为 `%{sep}%`，`$$` 替换为 `${sep}$`
3. 调用 `expand_path()` 展开环境变量和 `~`
4. 移除分隔符，恢复 `%%` 和 `$$`

这样做的原因：
- `expand_path` 会展开 `%VAR%` 和 `$VAR` 形式的环境变量
- 但模板变量 `%(title)s` 中的 `%` 和元数据中的 `$` 字符不应被展开
- 通过临时占位符保护，确保只有模板字符串本身的路径变量被展开

### 2.2 `expand_path` 函数

位置：[_utils.py#L768-L770](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/_utils.py#L768-L770)

```python
def expand_path(s):
    """Expand shell variables and ~"""
    return os.path.expandvars(compat_expanduser(s))
```

---

## 三、模板变量展开（核心）

### 3.1 入口函数链

| 函数 | 位置 | 作用 |
|------|------|------|
| `evaluate_outtmpl` | [YoutubeDL.py#L1516-L1518](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1516-L1518) | 对外接口，调用 prepare_outtmpl 后执行格式化 |
| `prepare_outtmpl` | [YoutubeDL.py#L1263-L1514](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1263-L1514) | 核心：解析模板语法、遍历 info_dict、应用格式转换 |

### 3.2 `prepare_outtmpl` 详细处理流程

#### 3.2.1 自动生成字段

在模板展开前，会向 `info_dict` 中注入以下自动生成的字段：

| 字段 | 说明 |
|------|------|
| `epoch` | Unix 时间戳，整个过程中保持一致 |
| `duration_string` | 格式化的时长字符串（文件名中用 `-` 替代 `:`） |
| `autonumber` | 自动编号，从 `autonumber_start` 开始 |
| `video_autonumber` | 视频自动编号 |
| `resolution` | 分辨率字符串（如 `1080p`） |

#### 3.2.2 字段大小兼容性映射

对于以下字段，`%(field)s` 会自动转换为 `%(field)0Nd` 格式：
- `playlist_index`: 位数由 `__last_playlist_index` 决定
- `playlist_autonumber`: 位数由 `n_entries` 决定
- `autonumber`: 位数由 `autonumber_size` 决定（默认 5 位）

#### 3.2.3 模板语法解析

使用正则表达式 `EXTERNAL_FORMAT_RE` 匹配 `%(key)format` 形式的模板变量。

内部格式正则 `INTERNAL_FORMAT_RE` 支持丰富的语法：

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

支持的语法元素：
- **对象遍历**：`%(formats.0.id)s` —— 使用点号遍历嵌套对象
- **取反**：`-%(field)s` —— 数值取反
- **数学运算**：`%(field+10)d` —— 支持 `+`、`-`、`*`
- **日期格式化**：`%(upload_date>%Y-%m-%d)s` —— 使用 `>` 指定 strftime 格式
- **替代字段**：`%(field1,field2)s` —— 使用 `,` 分隔，第一个为空则尝试下一个
- **替换格式**：`%(field&replacement)s` —— 使用 `&`，值非空时使用 `replacement` 格式化
- **默认值**：`%(field|default)s` —— 使用 `|`，值为空时使用默认值

#### 3.2.4 对象遍历 `_traverse_infodict`

位置：[YoutubeDL.py#L1327-L1341](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1327-L1341)

调用 `traverse_obj(info_dict, fields, traverse_string=True)` 进行深度遍历。

`traverse_obj` 函数（定义在 [traversal.py#L38-L100](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/traversal.py#L38-L100)）支持：
- 字典键访问
- 列表/元组索引访问
- 切片访问
- 集合类型过滤
- 字典转换（`{key1:path1,key2:path2}`）
- 分支路径

#### 3.2.5 格式转换类型

| 格式字符 | 含义 |
|----------|------|
| `s` | 字符串（默认） |
| `d`, `i` | 整数 |
| `f`, `F`, `e`, `E`, `g`, `G` | 浮点数 |
| `r` | repr 表示 |
| `a` | ASCII 表示 |
| `c` | 首字符 |
| `l` | 列表格式（用 `, ` 或 `\n` 连接） |
| `j` | JSON 格式 |
| `h` | HTML 转义 |
| `q` | Shell 引号转义 |
| `B` | 字节格式化 |
| `U` | Unicode 规范化（NFC/NFKC/NFD/NFKD） |
| `D` | 十进制后缀（如 KB、MB） |
| `S` | 文件名清理 |

### 3.3 转义与最终求值

#### 3.3.1 `escape_outtmpl`

位置：[YoutubeDL.py#L1236-L1241](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1236-L1241)

转义模板中剩余的 `%` 字符，避免 Python 字符串格式化时出错。

#### 3.3.2 最终求值

```python
self.escape_outtmpl(outtmpl) % info_dict
```

使用 Python 原生的 `%` 字符串格式化操作完成最终替换。

---

## 四、文件名清理规则

### 4.1 `sanitize_filename` 函数

位置：[_utils.py#L631-L683](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/_utils.py#L631-L683)

函数签名：
```python
def sanitize_filename(s, restricted=False, is_id=NO_DEFAULT):
```

#### 4.1.1 三种模式

| 参数组合 | 模式 | 特点 |
|----------|------|------|
| `restricted=False, is_id=NO_DEFAULT` | 默认模式（新规则） | 保留 Unicode，特殊字符转全角 |
| `restricted=True` | 受限模式 | 仅 ASCII，去除重音 |
| `is_id=True` | ID 模式 | 尽量保持 ID 不变 |
| `is_id=False` | 非 ID 模式 | 常规文件名清理 |

#### 4.1.2 默认模式（新规则）清理规则

1. **时间戳处理**：`0:12:34` → `0_12_34`（用正则匹配时间格式）
2. **全角字符替换**：以下字符替换为对应的 Unicode 全角版本：
   - `"` → `＂` (U+FF02)
   - `*` → `＊` (U+FF0A)
   - `:` → `：` (U+FF1A)
   - `<` → `＜` (U+FF1C)
   - `>` → `＞` (U+FF1E)
   - `?` → `？` (U+FF1F)
   - `|` → `｜` (U+FF5C)
   - `/` → `⧸` (U+29F8)
   - `\` → `⧹` (U+29F9)
3. **控制字符删除**：`?`、ASCII 控制字符（< 32）、DEL（127）直接删除
4. **冒号特殊处理**：非 restricted 模式下替换为 ` -`（空格+连字符）
5. **路径分隔符处理**：`\ / | * < >` 替换为 `_`
6. **重复替换字符去重**：连续的相同替换字符合并为一个
7. **首尾清理**：移除开头和结尾的替换字符、空格、下划线、连字符
8. **空值兜底**：如果结果为空，返回 `_`

#### 4.1.3 受限模式（restricted）清理规则

1. **Unicode 规范化**：使用 NFKC 规范化
2. **重音字符转换**：使用 `ACCENT_CHARS` 映射表将带重音字符转换为 ASCII 对应字符
   - 例如：`Ä` → `A`，`é` → `e`
3. **更多字符删除或替换**：
   - `! & ' ( ) [ ] { } $ ; ` ^ , #` 以及空格 → 删除或替换为 `_`
   - 所有非 ASCII 字符 → 删除（如果是组合字符）或替换为 `_`
4. **特殊处理**：
   - 开头的 `-_` 前缀移除（处理 "外文歌名 - 英文歌名" 的常见情况）
   - 开头的 `-` 替换为 `_`
   - 开头的 `.` 移除
   - 连续 `__` 合并为 `_`
   - 首尾 `_` 去除

#### 4.1.4 ID 模式特殊规则

当 `is_id=True` 时：
- 不执行重复替换字符去重
- 不执行首尾清理
- 不合并连续下划线
- 尽量保持原始 ID 的格式

默认情况下（`is_id=NO_DEFAULT`），根据字段名判断是否为 ID：
- 字段名匹配 `(^|[_.])id(\.|$)` 时视为 ID

位置：[YoutubeDL.py#L1384-L1388](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1384-L1388)

### 4.2 文件名清理的调用时机

在 `prepare_outtmpl` 中，当 `sanitize=True` 时：

```python
if sanitize:
    if fmt[-1] == 'r':
        value, fmt = repr(value), str_fmt
    elif fmt[-1] == 'a':
        value, fmt = ascii(value), str_fmt
    if fmt[-1] in 'csra':
        value = sanitize(last_field, value)
```

即对于 `c`、`s`、`r`、`a` 格式类型，会对值进行文件名清理。

### 4.3 路径清理 `sanitize_path`

位置：[_utils.py#L706-L733](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/_utils.py#L706-L733)

Windows 平台特有处理：
- 解析 UNC 路径、绝对路径、相对路径
- 对每个路径段调用 `_sanitize_path_parts`
- 替换无效字符和结尾的点/空格为 `#`
- 处理 `.` 和 `..` 路径段

---

## 五、文件名生成与后处理

### 5.1 `_prepare_filename`

位置：[YoutubeDL.py#L1521-L1546](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1521-L1546)

处理步骤：
1. 根据 `tmpl_type` 或 `outtmpl` 参数选择模板
2. 调用 `_outtmpl_expandpath` 展开路径
3. 调用 `evaluate_outtmpl` 求值（`sanitize=True`）
4. 处理扩展名：
   - 对于 `''` 和 `'temp'` 类型：处理 `final_ext` 替换
   - 对于其他类型：使用 `OUTTMPL_TYPES` 中定义的扩展名进行强制替换
5. 文件名长度裁剪：如果设置了 `trim_file_name`，裁剪文件名（不含扩展名）到指定长度

### 5.2 `prepare_filename`

位置：[YoutubeDL.py#L1551-L1570](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1551-L1570)

对外接口，在 `_prepare_filename` 基础上：
1. 处理空文件名情况
2. 处理 stdout 输出（`-`）
3. 调用 `get_output_path` 拼接最终路径

### 5.3 `get_output_path`

位置：[YoutubeDL.py#L1211-L1218](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L1211-L1218)

```python
def get_output_path(self, dir_type='', filename=None):
    paths = self.params.get('paths', {})
    path = os.path.join(
        expand_path(paths.get('home', '').strip()),
        expand_path(paths.get(dir_type, '').strip()) if dir_type else '',
        filename or '')
    return sanitize_path(path, force=self.params.get('windowsfilenames'))
```

路径结构：`{home}/{dir_type}/{filename}`

---

## 六、文件命名冲突处理

### 6.1 `overwrites` 参数

位置：[YoutubeDL.py#L291-L293](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L291-L293)

| 值 | 含义 |
|----|------|
| `True` | 覆盖所有视频和元数据文件 |
| `None` | 只覆盖非视频文件（默认行为） |
| `False` | 不覆盖任何文件 |

兼容性：`nooverwrites` 参数与 `overwrites` 互为反义，两者保持同步。

### 6.2 `existing_file` 方法

位置：[YoutubeDL.py#L3320-L3328](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L3320-L3328)

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

逻辑：
1. 过滤出存在的文件路径
2. 如果不允许覆盖且有文件存在，返回第一个存在的文件路径
3. 如果允许覆盖，删除所有已存在的文件并返回 `None`

### 6.3 下载归档 `download_archive`

#### 6.3.1 加载归档

位置：[YoutubeDL.py#L839-L857](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L839-L857)

程序启动时从文件加载已下载视频的 ID 集合。

#### 6.3.2 检查归档 `in_download_archive`

位置：[YoutubeDL.py#L3868-L3874](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L3868-L3874)

检查视频 ID（包括 `_old_archive_ids`）是否在归档中。

#### 6.3.3 记录归档 `record_download_archive`

位置：[YoutubeDL.py#L3876-L3887](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L3876-L3887)

下载完成后将视频 ID 写入归档文件和内存集合。

### 6.4 `SameFileError` 异常

位置：[_utils.py#L1080-L1091](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/_utils.py#L1080-L1091)

触发条件（位置：[YoutubeDL.py#L3697-L3702](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L3697-L3702)）：
- 有多个 URL 需要下载
- 输出模板不是 `-`（stdout）
- 输出模板中不含 `%`（即固定文件名）
- `max_downloads` 不是 1

这种情况下多个文件会写入同一文件名，直接抛出异常。

### 6.5 下载过程中的冲突处理

在 `process_info` 函数（[YoutubeDL.py#L3331](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py#L3331)）中：

1. 生成最终文件名 `full_filename` 和临时文件名 `temp_filename`
2. 写入字幕、缩略图、信息 JSON 等辅助文件时检查覆盖
3. 视频下载前调用 `existing_video_file` 检查是否已存在
4. 下载完成后通过后处理将临时文件移动到最终位置

---

## 七、关键代码速查表

| 功能 | 文件 | 行号 |
|------|------|------|
| 模板解析 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py) | L1201-L1209 |
| 路径展开 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py) | L1221-L1233 |
| 模板准备（核心） | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py) | L1263-L1514 |
| 模板求值 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py) | L1516-L1518 |
| 文件名准备 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py) | L1521-L1570 |
| 输出路径生成 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py) | L1211-L1218 |
| 文件名清理 | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/_utils.py) | L631-L683 |
| 路径清理 | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/_utils.py) | L706-L733 |
| 冲突检测 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py) | L3320-L3328 |
| 下载归档 | [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/YoutubeDL.py) | L3868-L3887 |
| 对象遍历 | [traversal.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/traversal.py) | L38-L100 |
| 默认模板定义 | [_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/95-yt-dlp/yt_dlp/utils/_utils.py) | L2861-L2877 |
