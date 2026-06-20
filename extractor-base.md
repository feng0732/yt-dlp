# 站点提取器公共抽象（InfoExtractor）

> 路径约定：本文档中所有代码引用以仓库根为基准，格式为 `路径`（如 `yt_dlp/extractor/common.py`）。行号标注在段落中直接说明。

## 一、类定位与整体架构

`InfoExtractor` 是所有站点提取器的基类，定义在 `yt_dlp/extractor/common.py` 第 105 行起。它的职责分为三层：

1. **声明层（类属性）**：描述"这个提取器是什么"——URL 规则、测试用例、登录方式、地理特性、工作状态。框架通过读取这些属性来注册、匹配、测试提取器。
2. **工具层（实例方法）**：提供"下载+解析"的复用能力——网页/JSON/XML 下载、正则提取、m3u8/MPD 解析、错误报告。
3. **扩展层（钩子方法）**：定义"子类必须/可以做什么"——`_real_extract`、`_real_initialize`、`_perform_login`、`_extract_from_webpage` 等。

---

## 二、元数据约定（声明层）

### 2.1 URL 匹配链

```
_VALID_URL 声明（正则 str 或 [str]）
    ↓ 编译与缓存（按类隔离）
_match_valid_url(cls, url)  → re.Match | None
    ↓ 布尔封装
suitable(cls, url)          → bool
    ↓ 提取命名组
_match_id(cls, url)         → str (id)
```

**关键实现细节**（`common.py` 第 615–630 行）：

- `_VALID_URL_RE` 使用 `cls.__dict__` 检查而非 `getattr`，目的是**避免继承缓存干扰**——每个子类必须独立编译自己的正则。
- `_VALID_URL = False` 表示"仅嵌入提取器"：不直接匹配 URL，只在 `GenericIE` 扫描网页时通过嵌入正则工作。
- `variadic()` 工具把单个正则和正则列表归一化为 tuple，使 `_VALID_URL` 同时支持单值和多值。

### 2.2 测试用例约定

| 属性 | 作用域 | 说明 |
|---|---|---|
| `_TEST` / `_TESTS` | 当前类（不查父类） | 单/多个测试用例 |
| `_WEBPAGE_TESTS` | 当前类 | 网页级测试 |

`get_testcases()`（`common.py` 第 3805–3819 行）的实现细节：

- 用 `vars(cls)` 而非 `getattr` 读取 `_TESTS`，**不向父类查找**。这意味着：子类不会自动继承父类的测试用例，每个具体 IE 必须自己声明测试。
- `only_matching: True` 标记的用例只验证 URL 匹配，不执行实际提取。
- `skip` 字段用于跳过不稳定/失效的测试。
- `info_dict` 支持特殊前缀：`re:` 正则匹配、`md5:` 哈希匹配、`startswith:` 前缀匹配。

测试用例还有两个**派生用途**：
- `age_limit` classproperty：从所有测试的 `info_dict.age_limit` 取最大值，自动推断年龄限制。
- `_RETURN_TYPE` classproperty：根据测试是否含 `playlist*` 字段，自动判断提取器返回 video / playlist / any。

### 2.3 登录约定

```
_NETRC_MACHINE（类属性）
    ↓ 决定
supports_login() → bool
    ↓ 控制 initialize() 是否走登录分支
_initialize_pre_login() → _get_login_info() → _perform_login(u, p) → _real_initialize()
```

分层取舍：
- **类属性声明**（`_NETRC_MACHINE`）和**方法实现**（`_perform_login`）分离。这样框架可以在不实例化的情况下判断"是否支持登录"（用于 CLI 提示、文档生成）。
- `_get_login_info()` 优先读命令行 `--username/--password`，其次读 netrc。这是**配置优先级**的边界：显式参数 > 配置文件。
- `_initialize_pre_login` / `_real_initialize` 把登录前和登录后的初始化分开，支持子类在登录前准备 cookie、登录后获取 token。

### 2.4 状态与特性标记

| 属性 | 默认 | 分层含义 |
|---|---|---|
| `_WORKING` | `True` | 提取器是否工作。用于测试跳过和用户警告。 |
| `_ENABLED` | `True` | 是否默认启用。用于实验性/破损提取器。 |
| `_GEO_BYPASS` | `True` | 是否允许框架自动做 geo bypass。某些站点检测 X-Forwarded-For 反而会失败。 |
| `_GEO_COUNTRIES` / `_GEO_IP_BLOCKS` | `None` | 站点已知的不受限地区，供 bypass 时优先尝试。 |
| `_EMBED_REGEX` | `[]` | 嵌入正则列表，供 `GenericIE` 在 HTML 中发现嵌入视频。 |

### 2.5 返回类型约定

通过 `_type` 字段标识提取结果类型：

| `_type` | 含义 | 构造辅助 |
|---|---|---|
| `video`（默认） | 单视频 | 直接返回 dict |
| `playlist` | 视频列表 | `playlist_result()` |
| `multi_video` | 同一页面多个视频 | `playlist_result(multi_video=True)` |
| `url` | 转发到其他 IE | `url_result(url, ie_key)` |
| `url_transparent` | 透明转发（保留当前元数据） | `url_result(url, url_transparent=True)` |

---

## 三、下载辅助能力（工具层）—— 重点讲透工厂模式分层

### 3.1 下载方法工厂：`__create_download_methods` 的分层设计

`yt_dlp/extractor/common.py` 第 1096–1173 行定义了一个工厂函数，批量生成下载方法。这是整个工具层最精巧的抽象之一。

#### 工厂输入输出

```python
__create_download_methods(name, parser, note, errnote, return_value)
    → (_download_{name}_handle, _download_{name})
```

已生成的方法：

| 调用 | 生成方法 | 解析器 |
|---|---|---|
| `__create_download_methods('json', '_parse_json', ...)` | `_download_json_handle` / `_download_json` | `_parse_json` |
| `__create_download_methods('xml', '_parse_xml', ...)` | `_download_xml_handle` / `_download_xml` | `_parse_xml` |
| `__create_download_methods('socket_json', '_parse_socket_response_as_json', ...)` | `_download_socket_json_handle` / `_download_socket_json` | `_parse_socket_response_as_json` |
| `__create_download_methods('webpage', None, ...)` | `__download_webpage`（私有） | 无解析器 |

#### 工厂内部的三层结构

工厂为每种资源类型生成**两个方法**，形成三层调用链：

```
download_content (_download_json 等)
    ├── 功能 A: load_pages 磁盘缓存读取（调试用）
    ├── 功能 B: 通过 getattr(self, '_download_xxx_handle') 调用
    │         ↑ 注意：用 getattr 按名查找，支持子类覆盖 handle 方法
    └── parse() → 返回解析结果

download_handle (_download_json_handle 等)
    ├── 调用 _download_webpage_handle() → (content, urlh)
    └── parse() → 返回 (解析结果, urlh)

parse (内部闭包)
    └── getattr(ie, parser)(content, ...)
              ↑ 注意：用 getattr 按名查找解析器，支持子类覆盖解析方法
```

#### 分层取舍分析

**取舍 1：`getattr(self, method_name)` 而非直接引用函数**

```python
# 工厂内 parse 闭包
return getattr(ie, parser)(content, *args, **kwargs)  # 第 1104 行

# 工厂内 download_content
res = getattr(self, download_handle.__name__)(url_or_request, video_id, **kwargs)  # 第 1151 行
```

这两处都用字符串名查找，而不是直接闭包引用。**设计目的是支持子类覆盖**：子类可以重写 `_parse_json` 或 `_download_json_handle`，工厂生成的方法会自动使用子类版本。如果直接闭包引用，就会锁死在基类实现上。

**取舍 2：handle 版本 + content 版本的分离**

- `_download_xxx_handle()` 返回 `(parsed, urlh)`：需要访问响应头、cookies 时使用。
- `_download_xxx()` 只返回 `parsed`：90% 的场景只需要内容。

分离的代价是多维护一套方法，但通过工厂生成消除了重复代码。

**取舍 3：`load_pages` 调试旁路放在 content 层**

`download_content`（第 1121–1134 行）在走网络前先检查 `load_pages` 参数，如果命中就从本地磁盘读取之前 dump 的请求。这个功能放在 content 层（而非 handle 层）的原因是：磁盘加载不需要 urlh，跳过了整个网络栈，属于"替代下载"的最高层决策。

**取舍 4：`_download_webpage` 的特殊处理**

工厂生成了私有方法 `__download_webpage`，但基类又额外定义了公有的 `_download_webpage`（第 1175–1205 行），在其上增加了 `IncompleteRead` 重试循环。这是因为网页下载最容易出现不完整读取，而 JSON/XML 下载相对可靠——**给最常用的场景单独加功能，而不污染通用工厂**。

### 3.2 核心下载链的层次

```
_request_webpage()           第 863 行 — 请求层：HTTP、sleep、geo header、impersonate
    ↓
_download_webpage_handle()   第 928 行 — 协议层：URL 处理、编码检测、内容读取
    ↓
_download_webpage()          第 1175 行 — 内容层：IncompleteRead 重试
    ↓
[工厂生成的 _download_json / _download_xml 等]
```

各层职责：

| 层 | 方法 | 职责边界 |
|---|---|---|
| 请求层 | `_request_webpage` | 只管发请求、拿 response handle。处理网络异常、请求间隔、X-Forwarded-For 注入、浏览器模拟。**不解析内容**。 |
| 协议层 | `_download_webpage_handle` | 把 response handle 解码为字符串。处理编码自动检测（Content-Type → HTML meta → BOM → UTF-8）、Websense 拦截检测。返回 `(content, urlh)`。 |
| 内容层 | `_download_webpage` 及工厂方法 | 处理内容级重试（IncompleteRead）、磁盘缓存、格式解析（JSON/XML）。 |

### 3.3 内容搜索辅助

正则搜索家族（`_search_regex`、`_html_search_regex`、`_html_search_meta`）统一约定：
- `fatal=True` 失败抛异常，`fatal=False` 仅警告返回默认值
- `name` 参数用于用户友好的错误消息

JSON 搜索家族：
- `_parse_json`：宽松 JSON 解析，支持 `transform_source` 预处理
- `_search_json`：先正则定位 JSON 字符串边界，再解析
- `_search_json_ld`：schema.org JSON-LD 提取
- `_search_nextjs_data`：Next.js `__NEXT_DATA__` 提取
- `_search_nextjs_v13_data`：Next.js 13+ App Router Flight 数据提取

### 3.4 格式提取辅助

针对流媒体协议的高层封装：`_extract_m3u8_formats`、`_extract_mpd_formats`、`_extract_ism_formats`、`_extract_f4m_formats`。这些方法内部封装了 manifest 下载解析、音视频流识别、字幕提取、DRM 检测、质量排序。

---

## 四、地理重试机制 —— 重点讲透分层取舍

地理限制绕过分布在**三个层次**，各有不同的触发时机和职责边界。

### 4.1 三层架构

```
初始化层: _initialize_geo_bypass()       第 672 行
    ├── 启动时就尝试 fake IP
    └── 数据源: 类属性 _GEO_COUNTRIES / _GEO_IP_BLOCKS

提取层:   extract() 的 retry 循环         第 755–776 行
    ├── for _ in range(2): 最多重试 1 次
    └── 捕获 GeoRestrictedError → __maybe_fake_ip_and_retry()

请求层:   _request_webpage() 注入 header  第 891–893 行
    └── 每次请求自动附加 X-Forwarded-For
```

### 4.2 初始化层：`_initialize_geo_bypass`

`common.py` 第 672–753 行。启动时主动尝试 fake IP。

**决策优先级**（从高到低）：
1. 用户显式指定 `--geo-bypass-ip-block` → 直接用
2. 用户显式指定 `--geo-bypass-country` → 直接用
3. 如果 `_GEO_BYPASS=True` 且有 `_GEO_IP_BLOCKS` → 随机选 IP 段
4. 如果 `_GEO_BYPASS=True` 且有 `_GEO_COUNTRIES` → 随机选国家

**取舍：为什么分 IP 段和国家两条路径？**

- IP 段（CIDR）精度高，直接生成该网段内的随机 IP，不需要额外的地理位置数据库。
- 国家代码更直观，但依赖 `GeoUtils.random_ipv4(country)` 内部的地理位置→IP 映射。
- 优先 IP 段是因为它不依赖外部数据库，可靠性更高。

**取舍：为什么 `extract()` 也需要重试，而不只是初始化时做？**

因为很多站点的地理限制不是在首页就暴露的，而是在**请求具体视频 API 时**才返回限制错误。`initialize()` 阶段可能只做了登录，还没请求受限资源。这就是两层设计的根本原因：

- 初始化层 = **预防性**绕过（知道这个站点整体受限，一开始就 fake IP）
- 提取重试层 = **反应性**绕过（实际碰到了限制再 fake IP 重试）

### 4.3 提取重试层：`__maybe_fake_ip_and_retry`

`common.py` 第 789–802 行。触发条件非常严格：

```python
if (not self.get_param('geo_bypass_country', None)   # 用户没手动指定国家
    and self._GEO_BYPASS                              # 提取器允许 bypass
    and self.get_param('geo_bypass', True)            # 用户没全局关闭 bypass
    and not self._x_forwarded_for_ip                  # 当前还没有 fake IP（避免循环）
    and countries):                                   # 错误中携带了可选国家
```

**取舍：为什么 `extract()` 最多只重试 2 次？**

- 第一次：正常提取
- 第二次：fake IP 后重试
- 更多次重试没有意义，因为 X-Forwarded-For 绕过成功率低，失败通常意味着该站点需要更复杂的 bypass（如 VPN），框架已经无能为力。

### 4.4 请求层：`_request_webpage` 自动注入

`common.py` 第 891–893 行：每个请求自动附加 `X-Forwarded-For` header。这是最底层的透明层，子类调用 `_download_webpage` / `_download_json` 时完全无感。

**取舍：为什么用 X-Forwarded-For 而不是其他方式？**

- 零依赖：不需要代理服务。
- 大量 CDN/站点信任这个 header（用于统计、日志），对只做简单 geo 检查的站点足够有效。
- 但不可靠：很多站点检查真实源 IP。所以 geo bypass 明确标注为"尽力而为"。

---

## 五、嵌入提取边界 —— 重点讲透执行顺序与独占异常

嵌入提取解决的问题：一个普通网页（如博客文章页）里嵌入了 YouTube/Vimeo 等视频，用户给出这个网页 URL，希望能下载里面的视频。

### 5.1 GenericIE 主流程：从 URL 到嵌入提取的完整路径

`yt_dlp/extractor/generic.py` 第 763 行起，`_real_extract` 按以下**严格顺序**执行，每一步如果成功就直接 `return`，不会走到后面的步骤：

```
_real_extract(url)
    ├── 阶段 1: URL 协议处理（第 763-793 行）
    │     ├── URL 无 scheme → 尝试 https:// 或 fallback 到 youtube 搜索
    │     └── 返回 url_result，让对应 IE 处理
    │
    ├── 阶段 2: 请求网页（第 818-850 行）
    │     ├── _request_webpage 下载响应头和前 512 字节
    │     ├── 3xx 重定向 → 返回 url_result 跟随重定向
    │     └── Cloudflare 403 特殊处理 → 提示 impersonation
    │
    ├── 阶段 3: 直链检测（第 858-914 行）—— 最优先的快速路径
    │     ├── Content-Type 是 audio/video/mpegurl → 直接构造 format（第 860-886 行）
    │     ├── 前 512 字节 #EXTM3U → 解析 m3u8（第 894-899 行）
    │     └── 前 512 字节不是 HTML → 直接返回为直链（第 903-914 行）
    │
    ├── 阶段 4: XML 格式检测（第 924-963 行）
    │     ├── 尝试 XML 解析
    │     ├── RSS → _extract_rss（第 930-932 行）
    │     ├── SMIL → _parse_smil（第 937-940 行）
    │     ├── XSPF → _parse_xspf（第 941-947 行）
    │     ├── MPD → _parse_mpd_formats_and_subtitles（第 948-957 行）
    │     ├── SmoothStreamingMedia → _parse_ism_formats（第 933-936 行）
    │     └── F4M → _parse_f4m_formats（第 958-961 行）
    │
    ├── 阶段 5: 提取基本元数据（第 965-976 行）
    │     ├── title / description / thumbnail / age_limit
    │     └── 作为兜底信息，后续嵌入结果会 merge 这些字段
    │
    └── 阶段 6: 调用 _extract_embeds（第 978-984 行）
          └── embeds = list(self._extract_embeds(...))
              ├── 1 个嵌入 → merge_dicts(embeds[0], info_dict)
              ├── 多个嵌入 → playlist_result(embeds, **info_dict)
              └── 0 个嵌入 → raise UnsupportedError(url)
```

> **关键观察**：直链检测在阶段 3 就完成了，远早于嵌入提取。这意味着如果 URL 直接指向视频文件，不会进入嵌入扫描流程。

### 5.2 `_extract_embeds` 内部顺序：嵌入遍历 vs 播放器兜底

`yt_dlp/extractor/generic.py` 第 986 行起，`_extract_embeds` 内部按以下顺序执行：

```
_extract_embeds(url, webpage)
    │
    ├── 第一部分: 网页嵌入入口遍历（第 1000-1018 行）
    │     │
    │     ├── 遍历顺序来源: self._downloader._ies.values()
    │     │     └── _ies 的注册顺序由 extractors.py 第 25-30 行精心设计:
    │     │           1. Youtube 相关 IE 优先（提高匹配性能）
    │     │           2. 其他所有 IE（按类名排序后的导入表顺序，约 1500+ 个）
    │     │           3. GenericIE 最后（但 GenericIE 自己不参与嵌入扫描）
    │     │
    │     ├── 对每个 IE:
    │     │     ├── 跳过 block_ies 中的 IE（防止递归）
    │     │     ├── gen = ie.extract_from_webpage(ydl, url, webpage)
    │     │     ├── 手动迭代生成器（next(gen)）
    │     │     ├── 捕获 StopExtraction → 独占，立即 return
    │     │     └── 捕获 StopIteration → 收集当前 IE 的嵌入
    │     │
    │     └── if embeds: return embeds  ← 只要有嵌入结果，就不进播放器兜底
    │
    └── 第二部分: 播放器模式兜底（第 1020-1223 行）
          │
          ├── 兜底触发条件: 第一部分遍历完所有 IE，没有任何嵌入结果
          │
          ├── 兜底执行顺序（一旦找到就 return，不继续后面的）:
          │     1.  JW Player 数据（第 1020-1034 行）
          │     2.  Video.js 嵌入（第 1036-1097 行）
          │     3.  KVS Player（第 1099-1108 行）
          │     4.  JSON-LD VideoObject（第 1110-1122 行）
          │     5.  SWFObject 内嵌的 JW Player（第 1136-1139 行）
          │     6.  JW Player 嵌入（第 1142-1151 行）
          │     7.  通用 file/source 正则（第 1152-1156 行）
          │     8.  JW Player JS loader（第 1157-1162 行）
          │     9.  Flow Player（第 1164-1172 行）
          │     10. Cinerama player（第 1173-1178 行）
          │     11. Twitter card stream（第 1179-1187 行）
          │     12. Open Graph video（第 1188-1196 行）
          │     13. Meta refresh 重定向（第 1197-1214 行）
          │     14. Twitter player iframe（第 1216-1223 行）
          │
          └── 都没找到 → return []（最终会 raise UnsupportedError）
```

**分层取舍分析**：

> **为什么先遍历所有 IE 的嵌入，再走播放器兜底？**
>
> 1. **优先级**：特定站点的提取器（如 YouTubeIE、VimeoIE）对自己的嵌入格式有最准确的解析能力，应该优先尝试。
> 2. **正确性**：JW Player 等通用播放器模式是"瞎猜"，可能把非视频资源识别为视频，应该作为最后手段。
> 3. **性能**：遍历 1500+ 个 IE 每个只跑正则扫描，很快；而播放器兜底需要做多次正则搜索、JSON 解析，相对慢。
>
> **为什么第一部分找到任何嵌入就立即 return，不继续找更多？**
>
> 这是一个保守设计：如果某个专业 IE 已经识别出嵌入，就相信它的结果，不再让通用兜底模式产生多余结果。但如果多个 IE 都返回了结果（没有独占），会全部收集起来作为播放列表。

### 5.3 StopExtraction 独占异常的捕获机制

#### 5.3.1 异常抛出点

`InfoExtractor.StopExtraction` 定义在 `common.py` 第 4113-4114 行：
```python
class StopExtraction(Exception):
    pass
```

子类在重写 `_extract_from_webpage` 时，检测到网页特征后抛出：
```python
@classmethod
def _extract_from_webpage(cls, url, webpage):
    if 'invidious-player' in webpage:
        info_dict = {...}  # 直接从网页提取信息
        yield info_dict
        raise cls.StopExtraction  # 告诉 GenericIE 别再找其他 IE 了
```

#### 5.3.2 完整传播路径

异常从抛出到捕获要经过三层调用：

```
[子类 _extract_from_webpage] 抛出 StopExtraction
    ↓ 透传（for 循环不捕获非 StopIteration 异常）
[InfoExtractor.extract_from_webpage] 第 4086 行
    │   for info in ie._extract_from_webpage(url, webpage) or []:
    │       yield info
    │
    ↓ 透传（普通 for/yield 只会把 StopIteration 当作迭代结束，其他异常继续向外冒泡）
[GenericIE._extract_embeds] 第 1008 行
    │   gen = ie.extract_from_webpage(self._downloader, url, webpage)
    │   while True:
    │       current_embeds.append(next(gen))  ← 这里抛异常
    │
    ↓ 被捕获
[GenericIE._extract_embeds] 第 1009 行
        except self.StopExtraction:
            return current_embeds  # 立即返回，后续 IE 不再遍历
```

#### 5.3.3 关键实现细节

**细节 1：手动迭代生成器，而非 for 循环**

`generic.py` 第 1006-1015 行：
```python
gen = ie.extract_from_webpage(self._downloader, url, webpage)
current_embeds = []
try:
    while True:
        current_embeds.append(next(gen))  # 手动 next() 调用
except self.StopExtraction:
    return current_embeds  # 独占
except StopIteration:
    embeds.extend(current_embeds)  # 正常结束
```

**为什么不用 `for x in gen`？**
```python
# 如果这样写：
for x in gen:
    current_embeds.append(x)
```
`for` 循环会自动捕获 `StopIteration` 并静默结束循环。但 `StopExtraction` 是另一个异常，会正常冒泡。问题在于：捕获异常时，我们需要**区分三种情况**：
1. 生成器正常结束（`StopIteration`）→ 收集结果，继续下一个 IE
2. 生成器请求独占（`StopExtraction`）→ 立即返回，终止整个遍历
3. 其他异常（`ExtractorError` 等）→ 向上抛出，让上层处理

手动 `next()` 把"正常结束"和"请求独占"都放在同一个局部 `try` 块里处理：正常结束时把 `current_embeds` 合并到总结果，独占时只返回 `current_embeds` 并丢弃之前结果。这样不用额外的循环后状态标记，就能精确控制两个出口。

**细节 2：StopExtraction 继承 Exception，而非 StopIteration**

`common.py` 第 4113 行：
```python
class StopExtraction(Exception):  # 不是 StopIteration
    pass
```

**为什么不继承 StopIteration？**

- `StopIteration` 在生成器中有特殊语义，表示"生成器结束"。如果 `StopExtraction` 继承它，会被 Python 的生成器机制和 `for` 循环特殊处理，可能导致意外的静默结束。
- 继承普通 `Exception` 确保它会正常冒泡，只有我们显式的 `except` 块才能捕获它。
- 这是一个**控制流异常**（control flow exception），用异常实现非局部跳转，类似 `return` 但可以跨多层调用栈。

**细节 3：current_embeds 保留了独占 IE 已经 yield 的结果**

```python
except self.StopExtraction:
    # current_embeds 中已经包含了该 IE yield 的所有结果
    return current_embeds
```

这意味着：独占 IE 可以先 `yield` 一些结果，再 `raise StopExtraction`。`GenericIE` 会把已经 yield 的结果作为最终结果返回，**同时丢弃**之前其他 IE 已经收集的 `embeds`（因为 `return current_embeds` 而不是 `return embeds + current_embeds`）。

代码第 1010 行明确输出了这个行为：
```python
self.report_detected(f'{ie.IE_NAME} exclusive embed', len(current_embeds),
                     embeds and 'discarding other embeds')
```
第三个参数 `embeds and 'discarding other embeds'` 明确告诉用户：之前收集的其他嵌入被丢弃了。

**取舍：为什么独占要丢弃之前的结果？**

- 独占机制的设计目的是"这个网页属于我"，意味着它的解析是最权威的。
- 如果之前的 IE 已经识别了一些嵌入，那些很可能是误报（比如 invidious 页面里也有 YouTube 嵌入 URL，但实际上应该用 invidious 自己的提取器）。
- 但这是一个权衡：可能会丢失一些真正的多嵌入场景。所以独占机制应该谨慎使用，只在"这个网页是我的实例"时才触发。

### 5.4 四层架构（回顾与补充）

```
用户 URL → GenericIE._real_extract()
    ↓
    第 1 层: GenericIE 直接解析（直链、XML manifest 等）
    ↓ 如果未解析成功
    第 2 层: GenericIE._extract_embeds() 遍历所有 IE
        ↓ 按 Youtube 优先 → 其他 IE → 播放器兜底的顺序
        第 3 层: 每个 IE.extract_from_webpage() → _extract_from_webpage()
            ↓
            第 4 层: _extract_embed_urls() 基于 _EMBED_REGEX 扫描
            └── 或者: 子类重写 _extract_from_webpage() 做深度提取 + 独占
```

### 5.5 边界总结

| 组件 | 位置 | 层级 | 可被覆盖？ |
|---|---|---|---|
| `_EMBED_REGEX` | 类属性 | 声明层 | 子类定义自己的正则列表 |
| `_extract_embed_urls` | classmethod | 正则提取层 | 可覆盖（复杂嵌入场景） |
| `_extract_from_webpage` | classmethod / instance method | 提取层 | 可覆盖（需要深度解析 / 独占机制） |
| `extract_from_webpage` | classmethod | 调度层 | 不建议覆盖，处理实例化和默认信息 |
| `GenericIE._extract_embeds` | instance method | 全局调度层 | 属于 GenericIE 内部逻辑 |
| `GenericIE._real_extract` | instance method | 主入口层 | 属于 GenericIE 内部逻辑 |

---

## 六、子类边界与扩展点（扩展层）

### 6.1 必须实现 vs 可选实现

| 方法 | 是否必须 | 层级 | 说明 |
|---|---|---|---|
| `_real_extract(url)` | **必须** | 核心 | 返回信息字典 |
| `_real_initialize()` | 可选 | 初始化 | 登录后执行（如获取 token） |
| `_perform_login(u, p)` | 可选 | 登录 | 需配合 `_NETRC_MACHINE` |
| `_initialize_pre_login()` | 可选 | 初始化 | 登录前执行 |
| `suitable(url)` | 可覆盖 | 匹配 | 自定义 URL 匹配（需自行 import 依赖） |
| `_extract_from_webpage(url, html)` | 可覆盖 | 嵌入 | 从 HTML 提取嵌入 |
| `_extract_embed_urls(url, html)` | 可覆盖 | 嵌入 | 自定义嵌入 URL 提取 |

### 6.2 `extract()` 主流程

`common.py` 第 755–787 行：

```
extract(url)
    for _ in range(2):              # geo 重试最多一次
        try:
            initialize()
                → _initialize_geo_bypass
                → _initialize_pre_login
                → _perform_login (如果支持)
                → _real_initialize
            ie_result = _real_extract(url)   # ← 子类核心
            # 后处理: 注入 __x_forwarded_for_ip, 过滤字幕等
            return ie_result
        except GeoRestrictedError as e:
            if __maybe_fake_ip_and_retry(e.countries):
                continue
            raise
    # 外层异常包装: ExtractorError, IncompleteRead, KeyError 等
```

### 6.3 典型继承层级

实际站点常采用"基础 IE + 具体 IE"的分层：

```
InfoExtractor
    ├── VimeoBaseInfoExtractor          # 登录、OAuth、API 封装
    │     ├── VimeoIE                   # 单视频
    │     ├── VimeoPlaylistIE           # 播放列表
    │     └── VimeoChannelIE            # 频道
    └── DailymotionBaseInfoExtractor
          ├── DailymotionIE
          └── DailymotionPlaylistIE
```

基础 IE 负责：认证（`_perform_login`）、API 调用封装（如 `_call_api`）、通用数据解析。
具体 IE 负责：`_VALID_URL` 定义、`_real_extract` 调用基础 API、组装返回字典。

### 6.4 结果字典约定

**视频级**（`_type: video`）：`id`、`title` 为必填，`formats` 或 `url` 二选一。
可选：`uploader`、`description`、`thumbnail`、`duration`、`timestamp`、`view_count`、`age_limit`、`subtitles`、`chapters` 等。

**格式级**（`formats` 列表项）：`url` 必填。
可选：`format_id`、`ext`、`width/height`、`vcodec/acodec`、`vbr/abr/tbr`、`filesize`、`protocol`、`manifest_url` 等。

---

## 七、速查表

### 类属性速查

| 分类 | 属性 | 作用 |
|---|---|---|
| URL 匹配 | `_VALID_URL` | 正则或正则列表；`False` 表示仅嵌入 |
| 嵌入 | `_EMBED_REGEX` | 嵌入 URL 正则列表（须含 `(?P<url>...)`） |
| 身份 | `IE_NAME` / `IE_DESC` | 名称与描述 |
| 测试 | `_TESTS` / `_TEST` | 测试用例 |
| 登录 | `_NETRC_MACHINE` | netrc 机器名，设置后表示支持登录 |
| 状态 | `_WORKING` / `_ENABLED` | 工作状态与是否默认启用 |
| 地理 | `_GEO_BYPASS` / `_GEO_COUNTRIES` / `_GEO_IP_BLOCKS` | 地理绕过配置 |
| 搜索 | `SEARCH_KEY` | 搜索关键字（如 `ytsearch`） |

### 常用方法速查

| 分类 | 方法 | 用途 |
|---|---|---|
| 下载 | `_download_webpage` | 下载 HTML |
| 下载 | `_download_json` / `_download_xml` | 下载并解析 JSON/XML |
| 搜索 | `_search_regex` | 正则提取 |
| 搜索 | `_search_json` / `_search_json_ld` | JSON 提取 |
| 搜索 | `_search_nextjs_data` | Next.js 数据提取 |
| 搜索 | `_html_search_meta` | HTML meta 标签提取 |
| 格式 | `_extract_m3u8_formats` / `_extract_mpd_formats` | HLS / DASH 格式解析 |
| 构造 | `url_result` / `playlist_result` | 构造转发/播放列表结果 |
| 错误 | `raise_login_required` / `raise_geo_restricted` / `raise_no_formats` | 标准化错误 |
| 子类 | `_real_extract` | 子类核心实现 |
