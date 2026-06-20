# yt-dlp 浏览器 Cookie 导入流程

## 整体架构概览

Cookie 导入流程横跨三个层次：**来源识别 → 提取转换 → 请求侧使用**。核心代码集中在 [cookies.py](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py)，由 [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/YoutubeDL.py) 在运行时驱动，最终通过 networking 层的各 handler 注入到 HTTP 请求中。

```
用户命令行参数
    │
    ▼
YoutubeDL.cookiejar (cached_property)
    │
    ├── cookiesfrombrowser → load_cookies() → extract_cookies_from_browser()
    │       │
    │       ├── Firefox  → _extract_firefox_cookies()
    │       ├── Safari   → _extract_safari_cookies()
    │       └── Chromium → _extract_chrome_cookies() → ChromeCookieDecryptor
    │
    ├── cookiefile → YoutubeDLCookieJar.load()
    │
    └── 合并 → _merge_cookie_jars() → YoutubeDLCookieJar
                                            │
                                            ▼
                                   RequestHandler._get_cookiejar()
                                            │
                              ┌──────────────┼──────────────┐
                              ▼              ▼              ▼
                          Urllib        Requests       CurlCFFI
                       (HTTPCookie    (session.     (Session(
                       Processor)     cookies)      cookies=))
```

---

## 一、来源识别

### 1.1 命令行入口

在 [options.py#L1538-L1561](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/options.py#L1538-L1561) 定义了两个关键参数：

| 参数 | dest | 含义 |
|------|------|------|
| `--cookies` | `cookiefile` | Netscape 格式的 Cookie 文件路径 |
| `--cookies-from-browser` | `cookiesfrombrowser` | 浏览器 Cookie 导入规格 |

`--cookies-from-browser` 的格式为 `BROWSER[+KEYRING][:PROFILE][::CONTAINER]`，其中：
- **BROWSER**：浏览器名称，必填
- **KEYRING**：仅 Linux 上 Chromium 系浏览器需要，指定密钥环后端
- **PROFILE**：浏览器配置文件名或路径
- **CONTAINER**：仅 Firefox，指定容器名（`none` 表示无容器）

### 1.2 浏览器分类

在 [cookies.py#L49-L50](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L49-L50) 定义了支持的浏览器集合：

```python
CHROMIUM_BASED_BROWSERS = {'brave', 'chrome', 'chromium', 'edge', 'opera', 'vivaldi', 'whale'}
SUPPORTED_BROWSERS = CHROMIUM_BASED_BROWSERS | {'firefox', 'safari'}
```

三大浏览器族系分别对应完全不同的提取策略：

| 族系 | 数据库格式 | 加密方式 | 提取函数 |
|------|-----------|---------|---------|
| Firefox | `cookies.sqlite` (SQLite) | 无加密 | `_extract_firefox_cookies()` |
| Chromium 系 | `Cookies` (SQLite) | 平台相关加密 | `_extract_chrome_cookies()` |
| Safari | `Cookies.binarycookies` (二进制) | 无加密 | `_extract_safari_cookies()` |

### 1.3 规格解析

在 [_parse_browser_specification()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1155-L1162) 中完成命令行参数的校验：

1. 验证 `browser_name` 是否在 `SUPPORTED_BROWSERS` 中
2. 验证 `keyring` 是否在 `SUPPORTED_KEYRINGS` 中（Linux 上可选 `BASICTEXT`, `GNOMEKEYRING`, `KWALLET`, `KWALLET5`, `KWALLET6`）
3. 如果 `profile` 是一个文件系统路径（包含路径分隔符），则将其展开为绝对路径

返回四元组 `(browser_name, profile, keyring, container)`，供后续提取函数使用。

### 1.4 浏览器配置文件路径定位

每个浏览器族系根据当前操作系统定位用户数据目录：

**Firefox** — [_firefox_browser_dirs()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L203-L225)：

| 平台 | 路径 |
|------|------|
| Windows | `%APPDATA%\Mozilla\Firefox\Profiles`、`%LOCALAPPDATA%\Packages\Mozilla.Firefox_n80bbvh6b1yt2\...` |
| macOS | `~/Library/Application Support/Firefox/Profiles` |
| Linux | `~/.config/mozilla/firefox`（XDG）、`~/.mozilla/firefox`、Flatpak/Snap 路径 |

**Chromium 系** — [_get_chromium_based_browser_settings()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L234-L291)：

以 Windows 为例：

| 浏览器 | 路径 |
|--------|------|
| Chrome | `%LOCALAPPDATA%\Google\Chrome\User Data` |
| Edge | `%LOCALAPPDATA%\Microsoft\Edge\User Data` |
| Brave | `%LOCALAPPDATA%\BraveSoftware\Brave-Browser\User Data` |
| Opera | `%APPDATA%\Opera Software\Opera Stable` |

该函数同时返回 `keyring_name`（Linux/macOS 密钥环中浏览器对应的名称）和 `supports_profiles`（Opera 不支持多配置文件）。

**Safari** — [_extract_safari_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L568-L591)：

仅支持 macOS/iOS，路径为 `~/Library/Cookies/Cookies.binarycookies`，备选路径在容器化安装目录下。

---

## 二、提取转换

### 2.1 顶层调度

[load_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L93-L113) 是入口函数，被 [YoutubeDL.cookiejar](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/YoutubeDL.py#L4213-L4222) 属性调用：

```python
@functools.cached_property
def cookiejar(self):
    return load_cookies(
        self.params.get('cookiefile'),
        self.params.get('cookiesfrombrowser'), self)
```

`cached_property` 确保 Cookie 加载只执行一次。`load_cookies()` 的工作流程：

1. 如果 `browser_specification` 不为空，解析规格并调用 `extract_cookies_from_browser()` 得到一个 `YoutubeDLCookieJar`
2. 如果 `cookiefile` 不为空，创建 `YoutubeDLCookieJar` 并调用 `load()` 加载 Netscape 格式文件
3. 通过 `_merge_cookie_jars()` 合并两个来源的 Cookie（浏览器 Cookie 优先，文件 Cookie 补充）

### 2.2 Firefox Cookie 提取

[_extract_firefox_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L127-L200) 流程：

1. **定位数据库**：在搜索根目录下通过 glob 查找 `cookies.sqlite` 文件，取最新修改时间的那个（`_newest()`）
2. **处理容器**（Firefox Multi-Account Containers）：从 `containers.json` 中解析 `userContextId`
3. **复制数据库**：调用 [_open_database_copy()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1112-L1117) 将 SQLite 文件复制到临时目录（避免浏览器锁文件问题）
4. **读取与转换**：执行 SQL 查询 `SELECT host, name, value, path, expiry, isSecure FROM moz_cookies`，逐行构建 `http.cookiejar.Cookie` 对象并存入 jar
5. **Schema 版本适配**：FF142+ 的 schema version ≥ 16 将 expiry 从秒改为了毫秒，代码中做了 `expiry /= 1000` 的兼容

Firefox Cookie **无加密**，value 字段直接明文存储在 SQLite 中。

### 2.3 Chromium 系 Cookie 提取

[_extract_chrome_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L294-L369) 流程：

1. **定位数据库**：在 `browser_dir` 下递归搜索名为 `Cookies` 的文件
2. **复制数据库**：同 Firefox，复制到临时目录
3. **读取 meta_version**：从 `meta` 表读取 `version` 值，用于判断是否需要裁剪 hash 前缀（version ≥ 24）
4. **创建解密器**：调用 [get_cookie_decryptor()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L430-L435)，根据操作系统创建不同的解密器实例
5. **逐行处理**：SQL 查询 `SELECT host_key, name, value, encrypted_value, path, expires_utc, {secure_column} FROM cookies`，每行通过 [_process_chrome_cookie()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L372-L393) 判断是否加密并解密

#### 加密体系详解

Chromium 系 Cookie 的 `encrypted_value` 字段使用操作系统原生加密，解密策略因平台而异：

**Linux** — [LinuxChromeCookieDecryptor](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L438-L491)：

| 版本前缀 | 加密方式 | 密钥来源 |
|----------|---------|---------|
| `v10` | AES-CBC，固定盐 `saltysalt`，1 次 PBKDF2 迭代 | 密码 `peanuts` |
| `v11` | AES-CBC，同上 | 从 Linux 密钥环读取（KWallet/GNOME Keyring） |
| 其他 | 不支持 | - |

v10 和 v11 解密失败时会回退尝试空密码 `b''` 解密。`meta_version >= 24` 时解密结果需跳过前 32 字节 hash 前缀。

Linux 密钥环的选择逻辑在 [_choose_linux_keyring()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L845-L871) 中，先检测桌面环境，再映射到对应密钥环后端：

- KDE4 → KWALLET
- KDE5 → KWALLET5
- KDE6 → KWALLET6
- GNOME/Cinnamon/Deepin/Pantheon/Unity/XFCE/UKUI → GNOMEKEYRING
- KDE3/LXQT/OTHER → BASICTEXT（即 v10 固定密钥，无需密钥环密码）

**macOS** — [MacChromeCookieDecryptor](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L494-L525)：

| 版本前缀 | 加密方式 | 密钥来源 |
|----------|---------|---------|
| `v10` | AES-CBC，1003 次 PBKDF2 迭代 | macOS Keychain（`security find-generic-password`） |
| 其他 | 明文（"old data"） | - |

**Windows** — [WindowsChromeCookieDecryptor](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L528-L565)：

| 版本前缀 | 加密方式 | 密钥来源 |
|----------|---------|---------|
| `v10` | AES-GCM（96 位 nonce + 16 字节 auth tag） | `Local State` 文件中的 `os_crypt.encrypted_key`，经 DPAPI 解密 |
| 其他 | DPAPI 直接加密 | - |

Windows v10 密钥的获取在 [_get_windows_v10_key()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1013-L1037)：读取浏览器 `Local State` JSON 文件，取 `os_crypt.encrypted_key`，Base64 解码后去掉 `DPAPI` 前缀，调用 [_decrypt_windows_dpapi()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1073-L1105)（通过 `CryptUnprotectData` Win32 API）得到 AES-GCM 密钥。

### 2.4 Safari Cookie 提取

[_extract_safari_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L568-L591) → [parse_safari_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L723-L737)：

Safari 使用自定义的二进制格式 `Cookies.binarycookies`，结构为：

1. **文件头**：4 字节签名 `cook` + 页面数量 + 各页面大小
2. **页面**：每页包含多个 Cookie 记录
3. **记录**：包含 flags（secure 标志）、domain/name/path/value 的偏移量、expiration/creation 时间戳（Mac Absolute Time，需转换为 POSIX）

解析类 [DataParser](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L598-L646) 提供了按字节读取、校验、跳过等操作。时间戳通过 [_mac_absolute_time_to_posix()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L649-L650) 转换（基准时间 2001-01-01 UTC）。

### 2.5 Cookie 文件加载

[YoutubeDLCookieJar](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1276-L1420) 继承自 `http.cookiejar.MozillaCookieJar`，[load()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1354-L1403) 方法：

1. 逐行预处理：识别 `#HttpOnly_` 前缀并去除；跳过注释和空行；校验每行恰好 7 个 tab 分隔字段；校验 expires 格式
2. 若文件被误识别为 JSON 格式，抛出明确错误提示
3. 调用父类 `_really_load()` 解析 Netscape 格式
4. 将 `expires=0` 的 Cookie 标记为会话 Cookie（`discard=True`，`expires=None`），补齐 Python 标准库的缺陷

[save()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1333-L1352) 方法在下载完成后被调用，将 jar 中的 Cookie 写回文件（会话 Cookie 的 expires 写为 0）。

### 2.6 统一内存结构

所有来源的 Cookie 最终都转化为 `http.cookiejar.Cookie` 对象，存储在 `YoutubeDLCookieJar` 实例中。该 jar 同时提供了两个查询接口：

- [get_cookie_header(url)](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1405-L1409)：返回给定 URL 对应的 `Cookie` HTTP 头字符串
- [get_cookies_for_url(url)](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1411-L1416)：返回给定 URL 匹配的 `Cookie` 对象列表

---

## 三、请求侧使用

### 3.1 YoutubeDL 层的 Cookie 注入

[YoutubeDL._calc_headers()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/YoutubeDL.py#L2683-L2717) 是请求头计算的核心方法：

1. 合并全局 `http_headers` 和 `info_dict` 中的 `http_headers`
2. 如果是 `--load-info-json` 场景（`load_cookies=True`），先加载 Cookie 头和 Cookie 列表到 jar
3. **移除** `Cookie` 头（防止泄漏，参见安全公告 GHSA-v8mc-9377-rwjj）
4. 从 `cookiejar.get_cookies_for_url()` 获取匹配的 Cookie，编码后写入 `info_dict['cookies']`

这确保了 Cookie 不通过手动 Header 传递，而是完全由 jar 管理。

### 3.2 Extractor 层的 Cookie 操作

[InfoExtractor](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/extractor/common.py#L812-L814) 通过 `cookiejar` 属性访问全局 jar：

```python
@property
def cookiejar(self):
    return self._downloader.cookiejar
```

提供两个便捷方法：

- [_set_cookie()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/extractor/common.py#L3767-L3773)：向 jar 中添加指定 Cookie
- [_get_cookies(url)](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/extractor/common.py#L3775-L3777)：返回给定 URL 的 `LenientSimpleCookie` 对象

### 3.3 Networking 层的 Cookie 注入

[RequestHandler](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/networking/common.py#L223-L250) 在构造时接收 `cookiejar` 参数作为默认 jar。请求时通过 [_get_cookiejar(request)](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/networking/common.py#L283-L285) 获取实际使用的 jar——优先使用 request extension 中的 `cookiejar`，否则使用 handler 级别的默认 jar。

各 handler 的具体注入方式：

**Urllib** — [_urllib.py#L372-L399](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/networking/_urllib.py#L372-L399)：

在 `_create_instance()` 中创建 `HTTPCookieProcessor(cookiejar)` 并加入 `OpenerDirector`。每次请求时 `opener.open()` 会自动调用 `cookiejar.add_cookie_header()` 将 Cookie 写入请求头。

**Requests** — [_requests.py#L294-L307](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/networking/_requests.py#L294-L307)：

在 `_create_instance()` 中将 `session.cookies = cookiejar`。requests 库在发送请求时自动附带 session cookies。

**CurlCFFI** — [_curlcffi.py#L228-L229](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/networking/_curlcffi.py#L228-L229)：

在 `_create_instance()` 中将 `Session(cookies=cookiejar)`。注意在 [_send()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/networking/_curlcffi.py#L250-L253) 中，如果 request 已包含 `cookie` 头则不再注入 jar，避免重复。

**WebSockets** — [_websockets.py](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/networking/_websockets.py)：

通过 `cookiejar.get_cookie_header(request.url)` 获取 Cookie 头字符串，传给 WebSocket 连接。

### 3.4 Header Cookie 的安全处理

[YoutubeDL._load_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/YoutubeDL.py#L1799-L1839) 处理通过 HTTP Header 传入的 Cookie（兼容旧版用法），实现了域名作用域安全控制：

- 有 `domain` 属性的 Cookie：直接写入 jar
- 无 domain 且 `autoscope=True`：存入 `__header_cookies` 延迟列表，在提取时通过 [_apply_header_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/YoutubeDL.py#L1841-L1854) 将其作用域限定为当前 URL 的域名
- 无 domain 且 `autoscope` 为 URL：直接限定到该 URL 域名
- 无 domain 且 `autoscope=False`：报错，拒绝无作用域 Cookie

这一安全机制响应了 [GHSA-v8mc-9377-rwjj](https://github.com/yt-dlp/yt-dlp/security/advisories/GHSA-v8mc-9377-rwjj) 安全公告，防止 Cookie 被发送到非预期域名。

---

## 四、关键数据流总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                       来源识别 (Source)                              │
│                                                                     │
│  --cookies-from-browser chrome+kwallet:Profile1                     │
│       │                                                             │
│       ▼                                                             │
│  _parse_browser_specification() → (chrome, Profile1, kwallet, None)│
│       │                                                             │
│       ▼                                                             │
│  _get_chromium_based_browser_settings('chrome')                     │
│       → browser_dir, keyring_name, supports_profiles               │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       提取转换 (Extract)                             │
│                                                                     │
│  _extract_chrome_cookies()                                          │
│    ├─ 定位 Cookies 文件 → 复制到临时目录                              │
│    ├─ 读取 meta version → get_cookie_decryptor()                    │
│    │     └─ WindowsChromeCookieDecryptor                            │
│    │         └─ _get_windows_v10_key() → DPAPI → AES-GCM key       │
│    ├─ SELECT ... FROM cookies → 逐行 _process_chrome_cookie()       │
│    │     └─ decrypt(encrypted_value) → plaintext value              │
│    └─ 构建 http.cookiejar.Cookie → jar.set_cookie()                │
│                                                                     │
│  YoutubeDLCookieJar (合并浏览器 + 文件来源)                          │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       请求侧使用 (Request)                           │
│                                                                     │
│  YoutubeDL.cookiejar (cached_property)                              │
│       │                                                             │
│       ├── _calc_headers() → cookiejar.get_cookies_for_url()        │
│       │       → info_dict['cookies']                                │
│       │                                                             │
│       └── RequestHandler(cookiejar=...)                             │
│             ├── Urllib: HTTPCookieProcessor(cookiejar)              │
│             ├── Requests: session.cookies = cookiejar               │
│             ├── CurlCFFI: Session(cookies=cookiejar)                │
│             └── WebSockets: cookiejar.get_cookie_header(url)       │
│                                                                     │
│  → HTTP 请求自动附带匹配的 Cookie                                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、辅助工具类

| 类/函数 | 位置 | 用途 |
|---------|------|------|
| `LenientSimpleCookie` | [cookies.py#L1165](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1165) | 宽松版 `SimpleCookie`，容忍非标准 Cookie 格式 |
| `DataParser` | [cookies.py#L598](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L598) | Safari 二进制 Cookie 解析器 |
| `_LinuxDesktopEnvironment` | [cookies.py#L740](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L740) | Linux 桌面环境枚举 |
| `_LinuxKeyring` | [cookies.py#L760](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L760) | Linux 密钥环后端枚举 |
| `pbkdf2_sha1()` | [cookies.py#L1040](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1040) | PBKDF2-HMAC-SHA1 密钥派生 |
| `_decrypt_aes_cbc_multi()` | [cookies.py#L1044](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1044) | AES-CBC 多密钥尝试解密 |
| `_decrypt_aes_gcm()` | [cookies.py#L1057](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1057) | AES-GCM 解密 |
| `_decrypt_windows_dpapi()` | [cookies.py#L1073](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1073) | Windows DPAPI 解密 |
| `CookieLoadError` | [cookies.py#L89](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L89) | Cookie 加载失败异常 |
