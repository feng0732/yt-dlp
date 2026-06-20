# yt-dlp 代理配置进入网络请求的完整路径

按代码执行顺序，从代理来源到最终发出网络请求，拆为三个阶段：代理来源、客户端封装、异常处理。

---

## 一、代理来源：从命令行 / 环境变量到 `YoutubeDL.proxies`

### 1.1 命令行入口

`yt_dlp/options.py:L608-L613` 定义了两个代理相关命令行参数：

- `--proxy URL`：设置全局代理（HTTP/HTTPS/SOCKS），空字符串表示直连（不走代理）
- `--geo-verification-proxy URL`：专门用于地理验证请求的代理

解析后存入 `opts.proxy` / `opts.geo_verification_proxy`。

### 1.2 参数传递到 YoutubeDL

`yt_dlp/__init__.py:915` 把 `opts.proxy` 映射为 `YoutubeDL.params['proxy']`，`opts.geo_verification_proxy` 映射为 `params['geo_verification_proxy']`。

### 1.3 `YoutubeDL.proxies` 计算属性（核心汇聚点）

YoutubeDL.py#L4197-L4211（`yt_dlp/YoutubeDL.py:L4197-L4211`） 用 `@functools.cached_property` 定义了 `proxies` 属性，这是代理配置的核心汇聚逻辑：

```python
@functools.cached_property
def proxies(self):
    opts_proxy = self.params.get('proxy')
    if opts_proxy is not None:
        if opts_proxy == '':
            opts_proxy = '__noproxy__'
        proxies = {'all': opts_proxy}
    else:
        proxies = urllib.request.getproxies()
        if 'http' in proxies and 'https' not in proxies:
            proxies['https'] = proxies['http']
    return proxies
```

**决策逻辑**：

| 条件 | 结果 |
|------|------|
| `--proxy` 有值 | `{'all': '用户指定的URL'}` |
| `--proxy ""` | `{'all': '__noproxy__'}`（强制直连） |
| 未指定 `--proxy` | 调用 `urllib.request.getproxies()` 读取系统环境变量（`HTTP_PROXY`、`HTTPS_PROXY` 等），若只有 `http` 没有 `https` 则自动补充 |

返回的 dict 结构为 `{协议: 代理URL}`，key 可以是 `http`、`https`、`all`、`no`。

### 1.4 地理验证代理（特殊代理源）

extractor/common.py#L3971-L3976（`yt_dlp/extractor/common.py:L3971-L3976`） 中，`geo_verification_headers()` 把 `geo_verification_proxy` 写入请求头 `Ytdl-Request-Proxy`：

```python
def geo_verification_headers(self):
    headers = {}
    geo_verification_proxy = self.get_param('geo_verification_proxy')
    if geo_verification_proxy:
        headers['Ytdl-request-proxy'] = geo_verification_proxy
    return headers
```

这个 header 会在后续 `clean_proxies` 中被提取，覆盖所有代理设置。详见下文「三、地理验证代理的完整覆盖路径」。

---

## 二、客户端封装：从 `proxies` dict 到实际网络连接

### 2.1 `build_request_director`：代理注入各 Handler

YoutubeDL.py#L4338-L4371（`yt_dlp/YoutubeDL.py:L4338-L4371`） 的 `build_request_director()` 是代理配置注入各 RequestHandler 的入口：

```python
def build_request_director(self, handlers, preferences=None):
    proxies = self.proxies.copy()
    clean_proxies(proxies, headers)   # ← 清洗代理
    for handler in handlers:
        director.add_handler(handler(
            ...,
            proxies=proxies,           # ← 传入清洗后的代理dict
            ...
        ))
```

每个 `RequestHandler` 基类构造器把 `proxies` 存入 `self.proxies`。

### 2.2 `clean_proxies`：代理清洗

utils/networking.py#L169-L198（`yt_dlp/utils/networking.py:L169-L198`） 执行三步清洗：

1. **提取 `Ytdl-Request-Proxy` header**：若存在，清空所有代理，设 `proxies['all']` = 该值（最高优先级，连 `NO_PROXY` 都忽略）
2. **处理 `__noproxy__`**：将值设为 `None`（表示明确不走代理）
3. **补全 scheme**：无 scheme 的代理 URL 默认加 `http://`
4. **兼容 scheme 替换**：`socks5` → `socks5h`（远程DNS），`socks` → `socks4`

### 2.3 `Request` 对象中的代理

networking/common.py#L399-L422（`yt_dlp/networking/common.py:L399-L422`） 的 `Request` 类可携带独立的 `proxies` dict：

```python
class Request:
    def __init__(self, url, ..., proxies=None, ...):
        self.proxies = proxies or {}
```

### 2.4 `RequestHandler._get_proxies`：请求级代理优先

networking/common.py#L287-L288（`yt_dlp/networking/common.py:L287-L288`）：

```python
def _get_proxies(self, request):
    return (request.proxies or self.proxies).copy()
```

**优先级**：`request.proxies`（请求级） > `self.proxies`（Handler 全局级）。

### 2.5 代理校验：`_check_proxies`

networking/common.py#L296-L331（`yt_dlp/networking/common.py:L296-L331`） 在 `validate()` 时校验代理：

- `no` key 需要 `Features.NO_PROXY` 支持
- `all` key 需要 `Features.ALL_PROXY` 支持
- 不属于 Handler 支持的 URL scheme 的代理 key 会被跳过（不报错）
- 代理 URL 必须有 scheme，否则抛 `UnsupportedRequest`
- 代理 URL 的 scheme 必须在 `_SUPPORTED_PROXY_SCHEMES` 中

### 2.6 `select_proxy`：统一代理选择

utils/networking.py#L246-L256（`yt_dlp/utils/networking.py:L246-L256`）：

```python
def select_proxy(url, proxies):
    url_components = urllib.parse.urlparse(url)
    if 'no' in proxies:
        hostport = url_components.hostname + ...
        if urllib.request.proxy_bypass_environment(hostport, {'no': proxies['no']}):
            return
        elif urllib.request.proxy_bypass(hostport):
            return
    return traverse_obj(proxies, url_components.scheme or 'http', 'all')
```

**选择逻辑**：
1. 先检查 `no` 排除列表（环境变量 + 系统设置）
2. 按请求 URL 的 scheme 精确匹配（如 `https`），匹配不到则 fallback 到 `all`

### 2.7 各 Handler 实际使用代理的方式

#### UrllibRH

networking/_urllib.py#L354-L444（`yt_dlp/networking/_urllib.py:L354-L444`）

- `_create_instance(proxies, cookiejar, ...)` 构建 `OpenerDirector` 时把 proxies 传入 ProxyHandler（`yt_dlp/networking/_urllib.py:L242-L260`）
- `ProxyHandler.proxy_open()` 调用 `select_proxy()` 选择代理
- **HTTP 代理**：直接用 `urllib.request.ProxyHandler.proxy_open()` 处理
- **SOCKS 代理**：把代理 URL 写入 `Ytdl-socks-proxy` header，由 HTTPHandler._make_conn_class（`yt_dlp/networking/_urllib.py:L86-L91`） 取出，通过 make_socks_conn_class（`yt_dlp/networking/_urllib.py:L181-L200`） 创建 `SocksConnection` 子类
- `SocksConnection.connect()` 调用 _helper.create_socks_proxy_socket（`yt_dlp/networking/_helper.py:L217-L232`），底层用 `yt_dlp.socks.sockssocket` 建立 SOCKS 连接

#### RequestsRH

networking/_requests.py#L248-L363（`yt_dlp/networking/_requests.py:L248-L363`）

- `session.trust_env = False`：不使用 requests 自己的环境变量代理
- `_send()` 中 `session.request(proxies=self._get_proxies(request))` 把代理 dict 传给 requests
- `requests.adapters.select_proxy` 被替换为 yt-dlp 自己的 `select_proxy`（第111行（`yt_dlp/networking/_requests.py:111`））
- **SOCKS 代理**：SocksProxyManager（`yt_dlp/networking/_requests.py:L407-L415`） 替换了 `requests.adapters.SOCKSProxyManager`，使用 yt-dlp 自己的 `sockssocket` 实现，避免额外依赖 `PySocks`
- SocksHTTPConnection._new_conn（`yt_dlp/networking/_requests.py:L377-L392`） 调用 `_helper.create_connection` + `create_socks_proxy_socket`

#### CurlCFFIRH

networking/_curlcffi.py#L199-L341（`yt_dlp/networking/_curlcffi.py:L199-L341`）

- `_send()` 中用 `select_proxy(request.url, proxies)` 选出代理
- 通过 `session.curl.setopt(CurlOpt.PROXY, proxy)` 设置到 libcurl
- `no` 代理列表通过 `CurlOpt.NOPROXY` 设置
- HTTPS URL 自动启用 `CurlOpt.HTTPPROXYTUNNEL`（HTTP CONNECT 隧道）
- 设置 `CurlOpt.PROXY_CAINFO` 和 `PROXY_SSL_VERIFYPEER/HOST` 控制代理 TLS 验证

#### WebsocketsRH

networking/_websockets.py#L91-L189（`yt_dlp/networking/_websockets.py:L91-L189`）

- 只支持 SOCKS 代理（`_SUPPORTED_PROXY_SCHEMES` 不含 `http`/`https`）
- `_send()` 中用 `select_proxy()` 选出代理，若存在则调用 `make_socks_proxy_opts()` + `create_connection()` + `create_socks_proxy_socket()` 创建 SOCKS socket
- 把 socket 传给 `websockets.sync.client.connect(sock=sock, ...)`

---

## 三、地理验证代理的完整覆盖路径（从请求头到网络客户端）

地理验证代理（`--geo-verification-proxy`）不经过全局 `YoutubeDL.proxies`，而是**通过单次请求的 HTTP header 中转**，最终在请求临发出前被提取为该请求的专属代理 dict。整条路径共 7 步。

### 3.1 Step 1：extractor 调用 `geo_verification_headers()`

在有地理限制风险的 extractor 中（目前约 40+ 个，如 bilibili、qqmusic、youku 等），需要地理验证的请求会显式合并这个 header。典型调用有两种风格：

- **作为 headers 参数的一部分**：
  - bilibili.py#L251（`yt_dlp/extractor/bilibili.py:251`）：`headers={'Referer': url, **self.geo_verification_headers()}`
  - youku.py#L152（`yt_dlp/extractor/youku.py:152`）：`headers.update(self.geo_verification_headers())`
- **直接作为 `_download_json` / `_download_webpage` 的 `headers=` 参数**：
  - qqmusic.py#L178（`yt_dlp/extractor/qqmusic.py:178`）：`headers=self.geo_verification_headers()`
  - vrt.py#L97（`yt_dlp/extractor/vrt.py:97`）：`**self.geo_verification_headers()`

只有用户设置了 `--geo-verification-proxy URL`，`geo_verification_headers()` 才返回 `{'Ytdl-request-proxy': URL}`，否则返回空 dict `{}`。

### 3.2 Step 2：extractor 下载方法 → `_create_request` → headers 写入 `Request`

extractor 的所有下载入口最终都会走到 extractor/common.py#L851-L861（`yt_dlp/extractor/common.py:L851-L861`） 的 `_create_request`：

```
extractor 调用链：
  _download_webpage(url, headers=...)
    → _download_webpage_handle(...)
      → _request_webpage(url, headers=...)
        → _create_request(url, headers=headers, ...)
          → Request(url)
          → request.update(headers=headers)
            → self.headers.update(headers)   # Ytdl-request-proxy 被写入 Request.headers
```

`_create_request` 调用 `Request.update(headers=...)`，把含 `Ytdl-request-proxy` 的 dict 合并进 `request.headers`（一个 `HTTPHeaderDict`，大小写不敏感）。此时：
- `request.proxies = {}`（空，因为构造 Request 时没有传）
- `request.headers` 中有 `Ytdl-Request-Proxy: <URL>`（HTTPHeaderDict 内部会把 key 标题化）

> 注意：**下载器（FileDownloader）层面的请求**，如 downloader/http.py#L113-L119（`yt_dlp/downloader/http.py:L113-L119`） 中的 `HttpFD.real_download` 用 `info_dict['http_headers']` 构造 `Request` 后直接 `self.ydl.urlopen(request)`，因为它们只是下载视频文件流，不走地理验证。

### 3.3 Step 3：`YoutubeDL.urlopen` 入口

所有请求最终汇入 YoutubeDL.py#L4274-L4328（`yt_dlp/YoutubeDL.py:L4274-L4328`）：

```python
def urlopen(self, req):
    if isinstance(req, str):
        req = Request(req)
    elif isinstance(req, urllib.request.Request):
        req = urllib_req_to_req(req)   # 老 API 兼容转换

    clean_proxies(proxies=req.proxies, headers=req.headers)  # ← 关键行！
    clean_headers(req.headers)
    return self._request_director.send(req)
```

关键点：`clean_proxies(proxies=req.proxies, headers=req.headers)` 这行**直接操作的是 `req.proxies`（传引用）**和 `req.headers`，就地修改。

### 3.4 Step 4：`clean_proxies` 把 header 转为 `req.proxies['all']`

utils/networking.py#L169-L198（`yt_dlp/utils/networking.py:L169-L198`） 的前几行是整个覆盖机制的核心：

```python
def clean_proxies(proxies: dict, headers: HTTPHeaderDict):
    req_proxy = headers.pop('Ytdl-Request-Proxy', None)   # 从 headers 中取出并删除
    if req_proxy:
        proxies.clear()                                     # XXX: 清空所有原代理
        proxies['all'] = req_proxy                          # 写入全局代理 key 'all'
    # ... 后续清洗逻辑 ...
```

执行后的结果：
- `req.headers` 中不再有 `Ytdl-Request-Proxy`（被 `pop` 掉）
- `req.proxies` 从 `{}` 变为 `{'all': '<geo-verification-proxy-URL>'}`

**注意这里覆盖的语义**：代码注释 `XXX: compat: Ytdl-Request-Proxy takes preference over everything, including NO_PROXY` 明确指出：只要存在这个 header，就会 `proxies.clear()`，**丢弃 NO_PROXY 和所有按协议指定的代理**，强制所有协议都走该代理。

### 3.5 Step 5：`RequestDirector.send` → 选中 Handler → `validate`

进入 `RequestDirector.send(req)`（networking/common.py#L94-L130（`yt_dlp/networking/common.py:L94-L130`）），按偏好遍历 Handler：

```python
for handler in self._get_handlers(request):
    handler.validate(request)   # ← 代理 schema 校验在这里
    response = handler.send(request)
```

在 `handler.validate()` 中会调用 `_validate(request)` → `_check_proxies(request.proxies or self.proxies)`：

```python
# networking/common.py#L340-L347
def _validate(self, request):
    self._check_url_scheme(request)
    self._check_proxies(request.proxies or self.proxies)  # ← 用的是 request.proxies（已被 clean_proxies 注入）
```

由于 `request.proxies` 此时已经非空（含 `{'all': '...'}`），所以这里校验的就是**地理验证代理**，而不是 Handler 全局的 `self.proxies`。校验包括：
- 是否声明 `Features.ALL_PROXY` 支持
- 代理 URL 是否有合法 scheme（http/https/socks4/socks5/socks5h 等）
- scheme 是否在 `_SUPPORTED_PROXY_SCHEMES` 中

如果校验失败抛 `UnsupportedRequest`，会继续尝试下一个 Handler。

### 3.6 Step 6：`_get_proxies` 返回请求级代理（覆盖全局）

`_send()` 内部各 Handler 都调用：

```python
proxies = self._get_proxies(request)
```

networking/common.py#L287-L288（`yt_dlp/networking/common.py:L287-L288`）：

```python
def _get_proxies(self, request):
    return (request.proxies or self.proxies).copy()
```

因为 `request.proxies = {'all': '<geo-url>'}` 非空，所以 **or 短路**，返回的就是地理验证代理 dict 的副本，完全忽略了 Handler 构造时注入的 `self.proxies`（全局 `--proxy`）。

### 3.7 Step 7：各网络客户端实际消费地理验证代理

各 Handler 把 `_get_proxies` 返回的 dict（含 `{'all': '<geo-url>'}`）传给自己的底层机制：

| Handler | 消费方式 | 关键代码位置 |
|---------|---------|-------------|
| **UrllibRH** | `_get_instance(proxies=...)` 用 `ProxyHandler(proxies)` 构建 opener；`select_proxy(url, proxies)` 按 scheme 精确匹配失败后 fallback 到 `'all'` | _urllib.py#L372-L399（`yt_dlp/networking/_urllib.py:L372-L399`）、`yt_dlp/networking/_urllib.py:L251-L260` |
| **RequestsRH** | `session.request(proxies=self._get_proxies(request))` 直接把 dict 传给 requests；`requests.adapters.select_proxy` 已被替换为 yt-dlp 的实现，同样走 scheme → `'all'` fallback | _requests.py#L323-L333（`yt_dlp/networking/_requests.py:L323-L333`）、`yt_dlp/networking/_requests.py:L108-L111` |
| **CurlCFFIRH** | `proxies = self._get_proxies(request)` 后再 `proxy = select_proxy(request.url, proxies=proxies)` 选出；`session.curl.setopt(CurlOpt.PROXY, proxy)` 设给 libcurl | _curlcffi.py#L258-L279（`yt_dlp/networking/_curlcffi.py:L258-L279`） |
| **WebsocketsRH** | `proxy = select_proxy(request.url, self._get_proxies(request))`；若有代理则走 SOCKS 建连 | _websockets.py#L143-L157（`yt_dlp/networking/_websockets.py:L143-L157`） |

`select_proxy` 的 fallback 逻辑（utils/networking.py#L246-L256（`yt_dlp/utils/networking.py:L246-L256`））保证了即使请求 URL 是 `https://` 或 `wss://`，dict 里唯一的 `'all'` key 也能被正确匹配。

### 3.8 小结：地理验证代理覆盖普通代理的关键节点

```
用户同时设置 --proxy A 和 --geo-verification-proxy B
        │
        ├──► YoutubeDL.proxies = {'all': 'A'}    （全局代理）
        │         │
        │         ▼
        │    build_request_director()
        │         │
        │         └──► self.proxies = {'all': 'A'}  （各 RequestHandler 成员变量）
        │
        └──► extractor 需要地理验证的请求
                  │
                  ▼
            geo_verification_headers()
                  │ {'Ytdl-request-proxy': 'B'}
                  ▼
            request.headers['Ytdl-Request-Proxy'] = 'B'
                  │
                  ▼
            YoutubeDL.urlopen(req)
                  │
                  ▼
            clean_proxies(proxies=req.proxies, headers=req.headers)
                  │  ① headers.pop('Ytdl-Request-Proxy') → 'B'
                  │  ② req.proxies.clear()               ← 丢弃按协议代理和 NO_PROXY
                  │  ③ req.proxies['all'] = 'B'
                  ▼
            req.proxies = {'all': 'B'}
                  │
                  ▼
            RequestHandler._get_proxies(request)
                  │  (request.proxies or self.proxies)
                  │  = {'all': 'B'}     ← 短路，self.proxies={'all':'A'} 被忽略
                  ▼
            实际网络请求走代理 B
```

**两次调用 `clean_proxies` 的区别**：

| 调用时机 | 作用对象 | 是否会遇到 Ytdl-Request-Proxy | 效果 |
|---------|---------|----------------------------|-----|
| `build_request_director()` | 全局 `self.proxies.copy()` + 全局 headers | **不会**（全局 headers 里没有这个 key） | 只清洗 `__noproxy__`、补 scheme、兼容替换 |
| `YoutubeDL.urlopen(req)` | `req.proxies` + `req.headers` | **会**（extractor 注入的） | 把 header 转为 `req.proxies['all']`，覆盖一切 |

---

## 四、异常处理：代理相关错误的捕获和转换

### 4.1 异常层次结构

networking/exceptions.py（`yt_dlp/networking/exceptions.py`） 定义了代理相关的异常链：

```
YoutubeDLError
  └── RequestError          # 所有请求异常的基类，带 handler 和 cause 属性
        ├── UnsupportedRequest   # Handler 不支持该请求（含代理 scheme 不支持）
        ├── NoSupportingHandlers # 所有 Handler 都无法处理
        ├── TransportError       # 网络传输层错误
        │     ├── SSLError
        │     │     └── CertificateVerifyError
        │     ├── IncompleteRead
        │     └── ProxyError     # 代理连接失败
        └── HTTPError            # HTTP 状态码错误
```

### 4.2 `RequestDirector.send` 的异常分发

networking/common.py#L94-L130（`yt_dlp/networking/common.py:L94-L130`） 中 `RequestDirector.send()` 遍历所有 Handler：

1. `handler.validate(request)` 抛 `UnsupportedRequest` → 记入 `unsupported_errors`，尝试下一个 Handler
2. `handler.send(request)` 抛 `RequestError`（含 `ProxyError`） → **直接 raise**，不尝试下一个 Handler
3. `handler.send(request)` 抛其他异常 → 记入 `unexpected_errors`，尝试下一个 Handler
4. 全部 Handler 失败 → 抛 `NoSupportingHandlers`

### 4.3 各 Handler 的代理异常处理

#### UrllibRH

networking/_urllib.py#L418-L443（`yt_dlp/networking/_urllib.py:L418-L443`）

```python
except urllib.error.URLError as e:
    cause = e.reason
    if 'tunnel connection failed' in str(cause).lower() or isinstance(cause, SocksProxyError):
        raise ProxyError(cause=e) from e    # HTTP CONNECT 隧道失败 / SOCKS 代理错误
    handle_response_read_exceptions(cause)
    raise TransportError(cause=e) from e
```

- SOCKS 代理的 `SocksProxyError` → `ProxyError`
- HTTP CONNECT 隧道失败 → `ProxyError`
- 其他 URL 错误 → `TransportError`

#### RequestsRH

networking/_requests.py#L339-L356（`yt_dlp/networking/_requests.py:L339-L356`）

```python
except requests.exceptions.ProxyError as e:
    raise ProxyError(cause=e) from e
```

- requests 自身的 `ProxyError` → yt-dlp 的 `ProxyError`
- `SocksHTTPConnection._new_conn` 中的 `SocksProxyError` → `urllib3.exceptions.ProxyError` → `requests.exceptions.ProxyError`

#### CurlCFFIRH

networking/_curlcffi.py#L317-L334（`yt_dlp/networking/_curlcffi.py:L317-L334`）

```python
elif e.code == CurlECode.PROXY or (e.code == CurlECode.RECV_ERROR and 'CONNECT' in str(e)):
    raise ProxyError(cause=e) from e
```

- curl 的 `CurlECode.PROXY` 错误码 → `ProxyError`
- `RECV_ERROR` 且包含 `CONNECT` → `ProxyError`（HTTP CONNECT 隧道失败）

#### WebsocketsRH

networking/_websockets.py#L171-L172（`yt_dlp/networking/_websockets.py:L171-L172`）

```python
except SocksProxyError as e:
    raise ProxyError(cause=e) from e
```

- `SocksProxyError` → `ProxyError`（连接阶段和收发阶段均会捕获）

### 4.4 `YoutubeDL.urlopen` 的二次处理

YoutubeDL.py#L4294-L4328（`yt_dlp/YoutubeDL.py:L4294-L4328`） 在 `RequestDirector.send()` 之上增加友好提示：

- `NoSupportingHandlers` + `unsupported proxy type: "https"` → 提示安装 `requests` 或 `curl_cffi`
- `SSLError` + `UNSAFE_LEGACY_RENEGOTIATION_DISABLED` → 提示用 `--legacy-server-connect`
- 地理限制错误 → 提示用 `--proxy` 或 VPN

### 4.5 `wrap_request_errors` 装饰器

networking/_helper.py#L190-L199（`yt_dlp/networking/_helper.py:L190-L199`） 确保所有从 `RequestHandler.validate()` 和 `RequestHandler.send()` 抛出的 `RequestError` 自动绑定 `handler` 属性，便于后续定位是哪个 Handler 报的错。

---

## 五、外部网络请求的代理路径

前面的章节覆盖了 yt-dlp 内置网络栈（Request → RequestDirector → RequestHandler）的代理路径。但项目中有三类组件**绕过内置网络栈**，需要独立的代理传递机制：YouTube PoToken 提供者、外部 JS 运行时（Deno/Bun）、外部下载器（ffmpeg/aria2c 等）。

### 5.1 YouTube PoToken 提供者的代理路径

PoToken 是 YouTube 要求的反机器人令牌。获取过程可能需要外部 HTTP 请求（访问 YouTube 或第三方服务），代理必须传进去。

#### 5.1.1 代理注入入口：`_fetch_po_token`

`yt_dlp/extractor/youtube/_video.py:L2849-L2901` 构造 `PoTokenRequest` 时注入代理：

```python
def _fetch_po_token(self, client, **kwargs):
    proxies = self._downloader.proxies.copy()
    clean_proxies(proxies, headers)

    pot_request = PoTokenRequest(
        ...
        request_proxy=(
            select_proxy('https://www.youtube.com', proxies)
            or select_proxy(f'https://{innertube_host}', proxies)
        ),
        ...
    )
```

**关键逻辑**：
1. 从 `self._downloader.proxies`（即 `YoutubeDL.proxies`）取全局代理 dict
2. `clean_proxies(proxies, headers)` 清洗（与 `YoutubeDL.urlopen` 中一致）
3. 用 `select_proxy('https://www.youtube.com', proxies)` 选出 YouTube 主站代理
4. 若主站没有匹配的代理，fallback 到 innertube host 的代理
5. **结果是一个字符串（代理 URL）或 None**，存入 `PoTokenRequest.request_proxy`

#### 5.1.2 `PoTokenRequest` 中的代理字段

pot/provider.py#L46-L74（`yt_dlp/extractor/youtube/pot/provider.py:L46-L74`） 的 `PoTokenRequest` dataclass 包含完整的网络参数：

```python
@dataclasses.dataclass
class PoTokenRequest:
    request_proxy: str | None = None
    request_headers: HTTPHeaderDict = dataclasses.field(default_factory=HTTPHeaderDict)
    request_timeout: float | None = None
    request_source_address: str | None = None
    request_verify_tls: bool = True
```

注意 `request_proxy` 是**单个字符串**而非 dict，因为它只服务于 YouTube 相关的请求。

#### 5.1.3 Provider 校验代理 scheme

pot/provider.py#L164-L174（`yt_dlp/extractor/youtube/pot/provider.py:L164-L174`） 中，`__validate_external_request_features` 检查 `request_proxy` 的 scheme 是否在该 Provider 声明的 `_SUPPORTED_EXTERNAL_REQUEST_FEATURES` 中：

```python
if request.request_proxy:
    scheme = urllib.parse.urlparse(request.request_proxy).scheme
    if scheme.lower() not in self._supported_proxy_schemes:
        raise PoTokenProviderRejectedRequest(...)
```

`_supported_proxy_schemes` 由 `_SUPPORTED_EXTERNAL_REQUEST_FEATURES` 映射而来（pot/provider.py#L149-L162（`yt_dlp/extractor/youtube/pot/provider.py:L149-L162`）），只有声明了 `PROXY_SCHEME_HTTP` 等特性的 Provider 才能使用对应协议的代理。不发起外部请求的 Provider 设 `_SUPPORTED_EXTERNAL_REQUEST_FEATURES = None` 跳过检查。

#### 5.1.4 `_request_webpage` 把代理传回内置网络栈

pot/provider.py#L203-L231（`yt_dlp/extractor/youtube/pot/provider.py:L203-L231`） 是 PoToken Provider 使用内置网络栈的入口：

```python
def _request_webpage(self, request, pot_request=None, note=None, **kwargs):
    req = request.copy()
    if pot_request is not None:
        req.headers = HTTPHeaderDict(pot_request.request_headers, req.headers)
        req.proxies = req.proxies or ({'all': pot_request.request_proxy} if pot_request.request_proxy else {})
        if pot_request.request_cookiejar is not None:
            req.extensions['cookiejar'] = req.extensions.get('cookiejar', pot_request.request_cookiejar)
    return self.ie._downloader.urlopen(req)
```

**代理转换**：`pot_request.request_proxy`（字符串）→ `req.proxies = {'all': URL}`，然后走 `YoutubeDL.urlopen(req)` 的正常流程（`clean_proxies` 会再次清洗，但此时 headers 中已无 `Ytdl-Request-Proxy`，所以不会覆盖）。

如果 Provider 不用内置网络栈而是自建外部连接（如调用外部服务），则需自行处理代理，此时 `_SUPPORTED_EXTERNAL_REQUEST_FEATURES` 的校验就至关重要。

#### 5.1.5 PoToken 代理路径总图

```
YoutubeDL.proxies {'all': URL_A}
        │
        ▼
_fetch_po_token()
   ├── proxies = self._downloader.proxies.copy()
   ├── clean_proxies(proxies, headers)
   └── request_proxy = select_proxy('https://www.youtube.com', proxies)  → URL_A
        │
        ▼
PoTokenRequest(request_proxy='URL_A')
        │
        ├──► __validate_external_request_features()   校验 scheme
        │
        ├──► Provider 使用 _request_webpage()        → req.proxies = {'all': 'URL_A'}
        │         │                                     → YoutubeDL.urlopen(req)
        │         └──► 内置网络栈走代理 URL_A
        │
        └──► Provider 自建外部连接                    → 自行使用 request_proxy
```

### 5.2 外部 JS 运行时（Deno / Bun）的代理路径

Deno 和 Bun 是 yt-dlp 用来执行 JavaScript 挑战求解器的外部进程。它们需要代理来下载 NPM 包或访问远程脚本。代理通过**进程环境变量**注入。

#### 5.2.1 Deno：`_get_env_options`

deno.py#L89-L99（`yt_dlp/extractor/youtube/jsc/_builtin/deno.py:L89-L99`）：

```python
def _get_env_options(self) -> dict[str, str]:
    options = os.environ.copy()
    request_proxies = self.ie._downloader.proxies.copy()
    clean_proxies(request_proxies, HTTPHeaderDict())
    if 'all' in request_proxies and request_proxies['all'] is not None:
        options['HTTP_PROXY'] = options['HTTPS_PROXY'] = request_proxies['all']
    for key, env in (('http', 'HTTP_PROXY'), ('https', 'HTTPS_PROXY'), ('no', 'NO_PROXY')):
        if key in request_proxies and request_proxies[key] is not None:
            options[env] = request_proxies[key]
    return options
```

**转换逻辑**：
1. `self.ie._downloader.proxies` → 取全局代理 dict（`YoutubeDL.proxies`）
2. `clean_proxies(request_proxies, HTTPHeaderDict())` → 清洗（header 为空，所以 `Ytdl-Request-Proxy` 不会被提取）
3. `'all'` key → 同时设 `HTTP_PROXY` + `HTTPS_PROXY`
4. 按 scheme 分配 → `'http'` → `HTTP_PROXY`、`'https'` → `HTTPS_PROXY`、`'no'` → `NO_PROXY`（可覆盖 `'all'` 的设置）
5. 环境变量 dict 传给 `Popen(env=...)`，Deno 进程会自动读取

#### 5.2.2 Bun：`_get_env_options` + 代理兼容性检查

bun.py#L91-L114（`yt_dlp/extractor/youtube/jsc/_builtin/bun.py:L91-L114`）：

```python
def _get_env_options(self) -> dict[str, str]:
    options = os.environ.copy()
    request_proxies = self.ie._downloader.proxies.copy()
    clean_proxies(request_proxies, HTTPHeaderDict())
    if request_proxies.get('all') is not None:
        options['HTTP_PROXY'] = options['HTTPS_PROXY'] = request_proxies['all']
    for key, env in (('http', 'HTTP_PROXY'), ('https', 'HTTPS_PROXY')):
        val = request_proxies.get(key)
        if val is not None:
            options[env] = val
    if self.ie.get_param('nocheckcertificate'):
        options['NODE_TLS_REJECT_UNAUTHORIZED'] = '0'
    options['BUN_RUNTIME_TRANSPILER_CACHE_PATH'] = '0'
    return options
```

**与 Deno 的差异**：
- **没有 `'no'` → `NO_PROXY` 的转换**：Bun 不处理 NO_PROXY
- **新增 TLS 跳过**：`nocheckcertificate` → `NODE_TLS_REJECT_UNAUTHORIZED=0`
- **新增缓存禁用**：`BUN_RUNTIME_TRANSPILER_CACHE_PATH=0`

Bun 还有一层额外的代理兼容性检查 `yt_dlp/extractor/youtube/jsc/_builtin/bun.py:L80-L89`：

```python
SUPPORTED_PROXY_SCHEMES = ['http', 'https']

def _check_env_proxies(self, env):
    for key in ('HTTP_PROXY', 'HTTPS_PROXY'):
        proxy = env.get(key)
        if not proxy:
            continue
        scheme = urllib.parse.urlparse(proxy).scheme.lower()
        if scheme not in self.SUPPORTED_PROXY_SCHEMES:
            return scheme  # 返回不支持的 scheme
    return None
```

如果检测到 SOCKS 等不支持的代理 scheme，Bun 会跳过 NPM 远程下载并给出警告（bun.py#L62-L67（`yt_dlp/extractor/youtube/jsc/_builtin/bun.py:L62-L67`））。这是因为 Bun 的 NPM 包下载器仅支持 HTTP/HTTPS 代理。

#### 5.2.3 JS 运行时代理路径总图

```
YoutubeDL.proxies {'all': URL_A, 'no': 'localhost'}
        │
        ▼
DenoJCP / BunJCP._get_env_options()
   ├── proxies = self.ie._downloader.proxies.copy()
   ├── clean_proxies(proxies, HTTPHeaderDict())
   │
   ├── 'all' → HTTP_PROXY=URL_A, HTTPS_PROXY=URL_A
   ├── 'http' → HTTP_PROXY (可覆盖 'all')
   ├── 'https' → HTTPS_PROXY (可覆盖 'all')
   ├── 'no' → NO_PROXY='localhost'  (仅 Deno)
   │
   └── Popen(cmd, env={HTTP_PROXY: ..., HTTPS_PROXY: ..., ...})
         │
         ▼
   Deno/Bun 进程继承环境变量
```

### 5.3 JS 挑战求解脚本的四种加载源及代理路径

JS 挑战求解脚本（yt.solver.lib.js / yt.solver.core.js）有 4 种加载源，按优先级依次尝试。不同来源的代理路径完全不同。

#### 5.3.1 四种加载源与优先级

ejs.py#L259-L264（`yt_dlp/extractor/youtube/jsc/_builtin/ejs.py:L259-L264`） 定义了 `_iter_script_sources` 的顺序：

```
PYPACKAGE → CACHE → BUILTIN → WEB
```

| 来源 | `ScriptSource` 枚举值 | 触发条件 | 是否需要代理 |
|------|----------------------|---------|-------------|
| **PYPACKAGE** | `python package` | 已安装 `yt_dlp_ejs` PyPI 包 | ❌ 不需要（从本地 Python 包读取） |
| **CACHE** | `cache` | 之前从 WEB 下载过且已缓存到磁盘 | ❌ 不需要（从 yt-dlp 本地 cache 读取） |
| **BUILTIN** | `builtin` | yt-dlp 发行版内置 vendored 脚本 | ❌ 不需要（从包资源读取） |
| **WEB** | `web` | 用户启用了 `--remote-components ejs:github` | ✅ **需要**（从 GitHub Release 下载） |

每个来源返回一个 `Script` 对象（含 code、version、variant、source），通过 `hashlib.sha3_512(code.encode())` 计算哈希与 `_ALLOWED_HASHES` 白名单对比校验，未通过则继续尝试下一来源。

#### 5.3.2 PYPACKAGE 源：本地 Python 包读取

ejs.py#L266-L275（`yt_dlp/extractor/youtube/jsc/_builtin/ejs.py:L266-L275`）：

```python
def _pypackage_source(self, script_type: ScriptType, /) -> Script | None:
    if not _has_ejs:
        return None
    try:
        code = yt_dlp_ejs.yt.solver.core() if script_type is ScriptType.CORE else yt_dlp_ejs.yt.solver.lib()
    except Exception as e:
        self.logger.warning(f'Failed to load ... from python package: {e}')
        return None
    return Script(script_type, ScriptVariant.MINIFIED, ScriptSource.PYPACKAGE, yt_dlp_ejs.version, code)
```

直接从已安装的 `yt_dlp_ejs` 包中调用函数获取代码字符串。**无网络请求，不走任何代理路径**。

#### 5.3.3 CACHE 源：本地磁盘缓存读取

ejs.py#L277-L280（`yt_dlp/extractor/youtube/jsc/_builtin/ejs.py:L277-L280`）：

```python
def _cached_source(self, script_type: ScriptType, /) -> Script | None:
    if data := self.ie.cache.load(self._CACHE_SECTION, script_type.value):
        return Script(script_type, ScriptVariant(data['variant']), ScriptSource.CACHE, data['version'], data['code'])
    return None
```

从 yt-dlp 本地缓存（`_CACHE_SECTION = 'challenge-solver'`）读取之前 WEB 源下载成功后存入的 `{version, variant, code}`。**无网络请求，不走任何代理路径**。

若版本不匹配（minor version 不符）或哈希校验失败，会主动清空缓存（ejs.py#L228-L244（`yt_dlp/extractor/youtube/jsc/_builtin/ejs.py:L228-L244`））：

```python
if source is ScriptSource.CACHE:
    self.logger.debug('Clearing outdated cached script')
    self.ie.cache.store(self._CACHE_SECTION, script_type.value, None)
```

#### 5.3.4 BUILTIN 源：包内 vendored 资源读取

ejs.py#L282-L289（`yt_dlp/extractor/youtube/jsc/_builtin/ejs.py:L282-L289`）：

```python
def _builtin_source(self, script_type: ScriptType, /) -> Script | None:
    error_hook = lambda _: self.logger.warning(f'Failed to read builtin ... script')
    code = vendor.load_script(self._SCRIPT_FILENAMES[script_type], error_hook=error_hook)
    if code:
        return Script(script_type, ScriptVariant.UNMINIFIED, ScriptSource.BUILTIN, self._SCRIPT_VERSION, code)
    return None
```

从 `yt_dlp/extractor/youtube/jsc/_builtin/vendor/` 目录读取 vendored 脚本。**无网络请求，不走任何代理路径**。

注意返回的 variant 是 `ScriptVariant.UNMINIFIED`，因为 vendored 目录下的是未压缩版本。

#### 5.3.5 WEB 源：从 GitHub Release 下载（需要代理）

ejs.py#L291-L305（`yt_dlp/extractor/youtube/jsc/_builtin/ejs.py:L291-L305`）：

```python
def _web_release_source(self, script_type: ScriptType, /):
    if 'ejs:github' not in (self.ie.get_param('remote_components') or ()):
        return self._skip_component('ejs:github')
    url = f'https://github.com/{self._REPOSITORY}/releases/download/{self._SCRIPT_VERSION}/{self._MIN_SCRIPT_FILENAMES[script_type]}'
    if code := self.ie._download_webpage_with_retries(
        url, None, f'[{self.logger.prefix}] Downloading challenge solver {script_type.value} script from  {url}',
        f'[{self.logger.prefix}] Failed to download challenge solver {script_type.value} script', fatal=False,
    ):
        self.ie.cache.store(self._CACHE_SECTION, script_type.value, {
            'version': self._SCRIPT_VERSION,
            'variant': ScriptVariant.MINIFIED.value,
            'code': code,
        })
        return Script(script_type, ScriptVariant.MINIFIED, ScriptSource.WEB, self._SCRIPT_VERSION, code)
    return None
```

**完整代理路径**：

```
用户启用 --remote-components ejs:github
        │
        ▼
_web_release_source()
        │
        ▼
self.ie._download_webpage_with_retries(url, ...)
        │  走 extractor 常规下载路径
        ▼
YoutubeDL.urlopen(req)
        │  req.proxies 默认为空，headers 中没有地理验证代理头
        ▼
clean_proxies(req.proxies, req.headers)
        │
        ▼
内置网络栈 → 正常代理流程
```

由于使用了 `_download_webpage_with_retries` → `_request_webpage` → `_create_request` → `YoutubeDL.urlopen` 的完整 extractor 下载链路，WEB 源下载会走 yt-dlp 的内置网络栈：全局 `--proxy` 与系统环境变量代理会生效；但这条 GitHub Release 请求本身没有注入 `geo_verification_headers()`，所以不会自动使用 `--geo-verification-proxy`。下载成功后代码存入本地 cache，下次直接走 CACHE 源不再需要网络。

#### 5.3.6 脚本加载源的代理路径对比表

| 来源 | 网络请求 | 代理参与 | 代理来源 |
|------|---------|---------|---------|
| PYPACKAGE | ❌ | ❌ | — |
| CACHE | ❌ | ❌ | — |
| BUILTIN | ❌ | ❌ | — |
| WEB | ✅ GitHub HTTPS | ✅ | 内置网络栈代理（全局 `--proxy`、系统环境变量；默认不含地理验证代理） |

### 5.4 PoToken 缓存键如何纳入代理信息

PoToken 的缓存设计明确将「代理」作为缓存隔离的一部分，避免不同出口 IP（不同代理）获取的 PoToken 在另一个出口 IP 下使用时报错。

#### 5.4.1 缓存键生成流程

缓存键在 pot/_director.py#L149-L151（`yt_dlp/extractor/youtube/pot/_director.py:L149-L151`） 生成：

```python
def _generate_key(self, bindings: dict) -> str:
    binding_string = ''.join(repr(dict(sorted(bindings.items()))))
    return hashlib.sha256(binding_string.encode()).hexdigest()
```

所有 key_bindings 先按 key 字母排序后转 repr 字符串，再做 SHA-256 哈希，得到固定长度的缓存键。

#### 5.4.2 `WebPoPCSP` 的 key_bindings（含代理字段）

唯一内置的缓存规范提供者是 webpo_cachespec.py#L17-L48（`yt_dlp/extractor/youtube/pot/_builtin/webpo_cachespec.py:L17-L48`）：

```python
@register_spec
class WebPoPCSP(PoTokenCacheSpecProvider, BuiltinIEContentProvider):
    PROVIDER_NAME = 'webpo'

    def generate_cache_spec(self, request: PoTokenRequest) -> PoTokenCacheSpec | None:
        ...
        return PoTokenCacheSpec(
            key_bindings={
                't': 'webpo',                                   # 类型标记
                'cb': content_binding,                          # 内容绑定（visitor_id 或 video_id）
                'cbt': content_binding_type.value,              # 绑定类型：visitor / video
                'ip': traverse_obj(request.innertube_context, ('client', 'remoteHost')),  # 客户端出口 IP
                'sa': request.request_source_address,           # 源地址绑定（--source-address）
                'px': request.request_proxy,                    # ← 代理 URL 字符串
            },
            default_ttl=21600,    # 6 小时
            write_policy=write_policy,
        )
```

**代理信息以三个维度纳入缓存隔离**：

| 字段 | key | 来源 | 含义 |
|------|-----|------|------|
| 代理 URL | `px` | `PoTokenRequest.request_proxy`（即 `select_proxy('https://www.youtube.com', YoutubeDL.proxies)` 的结果） | 完整代理 URL 字符串，含 scheme、user:pass、host、port。不同代理 → 不同缓存键 |
| 源地址 | `sa` | `PoTokenRequest.request_source_address`（即 `YoutubeDL.params['source_address']`） | `--source-address` 绑定的出站 IP。不同出站 IP → 不同缓存键 |
| 远程 IP | `ip` | `request.innertube_context.client.remoteHost` | YouTube 端看到的客户端出口 IP（从 innertube API 返回）。即使代理相同，若出口 IP 不同（如轮询代理池），缓存也会隔离 |

这样保证了：换代理、换出口 IP、换源地址时，旧缓存不会被误命中，避免 YouTube 因「PoToken 的签发 IP 与当前请求 IP 不匹配」而拒绝。

#### 5.4.3 `_generate_key_bindings` 的额外处理

`yt_dlp/extractor/youtube/pot/_director.py:L138-L147` 在生成缓存键前对 bindings 做了清洗和补充：

```python
def _generate_key_bindings(self, spec: PoTokenCacheSpec) -> dict[str, str]:
    bindings_cleaned = {
        **{k: v for k, v in spec.key_bindings.items() if v is not None},   # 过滤 None 值
        '_dlp_cache': 'v1',                                                # 缓存版本号
    }
    if spec._provider:
        bindings_cleaned['_p'] = spec._provider.PROVIDER_KEY               # 规范提供者标识（'webpo'）
    return bindings_cleaned
```

过滤 `None` 值意味着：如果没设代理（`request_proxy=None`），`px` 字段不会出现在 bindings 中；如果后来设了代理，`px` 就会出现 → 排序后的 repr 字符串不同 → 哈希不同 → 缓存键不同。

#### 5.4.4 PoToken 缓存键生成示例

| 场景 | `px` | `sa` | `ip` | 缓存键（SHA-256）是否相同 |
|------|------|------|------|---------------------------|
| 无代理，无 source-address | None | None | '1.2.3.4' | ✅ |
| `--proxy http://a:8080` | 'http://a:8080' | None | '5.6.7.8' | ❌ 不同 |
| `--proxy http://b:8080` | 'http://b:8080' | None | '9.9.9.9' | ❌ 不同 |
| 相同代理，同 session 同 visitor | 'http://a:8080' | None | '5.6.7.8' | ✅ 相同（缓存命中） |
| 相同代理，出口 IP 变了（代理池） | 'http://a:8080' | None | '5.6.7.9' | ❌ 不同（缓存隔离） |
| `--source-address 10.0.0.1` | None | '10.0.0.1' | '1.2.3.4' | ❌ 不同 |

### 5.5 外部下载器的代理路径

外部下载器（ffmpeg、aria2c、curl、wget 等）是独立的系统进程，代理通过**命令行参数**或**环境变量**注入。

#### 5.5.1 curl：`--proxy` 命令行参数

external.py#L215-L249（`yt_dlp/downloader/external.py:L215-L249`） 的 `CurlFD._make_cmd`：

```python
cmd += self._option('--proxy', 'proxy')
```

`self._option('--proxy', 'proxy')` 读取 `self.params['proxy']`（即 `YoutubeDL.params['proxy']`，原始 `--proxy` 参数值），如果存在则生成 `['--proxy', '<URL>']`。curl 原生支持 HTTP/HTTPS/SOCKS 代理。

#### 5.5.2 aria2c：`--all-proxy` 命令行参数

external.py#L312-L348（`yt_dlp/downloader/external.py:L312-L348`） 的 `Aria2cFD._make_cmd`：

```python
cmd += self._option('--all-proxy', 'proxy')
```

同样读取 `self.params['proxy']`，生成 `['--all-proxy', '<URL>']`。aria2c 的 `--all-proxy` 对所有协议生效。

#### 5.5.3 wget：`--execute http_proxy=...` 命令行参数

external.py#L281-L301（`yt_dlp/downloader/external.py:L281-L301`） 的 `WgetFD._make_cmd`：

```python
proxy = self.params.get('proxy')
if proxy:
    for var in ('http_proxy', 'https_proxy'):
        cmd += ['--execute', f'{var}={proxy}']
```

wget 没有统一的 `--proxy` 参数，而是通过 `--execute http_proxy=...` 和 `--execute https_proxy=...` 分别设置。同一代理值同时设给两者。

#### 5.5.4 ffmpeg：环境变量注入

external.py#L411-L428（`yt_dlp/downloader/external.py:L411-L428`） 的 `FFmpegFD._call_downloader`：

```python
env = None
proxy = self.params.get('proxy')
if proxy:
    if not re.match(r'[\da-zA-Z]+://', proxy):
        proxy = f'http://{proxy}'
    if proxy.startswith('socks'):
        self.report_warning(
            f'{self.get_basename()} does not support SOCKS proxies. ...')
    env = os.environ.copy()
    env['HTTP_PROXY'] = proxy
    env['http_proxy'] = proxy
```

ffmpeg 的代理注入方式最特殊：
1. 从 `self.params.get('proxy')` 取原始 `--proxy` 参数
2. 补全 scheme（无 scheme 时加 `http://`）
3. **SOCKS 代理仅给出警告**，不会阻止执行但很可能失败
4. 通过 `Popen(env=env)` 设置 `HTTP_PROXY` + `http_proxy` 环境变量
5. 代码注释指出 ffmpeg 已支持 `-http_proxy` 选项，但版本检测尚未实现

#### 5.5.5 HttpieFD / AxelFD：无显式代理注入

- `yt_dlp/downloader/external.py:L351-L369`：`_make_cmd` 中不传任何代理参数
- `yt_dlp/downloader/external.py:L262-L275`：同上

#### 5.5.6 外部下载器代理来源差异

| 下载器 | 代理来源 | 传递方式 | SOCKS 支持 |
|-------|---------|---------|-----------|
| curl | `params['proxy']`（原始值） | `--proxy URL` 命令行参数 | ✅ |
| aria2c | `params['proxy']`（原始值） | `--all-proxy URL` 命令行参数 | ✅ |
| wget | `params['proxy']`（原始值） | `--execute http_proxy=URL` 命令行参数 | ❌ |
| ffmpeg | `params['proxy']`（原始值） | `HTTP_PROXY`/`http_proxy` 环境变量 | ❌（仅警告） |
| httpie | 无显式注入 | 依赖工具自身行为 / 继承环境 | 取决于工具 |
| axel | 无显式注入 | 依赖工具自身行为 / 继承环境 | 取决于工具 |

**重要区别**：外部下载器显式注入代理时直接读取 `self.params.get('proxy')`，这是 `--proxy` 的**原始参数值**，**不经过** `YoutubeDL.proxies` 计算属性的 `urllib.request.getproxies()` fallback，也**不经过** `clean_proxies` 的 scheme 补全和兼容替换。如果用户没有设 `--proxy`，yt-dlp 不会主动为外部下载器生成代理参数；不过这些子进程默认继承当前环境，外部工具自身仍可能读取宿主环境里的 `HTTP_PROXY` / `HTTPS_PROXY`。

---

## 流程总图

```
命令行 --proxy / 环境变量 HTTP_PROXY        命令行 --geo-verification-proxy
        │                                        │
        ▼                                        ▼
  YoutubeDL.params['proxy']            YoutubeDL.params['geo_verification_proxy']
        │                                        │
        ▼                                        │
  YoutubeDL.proxies (cached_property)            │
   ├── 有 --proxy → {'all': URL}                 │
   └── 无 --proxy → urllib.request.getproxies()  │
        │                                        │
        ▼                                        │
  build_request_director()                       │
   ├── clean_proxies() 清洗                      │
   │    ├── 提取 Ytdl-Request-Proxy header       │  （全局 header 中没有，跳过）
   │    ├── __noproxy__ → None                   │
   │    ├── 补 http:// scheme                    │
   │    └── socks5→socks5h, socks→socks4         │
   │                                             │
   └── 传入各 RequestHandler(proxies=...)        │
        │  self.proxies = {'all': URL_A}         │
        │                                        │
        │                          ┌─────────────┘
        │                          │  extractor 显式调用
        │                          ▼
        │                geo_verification_headers()
        │                          │ {'Ytdl-request-proxy': URL_B}
        │                          ▼
        │                _create_request()
        │                          │ request.headers['Ytdl-Request-Proxy'] = URL_B
        │                          │ request.proxies = {}
        │                          ▼
        │                YoutubeDL.urlopen(req)
        │                          │
        │                          ▼
        │                clean_proxies(req.proxies, req.headers)
        │                          │  ① pop('Ytdl-Request-Proxy') → URL_B
        │                          │  ② req.proxies.clear()
        │                          │  ③ req.proxies['all'] = URL_B
        │                          ▼
        │                req.proxies = {'all': URL_B}
        │                          │
        ▼                          ▼
  RequestHandler._get_proxies(request)
   → (request.proxies or self.proxies).copy()
   → 当 request.proxies 非空时，短路覆盖 self.proxies
        │
        ▼
  select_proxy(url, proxies)  统一选择
   ├── 检查 no 排除列表  （被 Ytdl-Request-Proxy 清空）
   └── 匹配 scheme → fallback 'all'  → 命中 URL_B
        │
        ├─ UrllibRH → ProxyHandler + SocksConnection
        ├─ RequestsRH → session.request(proxies=...) + SocksProxyManager
        ├─ CurlCFFIRH → CurlOpt.PROXY + CurlOpt.NOPROXY
        └─ WebsocketsRH → create_socks_proxy_socket()
        │
        ▼
  异常转换
   ├── SocksProxyError / tunnel failed → ProxyError
   ├── requests.exceptions.ProxyError → ProxyError
   ├── CurlECode.PROXY / RECV_ERROR+CONNECT → ProxyError
   └── 其他网络错误 → TransportError
        │
        ▼
  YoutubeDL.urlopen 二次处理
   ├── NoSupportingHandlers → 友好提示（缺依赖等）
   └── SSLError → 建议用 --legacy-server-connect


  ═══════════════════════════════════════════
  ║  以下为绕过内置网络栈的外部代理路径  ║
  ═══════════════════════════════════════════

  【PoToken 提供者】
  YoutubeDL.proxies
        │
        ▼
  _fetch_po_token()
   ├── proxies.copy() + clean_proxies()
   └── request_proxy = select_proxy('https://www.youtube.com', proxies)
        │  字符串 URL 或 None
        ▼
  PoTokenRequest.request_proxy
        │
        ├──► _request_webpage()
        │      → req.proxies = {'all': URL}
        │      → YoutubeDL.urlopen(req) → 内置网络栈
        │
        └──► Provider 自建外部连接 → 自行消费 request_proxy

  【Deno / Bun JS 运行时】
  YoutubeDL.proxies
        │
        ▼
  DenoJCP / BunJCP._get_env_options()
   ├── proxies.copy() + clean_proxies()
   ├── 'all' → HTTP_PROXY + HTTPS_PROXY
   ├── 'http'/'https' → HTTP_PROXY/HTTPS_PROXY (覆盖 'all')
   ├── 'no' → NO_PROXY (仅 Deno)
   └── Popen(cmd, env={HTTP_PROXY: ..., ...})
                │
                ▼
   外部进程通过环境变量使用代理

  【JS 挑战求解脚本加载】
  PYPACKAGE (yt_dlp_ejs) → 本地 Python 包函数调用，无网络
  CACHE (yt-dlp cache)  → 磁盘读取，无网络
  BUILTIN (vendor dir)  → 包资源读取，无网络
  WEB (GitHub Release)  → _download_webpage_with_retries()
                            → YoutubeDL.urlopen() → 内置网络栈（全局/环境代理）
                            → 成功后存入 CACHE

  【PoToken 缓存键（代理信息纳入隔离）】
  PoTokenRequest
   │  .request_proxy           → px
   │  .request_source_address  → sa
   │  .innertube_context.client.remoteHost → ip
   ▼
  WebPoPCSP.generate_cache_spec()
   → key_bindings {t, cb, cbt, ip, sa, px}
   → _generate_key_bindings()   过滤 None + 加 _dlp_cache + _p
   → _generate_key()            SHA-256(排序后的 repr 字符串)
   → 缓存键
         │
         ├──► cache.get(cache_key)   命中直接返回
         └──► cache.store(cache_key, response)

  【外部下载器】
  YoutubeDL.params['proxy']  ← 原始 --proxy 参数，不经 YoutubeDL.proxies
        │
        ├─► curl   → --proxy URL
        ├─► aria2c → --all-proxy URL
        ├─► wget   → --execute http_proxy=URL --execute https_proxy=URL
        ├─► ffmpeg → env{HTTP_PROXY=URL, http_proxy=URL}
        └─► httpie/axel → yt-dlp 无显式代理注入，是否使用代理取决于工具自身与继承环境
```
