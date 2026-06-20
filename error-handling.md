# yt-dlp 错误码与异常分类处理层级分析

## 1. 异常类层次结构

### 1.1 核心基类

**`YoutubeDLError`** ([_utils.py:L968-L977](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L968-L977))

所有 yt-dlp 异常的基类，包含 `msg` 属性存储错误消息。

### 1.2 主要异常子类

| 异常类 | 定义位置 | 用途 |
|--------|---------|------|
| **`ExtractorError`** | [_utils.py:L980-L1022](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L980-L1022) | 信息提取过程中的错误 |
| **`DownloadError`** | [_utils.py:L1057-L1068](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1057-L1068) | 下载过程中的错误 |
| **`PostProcessingError`** | [_utils.py:L1094-L1099](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1094-L1099) | 后处理过程中的错误 |
| **`DownloadCancelled`** | [_utils.py:L1102-L1119](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1102-L1119) | 下载被取消（含子类：`ExistingVideoReached`, `RejectedVideoReached`, `MaxDownloadsReached`） |
| **`UnavailableVideoError`** | [_utils.py:L1138-L1149](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1138-L1149) | 请求的格式不可用 |
| **`ContentTooShortError`** | [_utils.py:L1152-L1164](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1152-L1164) | 下载内容太短 |
| **`EntryNotInPlaylist`** | [_utils.py:L1071-L1077](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1071-L1077) | 条目不在播放列表中 |
| **`SameFileError`** | [_utils.py:L1080-L1091](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1080-L1091) | 多个文件需要下载到同一文件 |
| **`ReExtractInfo`** | [_utils.py:L1122-L1127](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1122-L1127) | 需要重新提取视频信息 |

### 1.3 ExtractorError 子类

| 异常类 | 定义位置 | 用途 |
|--------|---------|------|
| **`UnsupportedError`** | [_utils.py:L1024-L1028](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1024-L1028) | 不支持的URL |
| **`RegexNotFoundError`** | [_utils.py:L1031-L1033](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1031-L1033) | 正则表达式匹配失败 |
| **`GeoRestrictedError`** | [_utils.py:L1036-L1046](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1036-L1046) | 地理限制错误 |
| **`UserNotLive`** | [_utils.py:L1049-L1054](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1049-L1054) | 频道/用户未直播 |

### 1.4 网络异常层次

**`RequestError`** ([exceptions.py:L11-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/networking/exceptions.py#L11-L22))

```
RequestError
├── UnsupportedRequest    - 处理器无法处理请求
├── NoSupportingHandlers  - 没有处理器能支持请求
├── TransportError        - 网络传输错误
│   ├── IncompleteRead    - 读取不完整
│   ├── SSLError          - SSL错误
│   │   └── CertificateVerifyError - 证书验证失败
│   └── ProxyError        - 代理错误
└── HTTPError             - HTTP错误（含status, reason, redirect_loop）
```

## 2. 可忽略错误边界（expected 属性）

### 2.1 `expected=True` - 预期错误（非Bug，可忽略）

这些错误是正常业务流程中可能出现的，不是 yt-dlp 的 bug：

**自动标记为 expected=True：**
- 所有网络异常（`network_exceptions` = HTTPError + TransportError）
  见 [_utils.py:L987-L989](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L987-L989)

**显式标记为 expected=True：**
- `UnsupportedError` - 不支持的URL
- `GeoRestrictedError` - 地理限制
- `UserNotLive` - 用户未直播
- `raise_login_required()` - 需要登录
  见 [common.py:L1257](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L1257)
- `raise_geo_restricted()` - 地理限制
  见 [common.py:L1266](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L1266)
- DRM保护的视频（`_has_drm=True`）
  见 [YoutubeDL.py:L1191](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L1191)
- 网络错误导致的提取失败
  见 [common.py:L785](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L785)

### 2.2 `expected=False` - 非预期错误（可能是Bug）

- 未明确标记为 expected 的 `ExtractorError`
- 这些错误会在消息末尾附加 `bug_reports_message()`，提示用户报告bug
  见 [_utils.py:L1009](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L1009)

## 3. 致命失败边界

### 3.1 退出码定义

| 退出码 | 含义 | 设置位置 |
|--------|------|---------|
| `0` | 成功 | [YoutubeDL.py:L647](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L647) |
| `1` | 通用错误（下载/提取/后处理错误） | [YoutubeDL.py:L1100](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L1100) |
| `2` | 命令行参数错误（OptParseError） | [__init__.py:L1093](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L1093) |
| `100` | 更新失败 | [__init__.py:L995](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L995) |
| `101` | 下载被取消（DownloadCancelled） | [__init__.py:L1074](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L1074) |

### 3.2 不可忽略的致命错误

无论 `ignoreerrors` 设置如何，这些错误都会导致程序终止：

- `SameFileError` - 输出文件名冲突
  见 [__init__.py:L1083-L1084](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L1083-L1084)
- `CookieLoadError` - Cookie加载错误
  见 [__init__.py:L1081](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L1081)
- `UnsafeExecExpansionError` - 不安全的执行扩展错误
  见 [__init__.py:L1081](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L1081)
- `KeyboardInterrupt` - 用户中断
  见 [__init__.py:L1085-L1086](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L1085-L1086)
- `BrokenPipeError` - 管道破裂
  见 [__init__.py:L1087-L1091](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L1087-L1091)

## 4. `ignoreerrors` 参数层级

### 4.1 三个取值的边界

| 值 | 含义 | 下载错误 | 后处理错误 | 适用场景 |
|----|------|---------|-----------|---------|
| `False` | 遇到任何错误立即停止 | ❌ 终止 | ❌ 终止 | API默认 |
| `'only_download'` | 仅忽略下载错误 | ✅ 继续 | ❌ 终止 | CLI默认 |
| `True` | 忽略所有错误 | ✅ 继续 | ✅ 继续 | 批量下载 |

**关键代码位置：**
- 后处理错误处理：[YoutubeDL.py:L3804-L3809](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L3804-L3809)
- 字幕下载错误处理：[YoutubeDL.py:L4489-L4493](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L4489-L4493)
- 通用错误处理：[YoutubeDL.py:L1094-L1100](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L1094-L1100)

### 4.2 `ignore_no_formats_error` 特殊参数

- 忽略 "No video formats found" 错误
- 用于仅提取元数据而不下载的场景
- 见 [YoutubeDL.py:L257-L259](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L257-L259)

## 5. `fatal` 参数边界

### 5.1 `_error_or_warning` 方法

**提取器中：** [common.py:L4067-L4071](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L4067-L4071)

```python
def _error_or_warning(self, err, _count=None, _retries=0, *, fatal=True):
    RetryManager.report_retry(
        err, _count or int(fatal), _retries,
        info=self.to_screen, warn=self.report_warning,
        error=None if fatal else self.report_warning,  # 关键：fatal=False时用warning替代error
        sleep_func=...)
```

**下载器中：** [common.py:L410-L418](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/downloader/common.py#L410-L418)

```python
def report_retry(self, err, count, retries, frag_index=NO_DEFAULT, fatal=True):
    RetryManager.report_retry(
        err, count, retries,
        warn=lambda msg: self.__to_screen(f'[download] Got error: {msg}'),
        error=IDENTITY if not fatal else lambda e: self.report_error(f'\r[download] Got error: {e}'),
        ...)
```

### 5.2 `fatal` 在提取方法中的边界

| `fatal` 值 | 匹配失败时行为 | 示例方法 |
|-----------|---------------|---------|
| `True` | 抛出 `RegexNotFoundError` | `_search_regex`, `xpath_element`, `_download_json` |
| `False` | 仅发出警告，返回 `None` 或默认值 | 可选字段提取、备用格式解析 |

**示例：** [common.py:L1344-L1347](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L1344-L1347)
```python
elif fatal:
    raise RegexNotFoundError(f'Unable to extract {_name}')
else:
    self.report_warning(f'unable to extract {_name}' + bug_reports_message())
    return None
```

## 6. 用户提示边界（Warning vs Error）

### 6.1 Warning（警告）- 黄色，可继续

**适用场景：**
- 可恢复的临时性问题
- 配置不推荐但仍可工作
- 可选功能失败（缩略图、字幕、非关键字段）
- 非致命的解析错误（fatal=False）
- 重试过程中的通知
- 弃用功能提示

**代码路径：**
- `YoutubeDL.report_warning()` [YoutubeDL.py:L1134-L1144](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L1134-L1144)
- `InfoExtractor.report_warning()` [common.py:L1207-L1214](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L1207-L1214)

### 6.2 Error（错误）- 红色，可能终止

**适用场景：**
- 无法继续的核心功能失败
- 需要用户干预的问题（登录、地理限制、付费内容）
- 配置错误导致无法执行
- 重试耗尽后的最终失败
- DRM保护内容

**代码路径：**
- `YoutubeDL.report_error()` [YoutubeDL.py:L1155-L1160](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L1155-L1160)
- `YoutubeDL.trouble()` [YoutubeDL.py:L1080-L1100](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L1080-L1100)

### 6.3 特殊用户提示场景

**登录提示边界：** [common.py:L1249-L1257](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L1249-L1257)
- `metadata_available=True` 且 `ignore_no_formats_error=True` 时，降级为 Warning
- 否则抛出 Error（expected=True）

**地理限制提示边界：** [common.py:L1259-L1266](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L1259-L1266)
- 同上，元数据可用时可降级为 Warning
- 包含VPN/代理使用建议

**无格式提示边界：** [common.py:L1268-L1275](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L1268-L1275)
- `expected=True` 且 `ignore_no_formats_error=True` 时，降级为 Warning
- 否则抛出 Error

## 7. RetryManager 重试机制

### 7.1 核心逻辑

**定义：** [_utils.py:L5242-L5295](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L5242-L5295)

```python
for retry in RetryManager(retries, error_callback, fatal=True):
    try:
        # 操作代码
    except SomeException as err:
        retry.error = err  # 设置错误触发重试
        continue
```

### 7.2 重试耗尽后的行为

| `fatal` | `error` callback | 行为 |
|---------|-----------------|------|
| `True` | `None` | 重新抛出异常 |
| `True` | 函数 | 调用 error 函数（通常抛出错误） |
| `False` | 函数 | 调用 error 函数（通常发出警告） |

**关键代码：** [_utils.py:L5281-L5284](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L5281-L5284)
```python
if count > retries:
    if error:
        return error(f'{e}. Giving up after {count - 1} retries') if count > 1 else error(str(e))
    raise e
```

## 8. 错误处理流程图

```
异常发生
   ↓
┌─ 是否为 network_exceptions? ──是──→ expected=True ──┐
│                                                      ↓
│  否                                                   │
│   ↓                                                   │
└─→ 检查 expected 参数                                  │
     ↓                                                 │
┌─→ ExtractorError 消息生成                             │
│    (expected=False 时附加 bug 报告)                  │
│                                                      │
├──────────────────────────────────────────────────────┘
│
↓
进入 RetryManager 重试循环
   ↓
┌─ 重试次数 < 最大重试? ──是──→ 报告重试 → 等待 → 重试
│
│  否
│   ↓
└─→ 检查 fatal 参数
     ↓
┌─ fatal=True? ──是──→ 调用 error callback → 抛出错误
│
│  否
│   ↓
└─→ 调用 warning callback → 发出警告 → 返回 None/默认
     ↓
┌─ ignoreerrors 检查 ────────────────────────────┐
│                                                │
│  False → 抛出 DownloadError → 程序终止         │
│  'only_download' → 下载错误继续，后处理终止     │
│  True → 所有错误继续，设置 _download_retcode=1 │
└────────────────────────────────────────────────┘
```

## 9. 关键代码参考位置汇总

| 功能 | 文件位置 |
|------|---------|
| 异常类定义 | [yt_dlp/utils/_utils.py:L968-L1188](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L968-L1188) |
| 网络异常类 | [yt_dlp/networking/exceptions.py](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/networking/exceptions.py) |
| 核心错误处理 | [yt_dlp/YoutubeDL.py:L1080-L1160](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/YoutubeDL.py#L1080-L1160) |
| 提取器错误处理 | [yt_dlp/extractor/common.py:L1207-L1275](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/extractor/common.py#L1207-L1275) |
| 下载器重试报告 | [yt_dlp/downloader/common.py:L410-L418](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/downloader/common.py#L410-L418) |
| RetryManager | [yt_dlp/utils/_utils.py:L5242-L5295](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/utils/_utils.py#L5242-L5295) |
| 主入口退出码 | [yt_dlp/__init__.py:L1077-L1094](file:///d:/fz/0601-2/solo-dogfeeding/code/99-yt-dlp/yt_dlp/__init__.py#L1077-L1094) |
