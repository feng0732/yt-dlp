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

这个 header 会在后续 `clean_proxies` 中被提取，覆盖所有代理设置。

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

## 三、异常处理：代理相关错误的捕获和转换

### 3.1 异常层次结构

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

### 3.2 `RequestDirector.send` 的异常分发

networking/common.py#L94-L130（`yt_dlp/networking/common.py:L94-L130`） 中 `RequestDirector.send()` 遍历所有 Handler：

1. `handler.validate(request)` 抛 `UnsupportedRequest` → 记入 `unsupported_errors`，尝试下一个 Handler
2. `handler.send(request)` 抛 `RequestError`（含 `ProxyError`） → **直接 raise**，不尝试下一个 Handler
3. `handler.send(request)` 抛其他异常 → 记入 `unexpected_errors`，尝试下一个 Handler
4. 全部 Handler 失败 → 抛 `NoSupportingHandlers`

### 3.3 各 Handler 的代理异常处理

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

### 3.4 `YoutubeDL.urlopen` 的二次处理

YoutubeDL.py#L4294-L4328（`yt_dlp/YoutubeDL.py:L4294-L4328`） 在 `RequestDirector.send()` 之上增加友好提示：

- `NoSupportingHandlers` + `unsupported proxy type: "https"` → 提示安装 `requests` 或 `curl_cffi`
- `SSLError` + `UNSAFE_LEGACY_RENEGOTIATION_DISABLED` → 提示用 `--legacy-server-connect`
- 地理限制错误 → 提示用 `--proxy` 或 VPN

### 3.5 `wrap_request_errors` 装饰器

networking/_helper.py#L190-L199（`yt_dlp/networking/_helper.py:L190-L199`） 确保所有从 `RequestHandler.validate()` 和 `RequestHandler.send()` 抛出的 `RequestError` 自动绑定 `handler` 属性，便于后续定位是哪个 Handler 报的错。

---

## 流程总图

```
命令行 --proxy / 环境变量 HTTP_PROXY
        │
        ▼
  YoutubeDL.params['proxy']
        │
        ▼
  YoutubeDL.proxies (cached_property)
   ├── 有 --proxy → {'all': URL}
   └── 无 --proxy → urllib.request.getproxies()
        │
        ▼
  build_request_director()
   ├── clean_proxies() 清洗
   │    ├── 提取 Ytdl-Request-Proxy header
   │    ├── __noproxy__ → None
   │    ├── 补 http:// scheme
   │    └── socks5→socks5h, socks→socks4
   │
   └── 传入各 RequestHandler(proxies=...)
        │
        ▼
  RequestHandler._get_proxies(request)
   → request.proxies 优先，否则用 self.proxies
        │
        ▼
  select_proxy(url, proxies)  统一选择
   ├── 检查 no 排除列表
   └── 匹配 scheme → fallback 'all'
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
```
