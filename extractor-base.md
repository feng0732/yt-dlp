# 站点提取器公共抽象（InfoExtractor）

> 本文档从代码路径视角，讲清 `InfoExtractor` 基类的**元数据约定**、**下载辅助能力**和**子类边界**。

## 一、类定位

`InfoExtractor` 是所有站点提取器的基类，定义在 [common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/82-yt-dlp/yt_dlp/extractor/common.py#L105-L578)。它的职责是：

- **识别 URL**：判断一个 URL 是否属于当前站点
- **提取元数据**：从网页/API 中解析出视频标题、格式列表等信息
- **封装通用能力**：提供下载网页、解析 JSON、提取 m3u8 等复用方法
- **约定返回格式**：统一返回字典的结构，供 `YoutubeDL` 主流程消费

---

## 二、元数据约定（类属性层）

这些属性是**类级别**的声明，描述"这个提取器是什么、能匹配什么 URL、有什么特性"。框架通过读取这些属性来注册、匹配和测试提取器。

### 2.1 URL 匹配约定

| 属性/方法 | 类型 | 说明 |
|---|---|---|
| `_VALID_URL` | `str` / `Sequence[str]` / `False` | 匹配 URL 的正则表达式（或正则列表）。设为 `False` 表示仅嵌入提取器 |
| `_match_valid_url(cls, url)` | classmethod | 编译并匹配 URL，返回 `re.Match` 对象或 `None`。结果缓存在 `_VALID_URL_RE` 类变量中 |
| `suitable(cls, url)` | classmethod | 基于 `_match_valid_url` 的布尔封装，返回是否适合处理该 URL |
| `_match_id(cls, url)` | classmethod | 从 URL 中提取 `id` 命名组 |
| `get_temp_id(cls, url)` | classmethod | 容错版的 `_match_id`，失败返回 `None` |

**代码路径**：

```
_VALID_URL (声明)
    ↓
_match_valid_url()  [common.py#L615-L623]
    ├── 检查 _VALID_URL is False → 直接返回 None
    ├── 首次调用时编译正则到 _VALID_URL_RE（类级缓存）
    └── 遍历正则列表，返回第一个匹配
    ↓
suitable()  [common.py#L626-L630]
    └── 调用 _match_valid_url() is not None
```

**注意**：`_VALID_URL` 支持序列（多个正则），匹配时按顺序尝试，第一个命中即返回。`_VALID_URL_RE` 是**按类缓存**的，使用 `cls.__dict__` 而不是 `getattr`，避免继承干扰。

### 2.2 身份标识约定

| 属性 | 类型 | 说明 |
|---|---|---|
| `IE_NAME` | classproperty | 提取器名称，默认取类名去掉末尾 `IE`，如 `YoutubeIE` → `youtube` |
| `ie_key()` | classmethod | 同上，用于 `get_info_extractor()` 查找 |
| `IE_DESC` | `str` / `False` | 提取器描述。设为 `False` 时不显示在支持列表中 |
| `SEARCH_KEY` | `str` | 搜索支持的关键字（如 `ytsearch`） |

### 2.3 测试用例约定

| 属性 | 说明 |
|---|---|
| `_TEST` | 单个测试用例（dict），与 `_TESTS` 互斥 |
| `_TESTS` | 多个测试用例列表（list of dict） |
| `_WEBPAGE_TESTS` | 网页级测试用例 |
| `get_testcases()` | classmethod，读取当前类（不查父类）的测试用例 |
| `get_webpage_testcases()` | classmethod，网页测试用例 |

**测试用例结构**（以 [zetland.py](file:///d:/fz/0601-2/solo-dogfeeding/code/82-yt-dlp/yt_dlp/extractor/zetland.py#L8-L28) 为例）：

```python
_TESTS = [{
    'url': 'https://...',          # 测试 URL
    'info_dict': {                  # 期望的信息字典
        'id': '...',
        'title': '...',
        # 支持 md5 前缀、re: 正则前缀等特殊断言
        'thumbnail': r're:https://...',
        'description': 'md5:9619d426772c133f5abb26db27f26a01',
    },
    'only_matching': False,         # 仅验证 URL 匹配，不实际提取
    'skip': 'reason',               # 跳过测试的原因
    'playlist': [...],              # 播放列表测试（对应 _type=playlist）
}]
```

**派生用法**：测试用例还用于推导 `age_limit` 和 `_RETURN_TYPE`（video/playlist/any），见 `age_limit` 和 `_RETURN_TYPE` 两个缓存 classproperty。

### 2.4 登录约定

| 属性/方法 | 说明 |
|---|---|
| `_NETRC_MACHINE` | netrc 机器名，设置后表示支持账号密码登录 |
| `supports_login()` | classmethod，判断是否支持登录（基于 `_NETRC_MACHINE`） |
| `_perform_login(username, password)` | 子类重写，执行实际登录逻辑 |
| `_initialize_pre_login()` | 登录前的初始化钩子 |
| `_get_login_info()` | 获取登录信息（优先命令行参数，其次 netrc） |

**登录流程**（在 `initialize()` 中）：

```
_initialize_pre_login()
    ↓
如果 supports_login() 且子类重写了 _perform_login
    ↓
_get_login_info()  →  (username, password)
    ↓
_perform_login(username, password)
    ↓
_real_initialize()
```

### 2.5 状态与特性标记

| 属性 | 默认值 | 说明 |
|---|---|---|
| `_WORKING` | `True` | 提取器是否工作正常。设为 `False` 时警告用户并跳过测试 |
| `_ENABLED` | `True` | 是否默认启用。禁用后需显式开启 |
| `_GEO_BYPASS` | `True` | 是否允许地理绕过（X-Forwarded-For 伪造 IP） |
| `_GEO_COUNTRIES` | `None` | 推测不受限的国家列表，用于 geo bypass |
| `_GEO_IP_BLOCKS` | `None` | 不受限的 IP 段列表（CIDR） |
| `_EMBED_REGEX` | `[]` | 嵌入正则列表，用于 `GenericIE` 在网页中发现嵌入视频 |

### 2.6 返回类型约定

提取结果通过 `_type` 字段标识类型，默认是 `"video"`。

| `_type` 值 | 含义 | 必含字段 |
|---|---|---|
| `video`（默认） | 单个视频 | `id`, `title`, `formats` 或 `url` |
| `playlist` | 视频列表 | `entries`（可迭代） |
| `multi_video` | 多视频（同一页面的多个视频） | `entries` |
| `url` | 转发到另一个提取器处理 | `url`, `ie_key`（可选） |
| `url_transparent` | 透明转发（保留当前提取器的元数据） | `url` |

辅助构造方法：
- `url_result(url, ...)` — 构造 `url` / `url_transparent` 结果
- `playlist_result(entries, ...)` — 构造 `playlist` / `multi_video` 结果
- `playlist_from_matches(matches, ...)` — 从匹配结果批量构造播放列表

---

## 三、下载辅助能力（实例方法层）

这些是**实例级**的辅助方法，供子类在 `_real_extract` 中调用。它们封装了网络请求、重试、错误处理、编码检测等通用逻辑。

### 3.1 核心下载链

```
_request_webpage()       # 最底层：发请求，返回 response handle
    ↓  (包装错误处理、日志、sleep、geo bypass)
_download_webpage_handle()  # 下载并解码网页，返回 (content, urlh)
    ↓  (编码检测、内容校验)
_download_webpage()     # 只返回内容字符串
    ↓  (工厂方法批量生成同类方法)
_download_json()        # 下载并解析 JSON
_download_xml()         # 下载并解析 XML
_download_socket_json() # Socket JSON 轮询
```

#### `_request_webpage()` — 请求层

[common.py#L863-L926](file:///d:/fz/0601-2/solo-dogfeeding/code/82-yt-dlp/yt_dlp/extractor/common.py#L863-L926)

核心职责：
- 请求前 sleep（`sleep_interval_requests`）
- 注入 `X-Forwarded-For`（geo bypass）
- 处理 impersonation（浏览器模拟）
- 统一的错误处理：`HTTPError` / `network_exceptions`
- `fatal` 参数控制是否抛出 `ExtractorError`
- `expected_status` 接受非 2xx 状态码

#### `_download_webpage_handle()` — 网页下载层

[common.py#L928-L985](file:///d:/fz/0601-2/solo-dogfeeding/code/82-yt-dlp/yt_dlp/extractor/common.py#L928-L985)

在 `_request_webpage` 基础上增加：
- 去除 URL hash 片段
- 调用 `_webpage_read_content()` 解码内容
- 编码自动检测（`_guess_encoding_from_content`）：Content-Type → meta charset → BOM → UTF-8 默认
- Websense 拦截检测

#### 工厂方法生成的下载器

[common.py#L1096-L1173](file:///d:/fz/0601-2/solo-dogfeeding/code/82-yt-dlp/yt_dlp/extractor/common.py#L1096-L1173)

`__create_download_methods(name, parser, note, errnote, return_value)` 是一个工厂函数，批量生成：
- `_download_{name}_handle()` — 返回 `(解析结果, urlh)`
- `_download_{name}()` — 只返回解析结果

已生成的方法：

| 方法名 | 解析器 | 用途 |
|---|---|---|
| `_download_json` | `_parse_json` | 下载 JSON API 响应 |
| `_download_xml` | `_parse_xml` | 下载 XML / RSS |
| `_download_socket_json` | `_parse_socket_response_as_json` | WebSocket / 长轮询 |

每个方法都支持：`transform_source`（解析前转换）、`fatal`、`encoding`、`headers`、`query`、`data`（POST）、`impersonate` 等参数。

### 3.2 内容搜索辅助

这些方法在已下载的文本中提取结构化信息。

#### 正则搜索家族

| 方法 | 用途 |
|---|---|
| `_search_regex(pattern, string, name, ...)` | 通用正则搜索，支持单个/多个 pattern，自动处理错误 |
| `_html_search_regex(...)` | 在 HTML 中搜索，额外做 `clean_html` 去标签 + 反转义 |
| `_html_search_meta(name, html, ...)` | 搜索 HTML meta 标签的 content 属性 |
| `_dc_search_uploader(html)` | 搜索 Dublin Core creator 元数据 |

所有搜索方法的统一约定：
- `fatal=True` 时失败抛 `RegexNotFoundError` / `ExtractorError`
- `fatal=False` 时仅警告并返回 `None` / 默认值
- `name` 参数用于用户友好的错误消息

#### JSON 搜索家族

| 方法 | 用途 |
|---|---|
| `_parse_json(json_string, video_id, ...)` | 解析 JSON 字符串，使用宽松解析器 `LenientJSONDecoder` |
| `_search_json(start_pattern, string, name, video_id, ...)` | 从文本中正则定位并解析 JSON 对象 |
| `_search_json_ld(html, video_id, ...)` | 提取 JSON-LD 结构化数据（schema.org） |
| `_search_nextjs_data(webpage, video_id, ...)` | 提取 Next.js 的 `__NEXT_DATA__` |
| `_search_nextjs_v13_data(webpage, video_id, ...)` | 提取 Next.js 13+ App Router 的 flight 数据 |

### 3.3 格式提取辅助

针对常见流媒体协议的格式解析：

| 方法 | 用途 |
|---|---|
| `_extract_m3u8_formats(m3u8_url, video_id, ...)` | 解析 HLS m3u8，返回 formats 列表 |
| `_extract_m3u8_formats_and_subtitles(...)` | 同上，同时返回字幕 |
| `_parse_m3u8_formats_and_subtitles(m3u8_doc, ...)` | 从已下载的 m3u8 文本解析 |
| `_extract_mpd_formats(mpd_url, video_id, ...)` | 解析 DASH MPD 清单 |
| `_extract_mpd_formats_and_subtitles(...)` | 同上，带字幕 |
| `_extract_ism_formats(...)` | 解析 MSS Smooth Streaming |
| `_extract_f4m_formats(...)` | 解析 HDS F4M 清单 |

这些方法内部封装了：manifest 下载、解析、音视频流识别、字幕提取、DRM 检测、格式质量排序等复杂逻辑。

### 3.4 报告与错误辅助

| 方法 | 用途 |
|---|---|
| `report_extraction(id)` | 报告"正在提取信息" |
| `report_download_webpage(video_id)` | 报告"正在下载网页" |
| `report_warning(msg, ...)` | 输出警告，带 `[IE_NAME]` 前缀 |
| `to_screen(msg)` | 普通输出，带 `[IE_NAME]` 前缀 |
| `write_debug(msg)` | 调试输出 |
| `raise_no_formats(msg, ...)` | 抛出"无可用格式"错误 |
| `raise_login_required(msg, ...)` | 抛出登录要求错误，附带登录提示 |
| `raise_geo_restricted(msg, ...)` | 抛出地理限制错误 |
| `raise_unsupported_error(msg)` | 抛出不支持错误 |
| `get_param(name, default)` | 获取下载器参数 |

---

## 四、子类边界与扩展点

### 4.1 必须实现 vs 可选实现

| 方法 | 是否必须 | 说明 |
|---|---|---|
| `_real_extract(url)` | **必须** | 核心提取逻辑，返回信息字典 |
| `_real_initialize()` | 可选 | 初始化（登录后执行），如获取 token、设置 cookie |
| `_perform_login(username, password)` | 可选 | 登录逻辑，需配合 `_NETRC_MACHINE` |
| `_initialize_pre_login()` | 可选 | 登录前的初始化 |
| `suitable(url)` | 可覆盖 | 自定义 URL 匹配逻辑（需保持函数签名，且函数内部自行 import） |
| `_match_valid_url(url)` | 可覆盖 | 自定义 URL 匹配解析 |
| `_extract_from_webpage(url, webpage)` | 可覆盖 | 从网页 HTML 中提取（供 GenericIE 调用） |
| `_extract_embed_urls(url, webpage)` | 可覆盖 | 提取嵌入 URL |

### 4.2 主执行流程

`extract(url)` 是对外的主入口 [common.py#L755-L787](file:///d:/fz/0601-2/solo-dogfeeding/code/82-yt-dlp/yt_dlp/extractor/common.py#L755-L787)：

```
extract(url)
    ├── initialize()                 # 初始化（含登录）
    │     ├── _initialize_geo_bypass()
    │     ├── _initialize_pre_login()
    │     ├── _perform_login()       # 如有登录信息
    │     └── _real_initialize()
    ├── _real_extract(url)           # ← 子类实现的核心
    │     └── 调用各种 _download_* / _search_* / _extract_* 辅助方法
    ├── 后处理：注入 __x_forwarded_for_ip、过滤字幕等
    └── 异常处理 + 地理限制重试（最多 2 次）
```

### 4.3 典型继承层级

实际项目中常采用"基类 + 具体提取器"的分层模式：

```
InfoExtractor (common.py)
    ├── XxxBaseInfoExtractor        # 站点通用逻辑（token、API 封装）
    │     ├── XxxIE                 # 单视频提取
    │     ├── XxxPlaylistIE         # 播放列表提取
    │     └── XxxChannelIE          # 频道提取
    └── ...
```

示例（Vimeo 站点）：
- [VimeoBaseInfoExtractor](file:///d:/fz/0601-2\solo-dogfeeding/code/82-yt-dlp/yt_dlp/extractor/vimeo.py#L44-L150) — 封装登录、OAuth、API 调用
- 具体的 `VimeoIE`、`VimeoPlaylistIE` 等继承它，复用认证和 API 逻辑

### 4.4 结果字典的约定

`_real_extract` 返回的字典需要符合约定结构。核心字段：

**视频级字段**（`_type: video`）：
- `id` — 视频 ID（必需）
- `title` — 标题（必需，无标题设为空字符串，获取失败为 None）
- `formats` — 格式列表（或 `url` 字段）
- `uploader` / `uploader_id` / `uploader_url` — 上传者信息
- `description` — 描述
- `thumbnail` — 缩略图
- `duration` — 时长
- `timestamp` / `upload_date` — 上传时间
- `view_count` / `like_count` / `comment_count` — 统计数据
- `age_limit` — 年龄限制
- `subtitles` — 字幕字典
- `chapters` — 章节

**格式级字段**（`formats` 列表内每项）：
- `url` — 媒体 URL（必需）
- `format_id` — 格式标识符
- `ext` — 扩展名
- `width` / `height` / `resolution` — 分辨率
- `vcodec` / `acodec` — 音视频编码
- `vbr` / `abr` / `tbr` — 码率
- `filesize` / `filesize_approx` — 文件大小
- `protocol` — 协议（http, https, m3u8_native, http_dash_segments 等）
- `manifest_url` — 清单 URL（HLS/DASH 等）

### 4.5 嵌入提取边界

`_EMBED_REGEX` 和 `_extract_from_webpage` 构成嵌入提取的扩展点：

1. `GenericIE` 下载一个普通网页
2. 遍历所有提取器的 `_EMBED_REGEX` 在网页中搜索嵌入 URL
3. 对匹配到的 URL 再调用对应提取器处理
4. 如果提取器重写了 `_extract_from_webpage`，可以直接从网页提取信息
5. 可抛出 `self.StopExtraction` 来独占该网页（如 invidious/peertube 实例）

---

## 五、快速对照表

### 类属性速查

| 分类 | 属性 | 作用 |
|---|---|---|
| 匹配 | `_VALID_URL` | URL 正则 |
| 匹配 | `_EMBED_REGEX` | 嵌入正则 |
| 身份 | `IE_NAME` / `IE_DESC` | 名称与描述 |
| 测试 | `_TESTS` / `_TEST` | 测试用例 |
| 登录 | `_NETRC_MACHINE` | netrc 机器名 |
| 状态 | `_WORKING` / `_ENABLED` | 工作状态 |
| 地理 | `_GEO_BYPASS` / `_GEO_COUNTRIES` | 地理绕过 |
| 搜索 | `SEARCH_KEY` | 搜索关键字 |

### 常用方法速查

| 分类 | 方法 | 用途 |
|---|---|---|
| 下载 | `_download_webpage` | 下载网页 HTML |
| 下载 | `_download_json` | 下载并解析 JSON |
| 下载 | `_download_xml` | 下载并解析 XML |
| 搜索 | `_search_regex` | 正则提取 |
| 搜索 | `_search_json` | 正则定位 + JSON 解析 |
| 搜索 | `_search_json_ld` | JSON-LD 结构化数据 |
| 搜索 | `_search_nextjs_data` | Next.js 数据 |
| 搜索 | `_html_search_meta` | HTML meta 标签 |
| 格式 | `_extract_m3u8_formats` | HLS 格式解析 |
| 格式 | `_extract_mpd_formats` | DASH 格式解析 |
| 构造 | `url_result` | 构造 URL 转发结果 |
| 构造 | `playlist_result` | 构造播放列表结果 |
| 子类 | `_real_extract` | 子类核心实现 |
