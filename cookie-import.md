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

#### 属性转换边界

[_process_chrome_cookie()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L372-L393) 是 Chromium Cookie 属性转换的核心函数，将 SQLite 行数据逐字段映射为 `http.cookiejar.Cookie` 对象。各字段的转换边界如下：

**过期时间（expires_utc → expires）**：

Chromium 数据库中的 `expires_utc` 字段直接作为 `expires` 参数传入 Cookie 构造函数，仅做了零值判断：

```python
if not expires_utc:
    expires_utc = None
```

- **会话 Cookie**：`expires_utc == 0` 时转为 `None`，表示该 Cookie 没有持久过期时间，属于会话 Cookie
- **持久 Cookie**：`expires_utc` 为非零整数时直接使用其值
- **边界模糊**：`not expires_utc` 的判断覆盖了 `0`、`None`、空值等全部 falsy 值，这些都会被当作会话 Cookie 处理

> 注意：代码中**未对 `expires_utc` 做单位转换**。Chromium `cookies` 表的 `expires_utc` 为微秒级时间戳（基于 FILETIME epoch 或 Unix epoch 取决于具体版本），而 Python `http.cookiejar.Cookie` 的 `expires` 期望的是秒级 POSIX 时间戳。实际使用中 Chromium 版本差异可能导致过期时间量级偏差。

**域名（host_key → domain / domain_specified / domain_initial_dot）**：

```python
domain=host_key,
domain_specified=bool(host_key),
domain_initial_dot=host_key.startswith('.'),
```

- `domain`：直接使用 `host_key` 原值
- `domain_specified`：只要 `host_key` 非空就为 `True`（所有数据库中的 Cookie 都有域名）
- `domain_initial_dot`：检测 `host_key` 是否以 `.` 开头，用于判断是否匹配子域名

**路径（path → path / path_specified）**：

```python
path=path,
path_specified=bool(path),
```

- `path`：直接使用原值
- `path_specified`：只要路径非空就为 `True`

**安全标志（secure）**：

```python
secure=is_secure,
```

`is_secure` 的列名需要先探测——新旧版本 Chromium 分别使用 `secure` 和 `is_secure` 作为列名：

```python
secure_column = 'is_secure' if 'is_secure' in column_names else 'secure'
```

这是为了兼容 Chromium schema 变更而做的动态适配。

**discard 标志**：

代码中**硬编码为 `False`**：

```python
discard=False,
```

这意味着即使是会话 Cookie（`expires=None`），其 `discard` 标志也为 `False`。在 Python `http.cookiejar` 中，会话性由 `expires is None` 主导判断，`discard` 更多是 RFC 2965 语义的补充，因此该硬编码不影响功能，但与浏览器原生语义不完全一致。

**rest / 其他属性**：

`rest={}` 为空字典，表示 **HttpOnly、SameSite 等属性在提取时被丢弃**。Chromium 数据库中 `cookies` 表有 `is_httponly`、`samesite` 等字段，但代码的 SQL 查询没有选择这些字段，导致这些安全属性在导入后丢失。这是一个属性转换边界上的信息损失。

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

### 2.7 会话 Cookie 与持久 Cookie 的统一语义

各来源对会话/持久 Cookie 的表示方式不同，进入统一 jar 时需要对齐到 `http.cookiejar.Cookie` 的语义。以下为四个来源的对照：

| 来源 | 会话 Cookie 标识 | 持久 Cookie 标识 | `discard` 字段 |
|------|------------------|------------------|----------------|
| **Chromium 浏览器** | `expires_utc == 0` → `expires = None` | `expires_utc` 非零值 | 恒为 `False` |
| **Firefox 浏览器** | 数据库中 `expiry` 直接传入（未做零值特殊处理） | 同左 | 恒为 `False` |
| **Safari 浏览器** | 二进制格式中未做特殊区分，直接使用 `expiration_date` | 同左 | 恒为 `False` |
| **Netscape 文件** | `expires` 字段为空或 `0` → `expires = None`，`discard = True` | `expires` 为正整数时间戳 | 会话 Cookie 为 `True` |

#### Chromium 与会话 Cookie

Chromium 中会话 Cookie 的判断依据是 `expires_utc == 0`，由 [_process_chrome_cookie()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L384-L387) 转换：

```python
if not expires_utc:
    expires_utc = None
```

转换后 `expires=None`，在 `http.cookiejar` 语义中即表示「无过期时间」，等同于会话 Cookie。但 `discard` 字段硬编码为 `False`，与标准会话 Cookie 语义（关闭浏览器即丢弃）存在细微差异——实际使用中影响不大，因为 `CookieJar` 判断过期主要依赖 `expires` 字段。

#### Firefox 与会话 Cookie

Firefox 的 `moz_cookies` 表同样用 `expiry = 0` 表示会话 Cookie，但代码中**未显式处理零值**，而是直接把 `expiry` 原样传入 `expires` 参数。这意味着 Firefox 的会话 Cookie 进入 jar 后 `expires` 为 `0`，与标准语义（`None` 表示会话）不一致。

这个差异在文件加载阶段会被修正——Netscape 文件中 `expires=0` 会被转为 `expires=None`，但浏览器提取路径不会经过这个修正。不过 `http.cookiejar` 对 `expires=0` 的处理是将其视为 1970 年已过期，实际行为可能与预期不符。

#### 文件加载与会话 Cookie

Netscape 文件格式本身没有专门的会话 Cookie 标记，yt-dlp 通过约定 `expires = 0` 表示会话 Cookie。[YoutubeDLCookieJar.load()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1399-L1403) 中的处理：

```python
for cookie in self:
    if cookie.expires == 0:
        cookie.expires = None
        cookie.discard = True
```

文件加载路径同时设置了 `discard=True`，这比浏览器提取路径更规范。

#### 持久化回写时的对齐

[YoutubeDLCookieJar.save()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1345-L1348) 在写回文件前，会把所有 `expires is None` 的会话 Cookie 改为 `expires = 0`：

```python
for cookie in self:
    if cookie.expires is None:
        cookie.expires = 0
```

这样保证了「浏览器导入 → 写入文件 → 再次从文件加载」的往返过程中，会话 Cookie 的语义得以保留（虽然往返过程中 `discard` 会从 `False` 变为 `True`）。

#### 属性转换汇总表

| 属性 | Chromium 提取 | Firefox 提取 | Safari 提取 | Netscape 文件加载 |
|------|:---:|:---:|:---:|:---:|
| name | ✓ 原封传入 | ✓ 原封传入 | ✓ 原封传入 | ✓ 原封传入 |
| value | 解密后传入 | ✓ 原封传入 | ✓ 原封传入 | ✓ 原封传入 |
| domain | host_key 直接使用 | host 直接使用 | 原封传入 | domain_name 直接使用 |
| domain_specified | `bool(host_key)` | `bool(host)` | `bool(domain)` | 由父类设置 |
| domain_initial_dot | `host_key.startswith('.')` | `host.startswith('.')` | `domain.startswith('.')` | 由父类设置 |
| path | ✓ 原封传入 | ✓ 原封传入 | ✓ 原封传入 | ✓ 原封传入 |
| secure | is_secure / secure 列动态适配 | isSecure 列 | flags 位运算 | https_only 列 |
| expires | 0 → None，其余原样 | 原样传入（毫秒级需除以 1000） | Mac Absolute Time → POSIX | 0 → None，其余原样 |
| discard | 恒 False | 恒 False | 恒 False | 会话 Cookie 为 True |
| HttpOnly | ✗ 未读取 | ✗ 未读取 | ✗ 未读取 | ✓ 行前缀 `#HttpOnly_` |
| SameSite | ✗ 未读取 | ✗ 未读取 | ✗ 未读取 | ✗ 不支持 |

> **关键观察**：浏览器来源都不保留 HttpOnly 和 SameSite 属性，只有 Netscape 文件格式通过行前缀 `#HttpOnly_` 保留 HttpOnly 标记。这意味着从浏览器导入的 Cookie 在 yt-dlp 中**全部表现为非 HttpOnly**，这在安全属性上比浏览器原生环境更宽松。

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

---

## 六、来源冲突与优先级分析

### 6.1 两个来源可以同时存在

`--cookies` 和 `--cookies-from-browser` 两个参数互不排斥，可以同时指定。这在 [load_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L93-L113) 中体现为两个独立的 `if` 分支：

```python
def load_cookies(cookie_file, browser_specification, ydl):
    cookie_jars = []
    if browser_specification is not None:       # 分支1：浏览器来源
        ...
        cookie_jars.append(extract_cookies_from_browser(...))

    if cookie_file is not None:                  # 分支2：文件来源
        ...
        cookie_jars.append(jar)

    return _merge_cookie_jars(cookie_jars)
```

两者非互斥——当 `browser_specification` 和 `cookie_file` 都不为 `None` 时，两个 jar 都会进入合并列表。

### 6.2 合并策略：顺序追加，后者覆盖

[_merge_cookie_jars()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/cookies.py#L1141-L1148) 的实现：

```python
def _merge_cookie_jars(jars):
    output_jar = YoutubeDLCookieJar()
    for jar in jars:
        for cookie in jar:
            output_jar.set_cookie(cookie)
        if jar.filename is not None:
            output_jar.filename = jar.filename
    return output_jar
```

合并逻辑极其简洁：**按列表顺序遍历每个 jar，逐个调用 `output_jar.set_cookie(cookie)`**。关键行为由 `http.cookiejar.CookieJar.set_cookie()` 的语义决定：

- `set_cookie()` 对同一 `(domain, path, name)` 三元组的 Cookie 执行**替换**（后写入的覆盖先写入的）
- 对不同三元组的 Cookie 则直接追加

因此，**浏览器来源在列表中排第一（index 0），文件来源排第二（index 1）**，合并结果为：

| 场景 | 行为 |
|------|------|
| 浏览器和文件有不同 Cookie | 两者的 Cookie 全部保留，互不冲突 |
| 浏览器和文件有同名 Cookie（同 domain+path+name） | **文件来源覆盖浏览器来源** |

### 6.3 文件来源优先的实质

由于 `cookie_jars` 列表的构建顺序是 `[浏览器jar, 文件jar]`，文件来源的 Cookie 后写入，所以**文件来源的优先级高于浏览器来源**。

这意味着：

- 用户可通过 `--cookies` 文件**覆盖**从浏览器导入的特定 Cookie（例如修改某个 session token 的值）
- 未被覆盖的浏览器 Cookie 仍然生效
- 用户的文件只需包含想要覆盖的 Cookie，不必复制浏览器的全部 Cookie

### 6.4 filename 属性的继承

`_merge_cookie_jars()` 中还有一行特殊逻辑：

```python
if jar.filename is not None:
    output_jar.filename = jar.filename
```

由于列表顺序为 `[浏览器jar, 文件jar]`，而浏览器提取函数返回的 jar 的 `filename` 为 `None`（它们不关联文件），文件来源的 jar 的 `filename` 不为 `None`。因此 **合并后 jar 的 `filename` 继承自文件来源**。

这直接影响 [save_cookies()](file:///d:/fz/0601-2/solo-dogfeeding/code/89-yt-dlp/yt_dlp/YoutubeDL.py#L1050-L1052) 的行为——下载完成后 `cookiejar.save()` 会将合并后的全部 Cookie（包括浏览器来源的）写回 `--cookies` 指定的文件中。这实现了「浏览器 Cookie 持久化」：首次从浏览器导入后，后续运行可直接使用文件，无需再次访问浏览器数据库。

### 6.5 各场景的合并行为总结

| `--cookies-from-browser` | `--cookies` | 行为 |
|:---:|:---:|------|
| ✗ | ✗ | 空 jar，不加载任何 Cookie |
| ✓ | ✗ | 仅浏览器 Cookie，jar.filename 为 None，无法自动 save |
| ✗ | ✓ | 仅文件 Cookie，jar.filename 为文件路径，可自动 save |
| ✓ | ✓ | 浏览器 + 文件合并，同名 Cookie 以文件为准；jar.filename 为文件路径，save 时写入合并结果 |

### 6.6 潜在冲突场景与风险

**同名 Cookie 的静默覆盖**：当浏览器和文件中存在同 `(domain, path, name)` 的 Cookie 时，文件值静默替换浏览器值，无任何日志或警告。用户可能 unaware 浏览器中的 Cookie 被覆盖。

**Session Cookie vs 持久 Cookie 混淆**：浏览器提取时，Chromium 的 session Cookie（`expires_utc=0`）被转为 `expires=None`；文件加载时同样将 `expires=0` 标记为会话 Cookie。但如果文件中手动写了 `expires=0` 而浏览器中同名的持久 Cookie 已存在，覆盖后持久 Cookie 会变为会话 Cookie，行为可能不符合预期。

**域名匹配差异**：浏览器中 Cookie 的 domain 字段可能以 `.` 开头（如 `.youtube.com`），而 Netscape 格式文件中通过第二列 `TRUE/FALSE` 控制是否包含子域名。如果文件中的 domain 写法与浏览器提取结果不一致（如文件写 `youtube.com` 而浏览器为 `.youtube.com`），两者会被视为不同 Cookie 而同时存在，可能导致请求时发送重复的 Cookie 头。
