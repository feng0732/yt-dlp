# yt-dlp 错误码与异常分类处理层级分析

> 源码引用说明：本文档使用相对路径的可迁移写法，基准目录为项目根目录 `99-yt-dlp/`。
> 例如 `yt_dlp/utils/_utils.py#L968-L977` 对应文件 `yt_dlp/utils/_utils.py` 第 968-977 行。

## 1. 异常类层次结构

### 1.1 核心基类

**`YoutubeDLError`** — `yt_dlp/utils/_utils.py#L968-L977`

所有 yt-dlp 异常的基类，包含 `msg` 属性存储错误消息。

### 1.2 主要异常子类

| 异常类 | 定义位置 | 用途 |
|--------|---------|------|
| **`ExtractorError`** | `yt_dlp/utils/_utils.py#L980-L1022` | 信息提取过程中的错误 |
| **`DownloadError`** | `yt_dlp/utils/_utils.py#L1057-L1068` | 下载过程中的错误 |
| **`PostProcessingError`** | `yt_dlp/utils/_utils.py#L1094-L1099` | 后处理过程中的错误 |
| **`DownloadCancelled`** | `yt_dlp/utils/_utils.py#L1102-L1119` | 下载被取消（含子类：`ExistingVideoReached`, `RejectedVideoReached`, `MaxDownloadsReached`） |
| **`UnavailableVideoError`** | `yt_dlp/utils/_utils.py#L1138-L1149` | 请求的格式不可用 |
| **`ContentTooShortError`** | `yt_dlp/utils/_utils.py#L1152-L1164` | 下载内容太短 |
| **`EntryNotInPlaylist`** | `yt_dlp/utils/_utils.py#L1071-L1077` | 条目不在播放列表中 |
| **`SameFileError`** | `yt_dlp/utils/_utils.py#L1080-L1091` | 多个文件需要下载到同一文件 |
| **`ReExtractInfo`** | `yt_dlp/utils/_utils.py#L1122-L1127` | 需要重新提取视频信息 |

### 1.3 ExtractorError 子类

| 异常类 | 定义位置 | 用途 |
|--------|---------|------|
| **`UnsupportedError`** | `yt_dlp/utils/_utils.py#L1024-L1028` | 不支持的URL |
| **`RegexNotFoundError`** | `yt_dlp/utils/_utils.py#L1031-L1033` | 正则表达式匹配失败 |
| **`GeoRestrictedError`** | `yt_dlp/utils/_utils.py#L1036-L1046` | 地理限制错误 |
| **`UserNotLive`** | `yt_dlp/utils/_utils.py#L1049-L1054` | 频道/用户未直播 |

### 1.4 网络异常层次

**`RequestError`** — `yt_dlp/networking/exceptions.py#L11-L22`

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

这些错误是正常业务流程中可能出现的，不是 yt-dlp 的 bug。

**自动标记为 expected=True：**
- 所有网络异常（`network_exceptions` = HTTPError + TransportError）
  见 `yt_dlp/utils/_utils.py#L987-L989`

**显式标记为 expected=True：**
- `UnsupportedError` - 不支持的URL
- `GeoRestrictedError` - 地理限制
- `UserNotLive` - 用户未直播
- `raise_login_required()` - 需要登录
  见 `yt_dlp/extractor/common.py#L1257`
- `raise_geo_restricted()` - 地理限制
  见 `yt_dlp/extractor/common.py#L1266`
- DRM保护的视频（`_has_drm=True`）
  见 `yt_dlp/YoutubeDL.py#L1191`
- 网络错误导致的提取失败
  见 `yt_dlp/extractor/common.py#L785`

### 2.2 `expected=False` - 非预期错误（可能是Bug）

- 未明确标记为 expected 的 `ExtractorError`
- 这些错误会在消息末尾附加 `bug_reports_message()`，提示用户报告bug
  见 `yt_dlp/utils/_utils.py#L1009`

## 3. 致命失败边界

### 3.1 退出码定义

| 退出码 | 含义 | 设置位置 |
|--------|------|---------|
| `0` | 成功 | `yt_dlp/YoutubeDL.py#L647` |
| `1` | 通用错误（下载/提取/后处理错误） | `yt_dlp/YoutubeDL.py#L1100` |
| `2` | 命令行参数错误（OptParseError） | `yt_dlp/__init__.py#L1093` |
| `100` | 更新失败 | `yt_dlp/__init__.py#L995` |
| `101` | 下载被取消（DownloadCancelled） | `yt_dlp/__init__.py#L1074` |

### 3.2 不可忽略的致命错误

无论 `ignoreerrors` 设置如何，这些错误都会导致程序终止：

- `SameFileError` - 输出文件名冲突
  见 `yt_dlp/__init__.py#L1083-L1084`
- `CookieLoadError` - Cookie加载错误
  见 `yt_dlp/__init__.py#L1081`
- `UnsafeExecExpansionError` - 不安全的执行扩展错误
  见 `yt_dlp/__init__.py#L1081`
- `KeyboardInterrupt` - 用户中断
  见 `yt_dlp/__init__.py#L1085-L1086`
- `BrokenPipeError` - 管道破裂
  见 `yt_dlp/__init__.py#L1087-L1091`

## 4. `ignoreerrors` 参数层级

### 4.1 CLI 默认值来源

**关键发现：`ignoreerrors` 的默认值是通过 compat option 机制间接设置的。**

设置链路：
1. `options.py` 中三个选项都未设置 `default` 值（`--ignore-errors`/`--no-abort-on-error`/`--abort-on-error`），初始值为 `None`
   见 `yt_dlp/options.py#L380-L391`
2. `__init__.py` 中通过 `set_default_compat()` 函数设置默认值
   ```python
   set_default_compat('abort-on-error', 'ignoreerrors', 'only_download')
   ```
   见 `yt_dlp/__init__.py#L155`
3. `set_default_compat()` 逻辑：
   - 如果 `opts.ignoreerrors is None`，设为 `'only_download'`
   - 如果 `compat_opts` 包含 `'abort-on-error'`，且值为 `None`，设为 `False`
   见 `yt_dlp/__init__.py#L141-L153`

**API vs CLI 默认值差异：**

| 运行模式 | `ignoreerrors` 默认值 | 原因 |
|---------|---------------------|------|
| CLI | `'only_download'` | 通过 `set_default_compat` 设置 |
| API | `False` | 不经过 CLI 选项解析，`YoutubeDL` 类内部默认为 `None`/`False` |

**`IN_CLI` 标记：**
- `main()` 函数入口设置 `IN_CLI.value = True`
  见 `yt_dlp/__init__.py#L1078`
- 用于区分 CLI 模式和 API 模式的行为差异
  见 `yt_dlp/globals.py#L31`

### 4.2 三个取值的边界

| 值 | 含义 | 下载错误 | 后处理错误 | 适用场景 |
|----|------|---------|-----------|---------|
| `False` | 遇到任何错误立即停止 | ❌ 终止 | ❌ 终止 | API默认 / `--abort-on-error` |
| `'only_download'` | 仅忽略下载错误 | ✅ 继续 | ❌ 终止 | CLI默认 / `--no-abort-on-error` |
| `True` | 忽略所有错误 | ✅ 继续 | ✅ 继续 | `--ignore-errors` / 批量下载 |

**关键代码位置：**
- 后处理错误处理：`yt_dlp/YoutubeDL.py#L3804-L3809`
- 字幕下载错误处理：`yt_dlp/YoutubeDL.py#L4489-L4493`
- 通用错误处理（`trouble` 方法）：`yt_dlp/YoutubeDL.py#L1094-L1100`

### 4.3 `ignore_no_formats_error` 特殊参数

- 默认值：`False`（CLI 和 API 均为 `False`）
  见 `yt_dlp/options.py#L1247`
- 忽略 "No video formats found" 错误
- 用于仅提取元数据而不下载的场景
- 见 `yt_dlp/YoutubeDL.py#L257-L259`

**提取器级别的 `ignore_no_formats_error` 覆盖：**
- 某些直播/事件类提取器会设置 `'ignore_no_formats_error': True`
  （如 `uliza.py`, `pialive.py`, `gamejolt.py`, `eplus.py` 等）
- 这些站点通常有元数据但流地址需要等待或在特定时间才可用

## 5. `wait_for_video` 对错误降级的影响

### 5.1 `wait_for_video` 参数说明

- 用途：等待预定直播/排期视频变为可用
- 取值：`(min_wait, max_wait)` 秒数元组
- 见 `yt_dlp/YoutubeDL.py#L384`

### 5.2 三类错误的降级逻辑

`wait_for_video` 与 `ignore_no_formats_error` 是**或（OR）**关系，
只要其中一个为真，就会触发错误降级（Error → Warning）。

**统一降级条件：**
```python
metadata_available and (ignore_no_formats_error or wait_for_video)
```

降级逻辑都在 `InfoExtractor` 类中实现，见 `yt_dlp/extractor/common.py`。

#### 登录提示降级

位置：`yt_dlp/extractor/common.py#L1249-L1257`

```python
def raise_login_required(self, msg='...', metadata_available=False, ...):
    if metadata_available and (
            self.get_param('ignore_no_formats_error') or self.get_param('wait_for_video')):
        self.report_warning(msg)   # 降级为警告
        return
    raise ExtractorError(msg, expected=True)  # 抛出错误
```

**触发条件：**
- `metadata_available=True`（视频元数据已获取，但需要登录才能看视频）
- 且 `ignore_no_formats_error=True` 或 `wait_for_video` 有值

**行为变化：**
- 未满足条件 → 抛出 `ExtractorError(expected=True)` → 终止当前视频
- 满足条件 → 仅发出 Warning → 继续流程（仅下载元数据）

#### 地理限制降级

位置：`yt_dlp/extractor/common.py#L1259-L1266`

```python
def raise_geo_restricted(self, msg='...', countries=None, metadata_available=False):
    if metadata_available and (
            self.get_param('ignore_no_formats_error') or self.get_param('wait_for_video')):
        self.report_warning(msg)   # 降级为警告
    else:
        raise GeoRestrictedError(msg, countries=countries)  # 抛出错误
```

**行为与登录提示相同**，但抛出的是 `GeoRestrictedError` 子类。

#### 无格式提示降级

位置：`yt_dlp/extractor/common.py#L1268-L1275`

```python
def raise_no_formats(self, msg, expected=False, video_id=None):
    if expected and (
            self.get_param('ignore_no_formats_error') or self.get_param('wait_for_video')):
        self.report_warning(msg, video_id)  # 降级为警告
    elif isinstance(msg, ExtractorError):
        raise msg
    else:
        raise ExtractorError(msg, expected=expected, video_id=video_id)
```

**注意：** 只有当 `expected=True` 时才会检查降级条件。
如果 `expected=False`（非预期的无格式错误），直接抛出错误，不会降级。

### 5.3 `UserNotLive` 的特殊处理

位置：`yt_dlp/YoutubeDL.py#L1862-L1867`

```python
try:
    ie_result = ie.extract(url)
except UserNotLive as e:
    if process:
        if self.params.get('wait_for_video'):
            self.report_warning(e)  # 先报告警告
        self._wait_for_video()     # 进入等待逻辑
    raise  # 重新抛出，由外层装饰器处理
```

**流程：**
1. 提取器抛出 `UserNotLive`
2. 如果设置了 `wait_for_video`，先报告 Warning
3. 调用 `_wait_for_video()` 进入等待
4. 等待结束后抛出 `ReExtractInfo(expected=True)` 触发重新提取
5. 重新提取成功则继续，失败则由 `_handle_extraction_exceptions` 处理

### 5.4 `_wait_for_video` 等待逻辑

位置：`yt_dlp/YoutubeDL.py#L1753-L1797`

**进入等待的条件（全部满足）：**
- `wait_for_video` 参数已设置
- 结果类型是 `video`
- 结果中没有 `formats` 和 `url`（即没有可下载的格式）

**等待结束方式：**
- 定时到达 → 抛出 `ReExtractInfo('[wait] Wait period ended', expected=True)`
- 用户按 Ctrl+C → 抛出 `ReExtractInfo('[wait] Interrupted by user', expected=True)`

**重新提取机制：**
`_handle_extraction_exceptions` 装饰器捕获 `ReExtractInfo` 后，会重新执行提取函数。
见 `yt_dlp/YoutubeDL.py#L1721-L1751`

### 5.5 无格式选择时的降级

位置：`yt_dlp/YoutubeDL.py#L3089-L3096`

```python
if not formats_to_download:
    if not self.params.get('ignore_no_formats_error'):
        raise ExtractorError(
            'Requested format is not available...',
            expected=True, ...)
    self.report_warning('Requested format is not available')
    formats_to_download = [{}]  # 继续处理，但没有实际格式
```

当请求的格式不可用时，如果 `ignore_no_formats_error=True`，
降级为 Warning 并继续流程（处理元数据、缩略图等）。

## 6. `fatal` 参数边界

### 6.1 `_error_or_warning` 方法

**提取器中：** `yt_dlp/extractor/common.py#L4067-L4071`

```python
def _error_or_warning(self, err, _count=None, _retries=0, *, fatal=True):
    RetryManager.report_retry(
        err, _count or int(fatal), _retries,
        info=self.to_screen, warn=self.report_warning,
        error=None if fatal else self.report_warning,  # 关键：fatal=False时用warning替代error
        sleep_func=...)
```

**下载器中：** `yt_dlp/downloader/common.py#L410-L418`

```python
def report_retry(self, err, count, retries, frag_index=NO_DEFAULT, fatal=True):
    RetryManager.report_retry(
        err, count, retries,
        warn=lambda msg: self.__to_screen(f'[download] Got error: {msg}'),
        error=IDENTITY if not fatal else lambda e: self.report_error(f'\r[download] Got error: {e}'),
        ...)
```

### 6.2 `fatal` 在提取方法中的边界

| `fatal` 值 | 匹配失败时行为 | 示例方法 |
|-----------|---------------|---------|
| `True` | 抛出 `RegexNotFoundError` | `_search_regex`, `xpath_element`, `_download_json` |
| `False` | 仅发出警告，返回 `None` 或默认值 | 可选字段提取、备用格式解析 |

**示例：** `yt_dlp/extractor/common.py#L1344-L1347`
```python
elif fatal:
    raise RegexNotFoundError(f'Unable to extract {_name}')
else:
    self.report_warning(f'unable to extract {_name}' + bug_reports_message())
    return None
```

## 7. 用户提示边界（Warning vs Error）

### 7.1 Warning（警告）- 黄色，可继续

**适用场景：**
- 可恢复的临时性问题
- 配置不推荐但仍可工作
- 可选功能失败（缩略图、字幕、非关键字段）
- 非致命的解析错误（`fatal=False`）
- 重试过程中的通知
- 弃用功能提示
- 登录/地理限制/无格式的降级提示（`wait_for_video` 或 `ignore_no_formats_error` 触发）

**代码路径：**
- `YoutubeDL.report_warning()` — `yt_dlp/YoutubeDL.py#L1134-L1144`
- `InfoExtractor.report_warning()` — `yt_dlp/extractor/common.py#L1207-L1214`

### 7.2 Error（错误）- 红色，可能终止

**适用场景：**
- 无法继续的核心功能失败
- 需要用户干预的问题（登录、地理限制、付费内容）
- 配置错误导致无法执行
- 重试耗尽后的最终失败（`fatal=True`）
- DRM保护内容

**代码路径：**
- `YoutubeDL.report_error()` — `yt_dlp/YoutubeDL.py#L1155-L1160`
- `YoutubeDL.trouble()` — `yt_dlp/YoutubeDL.py#L1080-L1100`

### 7.3 特殊用户提示场景

**登录提示边界：** `yt_dlp/extractor/common.py#L1249-L1257`
- `metadata_available=True` 且（`ignore_no_formats_error=True` 或 `wait_for_video` 有值）时，降级为 Warning
- 否则抛出 Error（`expected=True`）

**地理限制提示边界：** `yt_dlp/extractor/common.py#L1259-L1266`
- 同上，元数据可用时可降级为 Warning
- 包含 VPN/代理使用建议

**无格式提示边界：** `yt_dlp/extractor/common.py#L1268-L1275`
- `expected=True` 且（`ignore_no_formats_error=True` 或 `wait_for_video` 有值）时，降级为 Warning
- 否则抛出 Error

## 8. RetryManager 重试机制

### 8.1 核心逻辑

**定义：** `yt_dlp/utils/_utils.py#L5242-L5295`

```python
for retry in RetryManager(retries, error_callback, fatal=True):
    try:
        # 操作代码
    except SomeException as err:
        retry.error = err  # 设置错误触发重试
        continue
```

### 8.2 重试耗尽后的行为

| `fatal` | `error` callback | 行为 |
|---------|-----------------|------|
| `True` | `None` | 重新抛出异常 |
| `True` | 函数 | 调用 error 函数（通常抛出错误） |
| `False` | 函数 | 调用 error 函数（通常发出警告） |

**关键代码：** `yt_dlp/utils/_utils.py#L5281-L5284`
```python
if count > retries:
    if error:
        return error(f'{e}. Giving up after {count - 1} retries') if count > 1 else error(str(e))
    raise e
```

## 9. 错误处理完整流程图

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
┌─ 是否为 登录/地理限制/无格式 类型? ──┐
│                                      │
│  是 → 检查 metadata_available        │
│       AND (ignore_no_formats_error   │
│            OR wait_for_video)?       │
│           ↓                          │
│           是 → 降级为 Warning        │
│           否 → 保持 Error            │
│                                      │
└──────────────────────────────────────┘
     ↓
┌─ ignoreerrors 检查 ────────────────────────────┐
│                                                │
│  False → 抛出 DownloadError → 程序终止         │
│  'only_download' → 下载错误继续，后处理终止     │
│  True → 所有错误继续，设置 _download_retcode=1 │
└────────────────────────────────────────────────┘
     ↓
┌─ wait_for_video 检查 (针对 UserNotLive等) ──┐
│                                             │
│  已设置 → 等待 → 抛出 ReExtractInfo → 重试提取 │
│  未设置 → 保持错误状态                       │
└─────────────────────────────────────────────┘
```

## 10. 关键代码参考位置汇总

| 功能 | 文件位置 |
|------|---------|
| 异常类定义 | `yt_dlp/utils/_utils.py#L968-L1188` |
| 网络异常类 | `yt_dlp/networking/exceptions.py` |
| 核心错误处理 | `yt_dlp/YoutubeDL.py#L1080-L1160` |
| CLI 默认值设置 | `yt_dlp/__init__.py#L141-L155` |
| CLI 选项定义 | `yt_dlp/options.py#L380-L391` |
| `IN_CLI` 全局标记 | `yt_dlp/globals.py#L31` |
| 提取器错误处理 | `yt_dlp/extractor/common.py#L1207-L1275` |
| 下载器重试报告 | `yt_dlp/downloader/common.py#L410-L418` |
| RetryManager | `yt_dlp/utils/_utils.py#L5242-L5295` |
| 主入口退出码 | `yt_dlp/__init__.py#L1077-L1094` |
| `_wait_for_video` 逻辑 | `yt_dlp/YoutubeDL.py#L1753-L1797` |
| `UserNotLive` 特殊处理 | `yt_dlp/YoutubeDL.py#L1862-L1867` |
| `_handle_extraction_exceptions` 装饰器 | `yt_dlp/YoutubeDL.py#L1721-L1751` |
| 无格式选择降级 | `yt_dlp/YoutubeDL.py#L3089-L3096` |
| `raise_no_formats` (YoutubeDL) | `yt_dlp/YoutubeDL.py#L1186-L1194` |
| `raise_login_required` | `yt_dlp/extractor/common.py#L1249-L1257` |
| `raise_geo_restricted` | `yt_dlp/extractor/common.py#L1259-L1266` |
| `raise_no_formats` (InfoExtractor) | `yt_dlp/extractor/common.py#L1268-L1275` |
