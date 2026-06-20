# yt-dlp 协议下载器分派机制完整分析

## 一、核心架构与类层次

### 1.1 类继承关系

```
FileDownloader [yt_dlp/downloader/common.py]
    ├── HttpFD [yt_dlp/downloader/http.py]                    # HTTP 直链下载
    │   └── HttpQuietDownloader [yt_dlp/downloader/fragment.py]
    ├── FragmentFD [yt_dlp/downloader/fragment.py]            # 分片下载基类
    │   ├── ExternalFD [yt_dlp/downloader/external.py]        # 外部下载器基类
    │   │   ├── CurlFD [yt_dlp/downloader/external.py]
    │   │   ├── AxelFD [yt_dlp/downloader/external.py]
    │   │   ├── WgetFD [yt_dlp/downloader/external.py]
    │   │   ├── Aria2cFD [yt_dlp/downloader/external.py]
    │   │   ├── HttpieFD [yt_dlp/downloader/external.py]
    │   │   └── FFmpegFD [yt_dlp/downloader/external.py]      # FFmpeg 下载/合并
    │   ├── HlsFD [yt_dlp/downloader/hls.py]                  # HLS 原生下载
    │   ├── DashSegmentsFD [yt_dlp/downloader/dash.py]        # DASH 分片下载
    │   ├── F4mFD [yt_dlp/downloader/f4m.py]
    │   └── IsmFD [yt_dlp/downloader/ism.py]
    └── RtmpFD [yt_dlp/downloader/rtmp.py]
```

### 1.2 关键文件定位

| 文件（仓库相对路径） | 核心作用 |
|---------------------|---------|
| `yt_dlp/downloader/__init__.py` | 分派入口 `get_suitable_downloader` |
| `yt_dlp/downloader/common.py` | 基类 `FileDownloader` |
| `yt_dlp/downloader/http.py` | `HttpFD` HTTP 下载 |
| `yt_dlp/downloader/hls.py` | `HlsFD` HLS 原生下载 |
| `yt_dlp/downloader/dash.py` | `DashSegmentsFD` DASH 下载 |
| `yt_dlp/downloader/fragment.py` | `FragmentFD` 分片基类 |
| `yt_dlp/downloader/external.py` | 外部下载器族（含 FFmpegFD） |
| `yt_dlp/utils/_utils.py` | `determine_protocol` 协议判定 |
| `yt_dlp/YoutubeDL.py` | 调用下载器的业务逻辑（路径拆分等） |

---

## 二、完整分派流程（三层分派）

### 2.1 第一层：协议判定 (`determine_protocol`)

位置：`yt_dlp/utils/_utils.py` L3178-L3197

```python
def determine_protocol(info_dict):
    protocol = info_dict.get('protocol')
    if protocol is not None:
        return protocol  # extractor 已指定协议时直接使用

    url = sanitize_url(info_dict['url'])
    if url.startswith('rtmp'): return 'rtmp'
    elif url.startswith('mms'): return 'mms'
    elif url.startswith('rtsp'): return 'rtsp'

    ext = determine_ext(url)
    if ext == 'm3u8':
        # 关键点：直播用 m3u8(FFmpeg)，非直播用 m3u8_native(原生)
        return 'm3u8' if info_dict.get('is_live') else 'm3u8_native'
    elif ext == 'f4m':
        return 'f4m'

    return urllib.parse.urlparse(url).scheme  # http/https/ftp 等
```

**协议与下载器映射表** (`PROTOCOL_MAP`)：
位置：`yt_dlp/downloader/__init__.py` L41-L61

| 协议 | 下载器（映射表默认） | 说明 |
|------|---------------------|------|
| `http` / `https` / `ftp` | `HttpFD` (全局默认兜底) | 普通 HTTP 下载 |
| `m3u8_native` | `HlsFD` | HLS 原生下载（非直播） |
| `m3u8` | `FFmpegFD` | HLS 通过 FFmpeg（直播默认） |
| `http_dash_segments` | `DashSegmentsFD` | DASH 分片下载 |
| `http_dash_segments_generator` | `DashSegmentsFD` | DASH 直播回放（`--live-from-start`） |
| `rtmp` | `RtmpFD` | RTMP 协议 |

> **注意**：`PROTOCOL_MAP` 只是**最后兜底**，前面还有 6+ 层条件会改写选择。

---

### 2.2 第二层：主分派 (`get_suitable_downloader`)

位置：`yt_dlp/downloader/__init__.py` L4-L20

```python
def get_suitable_downloader(info_dict, params={}, default=NO_DEFAULT, protocol=None, to_stdout=False):
    info_dict['protocol'] = determine_protocol(info_dict)
    info_copy = info_dict.copy()
    info_copy['to_stdout'] = to_stdout

    # 支持多协议组合（如 http_dash_segments+http_dash_segments 表示音视频分轨）
    protocols = (protocol or info_copy['protocol']).split('+')
    downloaders = [_get_suitable_downloader(info_copy, proto, params, default) for proto in protocols]

    # 特殊规则1：所有子协议都指向 FFmpeg 且支持直接合并 -> 直接返回 FFmpegFD
    if set(downloaders) == {FFmpegFD} and FFmpegFD.can_merge_formats(info_copy, params):
        return FFmpegFD
    # 特殊规则2：全部是 DASH 生成器（--live-from-start 回放）且非 stdout 多协议 -> 返回 DashSegmentsFD
    elif (set(downloaders) == {DashSegmentsFD}
          and not (to_stdout and len(protocols) > 1)
          and set(protocols) == {'http_dash_segments_generator'}):
        return DashSegmentsFD
    # 特殊规则3：**只有一个协议**（协议列表长度为 1）时直接返回对应下载器
    elif len(downloaders) == 1:
        return downloaders[0]
    # 其他所有多协议情况：返回 None，交给上层 YoutubeDL 做路径拆分
    return None
```

---

#### ⚠️ 关键澄清：多协议聚合判断的真实条件（修正之前的错误）

**之前错误结论**：
> 特殊规则3："只有一个下载器（单协议 / 多协议但都选了同一个）→ 直接返回"

**正确结论**：

`downloaders` 是一个**列表**，长度永远等于 `protocols` 长度（即协议数量），不是集合。

| 判断条件 | 含义 | 触发场景 |
|---------|------|---------|
| `len(downloaders) == 1` | **只有 1 个协议**（`protocol` 中无 `+`） | 单轨视频 / 单轨音频 / 纯字幕 |
| `set(downloaders) == {FFmpegFD}` + `can_merge_formats` | 所有子协议都选了 FFmpegFD + FFmpeg 可直接合并 | 多轨全是 m3u8 / 多轨全是 http+直播 DASH 等 |
| `set(downloaders) == {DashSegmentsFD}` + `set(protocols) == {'http_dash_segments_generator'}` | **所有子协议全是 `http_dash_segments_generator`** 且非 stdout 多协议 | 仅限 `--live-from-start` 的 DASH 直播回放多轨 |
| 其他所有多协议情况 | 返回 `None` | **普通 DASH 多轨 / DASH+HLS 混合 / HTTP+DASH 等** |

**多协议返回非 None 的情况只有 2.5 种**：

```
多协议 (len(protocols) >= 2)?
  │
  ├─> 所有下载器都是 FFmpegFD + can_merge_formats?
  │   └─> 是 → FFmpegFD（FFmpeg 多输入 -i 直接合并）
  │
  ├─> 所有下载器都是 DashSegmentsFD
  │   && 所有协议都是 http_dash_segments_generator（仅限 --live-from-start）
  │   && 非 stdout 多协议?
  │   └─> 是 → DashSegmentsFD（内部通过 requested_formats 处理多轨）
  │
  └─> 其他任何情况?
      └─> 返回 None → YoutubeDL 做路径拆分，逐轨道独立下载
```

**典型反例（之前文档错误，现已纠正）**：
- `http_dash_segments + http_dash_segments`（普通 DASH 音视频分轨）→ **返回 None**，不会统一 DashSegmentsFD！
  - `set(downloaders) == {DashSegmentsFD}` ✅
  - 但 `set(protocols) == {'http_dash_segments'}`，不是 `{'http_dash_segments_generator'}` ❌
  - 条件不满足 → 返回 `None` → YoutubeDL 走路径拆分

---

**`can_merge_formats` 条件**（FFmpegFD 直接合并）：
位置：`yt_dlp/downloader/external.py` L387-L393
- 有 `requested_formats`（多轨）
- 有 `protocol` 字段
- 未设置 `allow_unplayable_formats`
- 未设置 `no-direct-merge` 兼容选项
- `FFmpegFD.can_download(info_dict)` 为真

---

### 2.3 第三层：单协议分派 (`_get_suitable_downloader`)

位置：`yt_dlp/downloader/__init__.py` L87-L126

这是最核心的分派逻辑，按优先级依次判断：

```
开始
  │
  ▼
默认下载器 default = HttpFD (若调用方传 NO_DEFAULT)
  │
  ▼
① 有 section_start 或 section_end 且 FFmpeg 可用? ──是──> FFmpegFD
  │否            （分段下载必须由 FFmpeg 截取）
  ▼
② 用户指定了外部下载器 (--downloader)?
  │是
  ├─> 指定的是 'native'? ──是──> 跳过，继续往下走
  │否
  └─> 外部下载器 can_download(协议在 SUPPORTED_PROTOCOLS 内)? ──是──> 返回该外部下载器
  │否 (external_downloader is None)
  ▼
③ 输出到 stdout 且 FFmpeg 支持多格式合并? ──是──> FFmpegFD
  │否
  ▼
④ 协议 == 'http_dash_segments'?
  │是
  └─> is_live=True 且 external_downloader 不是 'native'? ──是──> FFmpegFD
  │否
  ▼
⑤ 协议是 'm3u8' 或 'm3u8_native'?
  │是
  ├─> is_live=True? ──是──> FFmpegFD
  ├─> external_downloader == 'native'? ──是──> HlsFD
  ├─> 协议是 m3u8_native 且 有支持 m3u8_frag_urls 的外部下载器? ──是──> HlsFD
  ├─> hls_prefer_native is True? ──是──> HlsFD
  └─> hls_prefer_native is False? ──是──> FFmpegFD
  │否
  ▼
⑥ 最后查 PROTOCOL_MAP，找不到返回 default (HttpFD)
```

**关键判断点解析**：

1. **分段下载优先 FFmpeg**（①）：只要设置了 `section_start` 或 `section_end`，无论什么协议都优先用 FFmpegFD，因为原生下载器不支持时间轴截取。

2. **外部下载器介入点**（②）：用户通过 `--downloader` 指定外部下载器时，在检查完分段需求后立即判断。但需要 `impersonate`（浏览器伪装）为 None（外部命令行工具无法模拟浏览器指纹）。

3. **HLS 的两条路径**（⑤）：
   - `m3u8` 协议 → 默认走 FFmpegFD（直播流，`determine_protocol` 中已做区分）
   - `m3u8_native` 协议 → 走 HlsFD 原生下载，但会被 `hls_prefer_native=False` 条件改写为 FFmpegFD

4. **HLS 直播强制 FFmpeg**（⑤-1）：HLS 直播流 (`is_live=True`) 始终使用 FFmpegFD，原生下载器 HlsFD 明确不支持直播。

5. **DASH 直播走 FFmpeg**（④）：`http_dash_segments` + `is_live=True` + 非强制 native → FFmpegFD（DashSegmentsFD 内部还会对直播报错兜底）。

6. **片段 URL 协议检测**（⑤-3）：通过 `m3u8_frag_urls` 和 `dash_frag_urls` 伪协议检测外部下载器是否支持接管 HLS/DASH 的片段下载（目前无内置实现，是扩展点）。

---

## 三、HTTP、HLS、DASH 三者选择关系

### 3.1 HTTP 下载 (HttpFD)

**适用场景**：
- 普通直链下载（http/https/ftp）
- HLS/DASH 原生分片时，内部下载每个独立片段

**分派路径**：
1. `determine_protocol` 返回 `http`/`https`/`ftp`（URL scheme）
2. `_get_suitable_downloader` 中 ①~⑤ 层条件均不匹配
3. `PROTOCOL_MAP` 中没有这些协议的映射，返回兜底默认 `HttpFD`

**关键特性**：
- 支持断点续传（通过 `Range` 头 + `Content-Range` 校验）
- 支持分块下载（`http_chunk_size`，绕带宽限速）
- 支持速率限制（`ratelimit`）
- 支持节流检测（`throttledratelimit`，3 秒持续低速则抛错重试）
- 完整的重试机制（`RetryManager`）

**代码位置**：`yt_dlp/downloader/http.py` L23-L376

---

### 3.2 HLS 下载 (HlsFD vs FFmpegFD)

#### 3.2.1 HlsFD（原生下载）

**适用场景**：
- 非直播 HLS (`m3u8_native` 协议)
- 无 DRM 保护（FairPlay/PlayReady/Flash Access）
- 加密方式仅为 NONE 或 AES-128（**注意**：pycryptodomex 不可用**不一定**降级，见下方详细分析）
- 用户设置 `--downloader native` 或 `--hls-prefer-native`

**分派路径**：
```
protocol = m3u8_native
  │
  ├─> ① 分段截取需求? → 否
  ├─> ② 外部下载器 != native (或未设置)
  ├─> ③ stdout 合并? → 否
  ├─> ⑤ is_live? → 否
  ├─> ⑤ external_downloader='native'? → 可选条件
  ├─> ⑤ 有支持 m3u8_frag_urls 的外部下载器? → 可选条件
  └─> ⑤ hls_prefer_native=True? → 可选条件
  │
  ▼
返回 HlsFD
```

**内部再分派**（HlsFD.real_download）：
位置：`yt_dlp/downloader/hls.py` L74-L128

1. 下载并解析 m3u8 manifest（或使用 `hls_media_playlist_data` 中缓存的）
2. `can_download` 能力检查：
   - 不是直播（双重校验）
   - 无 DRM 标签
   - 加密方式仅为 NONE 或 AES-128
3. **AES-128 + pycryptodomex 缺失的精细处理**（⚠️ 之前说法不准确，详见下方 3.2.3 节）
4. **其他不支持情况时降级**：直接创建 `FFmpegFD` 实例并调用其 `real_download`
5. 尝试寻找支持 `m3u8_frag_urls` 协议的外部下载器接管片段批量下载
6. 无外部下载器时，使用 `FragmentFD` 机制，内部用 `HttpFD` 逐片段下载

**片段下载与解密机制**：
- `HlsFD` 继承自 `FragmentFD`
- `FragmentFD._download_fragment` 创建 `HttpQuietDownloader`（即 `HttpFD`，屏蔽输出）下载每个片段
- 下载后若有 AES-128 加密，通过 `aes_cbc_decrypt_bytes` 解密（`yt_dlp/aes.py`）：
  - Cryptodome.AES 可用时 → pycryptodomex 快速解密
  - Cryptodome.AES 不可用时 → 纯 Python 原生 AES 实现（极慢，但功能可用）
- 按顺序追加到目标文件（WebVTT 字幕还要特殊打包）

---

#### 3.2.3 AES-128 加密 + pycryptodomex 缺失的真实降级逻辑（⚠️ 纠正：不一定降级）

**之前错误结论**：
> "AES-128 + 无 pycryptodomex：若 ffmpeg 可用也降级到 FFmpegFD"

**正确结论**：pycryptodomex 缺失时**有 4 种组合**，并非一定降级。

**判断代码**（`yt_dlp/downloader/hls.py` L94-L109）：
```python
can_download, message = self.can_download(s, info_dict, self.params.get('allow_unplayable_formats')), None
if can_download:
    has_ffmpeg = FFmpegFD.available()
    if not Cryptodome.AES and '#EXT-X-KEY:METHOD=AES-128' in s:
        # Even if pycryptodomex isn't available, force HlsFD for m3u8s that won't work with ffmpeg
        ffmpeg_can_dl = not traverse_obj(info_dict, ((
            'extra_param_to_segment_url', 'extra_param_to_key_url',
            'hls_media_playlist_data', ('hls_aes', ('uri', 'key', 'iv')),
        ), any))
        message = 'The stream has AES-128 encryption and {} available'.format(
            'neither ffmpeg nor pycryptodomex are' if ffmpeg_can_dl and not has_ffmpeg else
            'pycryptodomex is not')
        if has_ffmpeg and ffmpeg_can_dl:
            can_download = False          # 只有这种情况才降级
        else:
            message += '; decryption will be performed natively, but will be extremely slow'
```

**四种组合的真实结果**：

| Cryptodome.AES | `FFmpegFD.available()` | `ffmpeg_can_dl`（无特殊参数） | 结果 | 说明 |
|---|---|---|---|---|
| ✅ 可用 | 任意 | 任意 | ✅ HlsFD 原生下载，Cryptodome 快速解密 | 最佳路径 |
| ❌ 不可用 | ✅ 可用 | ✅ 可处理（无 extra_param/hls_aes） | ❌ `can_download = False` → 降级 FFmpegFD | 唯一降级情况 |
| ❌ 不可用 | ✅ 可用 | ❌ 不可处理（有 extra_param/hls_aes 等）| ⚠️ HlsFD 继续，纯 Python 解密（极慢） | FFmpeg 无法处理这些特殊参数 |
| ❌ 不可用 | ❌ 不可用 | ✅/❌ | ⚠️ HlsFD 继续，纯 Python 解密（极慢） | 无 FFmpeg 可用，只能硬解 |

**`ffmpeg_can_dl = False` 的场景**（FFmpeg 无法处理，不降级）：
- `extra_param_to_segment_url`：片段 URL 需要额外 query 参数
- `extra_param_to_key_url`：密钥 URL 需要额外 query 参数
- `hls_media_playlist_data`：manifest 数据内嵌在 info_dict 中
- `hls_aes.uri` / `hls_aes.key` / `hls_aes.iv`：AES 密钥/IV 由 extractor 直接提供

**纯 Python AES 实现的位置**：`yt_dlp/aes.py` L17-L19
```python
else:
    def aes_cbc_decrypt_bytes(data, key, iv):
        """ Decrypt bytes with AES-CBC using native implementation since pycryptodome is unavailable """
        return bytes(aes_cbc_decrypt(*map(list, (data, key, iv))))
```
> 此实现不依赖任何外部库，但性能远低于 Cryptodome（每片段约几十毫秒 vs 几微秒），长视频会显著变慢。

---

#### 3.2.4 FFmpegFD（FFmpeg 处理 HLS）

**适用场景**：
- HLS 直播流 (`is_live=True`)
- 有 DRM 保护
- 不支持的加密方式（非 AES-128）
- 用户设置 `--hls-prefer-native=False` 或 `--downloader ffmpeg`
- 原生下载器 `can_download` 检查失败后降级
- 多格式合并且全部返回 FFmpegFD

**分派路径**：
```
protocol = m3u8 或 m3u8_native
  │
  ├─> ① 分段截取 + FFmpeg 可用? → 是 → FFmpegFD
  ├─> ② --downloader ffmpeg + 支持? → 是 → FFmpegFD
  ├─> ⑤ is_live=True? → 是 → FFmpegFD
  ├─> ⑤ hls_prefer_native=False? → 是 → FFmpegFD
  └─> HlsFD.can_download 失败 → FFmpegFD 兜底
```

**FFmpegFD 支持的协议全集**：
`('http', 'https', 'ftp', 'ftps', 'm3u8', 'm3u8_native', 'rtsp', 'rtmp', 'rtmp_ffmpeg', 'mms', 'http_dash_segments')`

---

### 3.3 DASH 下载（DashSegmentsFD vs FFmpegFD —— **纠正之前错误结论**）

> **重要修正**：DASH **并非始终**走 DashSegmentsFD 原生分片。至少有 **5 条路径** 会交给 FFmpegFD 处理。
> **此外，普通 DASH 多轨（非 generator）也不会统一走 DashSegmentsFD，而是返回 None 走路径拆分。**

#### 3.3.1 FFmpeg 处理 DASH 的 5 种情况

| # | 触发条件 | 判断位置 |
|---|---------|---------|
| 1 | 有 `section_start`/`section_end` 分段截取需求 + FFmpeg 可用 | `_get_suitable_downloader` ① |
| 2 | 用户指定 `--downloader ffmpeg`（或其他支持 DASH 的外部下载器） | `_get_suitable_downloader` ② + `FFmpegFD.SUPPORTED_PROTOCOLS` 含 `http_dash_segments` |
| 3 | `is_live=True` + 非强制 `native` | `_get_suitable_downloader` ④ |
| 4 | 多协议（音视频分轨）都返回 FFmpegFD + `can_merge_formats` 满足 | `get_suitable_downloader` 特殊规则 1 |
| 5 | 输出到 stdout + 多格式可合并 | `_get_suitable_downloader` ③ |

---

#### 3.3.2 直播 DASH 走 FFmpeg 的真实条件（⚠️ 纠正：强制 native 时只报错，不兜底 FFmpeg）

**之前错误结论**：
> "DASH 直播有三道关卡确保走 FFmpeg（分派层、DashSegmentsFD 内部二次报错、FFmpeg `-re` 参数适配）"

**正确结论**：DASH 直播走 FFmpeg **只有一道硬关卡**（分派层），且可以被 `--downloader native` 绕过。绕过之后 DashSegmentsFD **只报错，不降级 FFmpeg**。

---

**分派层判断代码**（`yt_dlp/downloader/__init__.py` L109-L111）：
```python
if protocol == 'http_dash_segments':
    if info_dict.get('is_live') and (external_downloader or '').lower() != 'native':
        return FFmpegFD
```

**三个必要条件缺一不可**：
1. `protocol == 'http_dash_segments'`（**注意**：`http_dash_segments_generator` 走 DashSegmentsFD，不走这条）
2. `is_live == True`（由 extractor 设置，不是从 URL 推断）
3. 用户**没有**强制指定 `--downloader native`（即 `external_downloader` 是 None / ffmpeg / aria2c 等非 native 值）

**当条件 3 不满足（用户强制 native）时**：
- 分派层条件不成立，继续往下 → 查 `PROTOCOL_MAP` → 返回 `DashSegmentsFD`
- 此时进入 DashSegmentsFD.real_download

**DashSegmentsFD 直播处理代码**（`yt_dlp/downloader/dash.py` L21-L22）：
```python
if info_dict.get('is_live'):
    self.report_error('Live DASH videos are not supported')
```

⚠️ **关键发现**：这里只调用了 `report_error`，**没有创建 FFmpegFD 兜底，没有降级逻辑**。执行完 `report_error` 后函数继续往下执行，但后续 `fragments` 解析会失败或下载异常，最终返回 `False`（下载失败）。

**FFmpeg 侧直播 DASH 特殊处理**（`yt_dlp/downloader/external.py` L484-L489）：
```python
elif protocol == 'http_dash_segments' and info_dict.get('is_live'):
    # 直播 DASH 加 -re 防止 ffmpeg 读超出最新可用分片
    args += ['-re']  # 别名 -readrate 1，但兼容旧版 ffmpeg
```
> 这段只有在分派层已选择 FFmpegFD 时才会执行。

---

**DASH 直播完整决策矩阵**：

| 用户参数 | 分派层结果 | DashSegmentsFD 内部 | 最终结果 |
|---------|-----------|-------------------|---------|
| 默认（无 `--downloader`） | 返回 FFmpegFD ✅ | 不执行 | FFmpeg 下载（加 `-re`） |
| `--downloader ffmpeg` | 返回 FFmpegFD ✅ | 不执行 | FFmpeg 下载（加 `-re`） |
| `--downloader aria2c` 等 | 先返回 Aria2cFD（若支持 DASH）否则回退 → 见下方 | - | - |
| `--downloader native` ⚠️ | 返回 **DashSegmentsFD** ❌ | `report_error` + 无降级 | **下载失败报错**，不走 FFmpeg |

> **修正结论**：DASH 直播**没有三道关卡**。分派层是唯一走 FFmpeg 的路径，但 `--downloader native` 可以明确绕过。DashSegmentsFD 内部只有报错，没有任何兜底逻辑确保走 FFmpeg。用户若误传 `--downloader native`，DASH 直播会直接失败退出。

**补充：外部下载器（非 native/非 ffmpeg）处理 DASH 直播**：
- `--downloader aria2c`：进入 `_get_suitable_downloader` L104-L107，`external_downloader != 'native'`，先检测 Aria2cFD 的 `can_download`
- Aria2cFD `SUPPORTED_PROTOCOLS` 是 `('http', 'https', 'ftp', 'ftps')`，**不含** `http_dash_segments` → `can_download` 为假
- 继续往下 → 命中 L109-L111 的 DASH 直播条件（因为 `external_downloader='aria2c' != 'native'`）→ 返回 FFmpegFD
- 所以：外部下载器（除了 native）即使不支持 DASH，也会被分派层"矫正"回 FFmpegFD。**只有 `'native'` 这个字符串会绕过分派层的 DASH 直播检测**。

---

#### 3.3.3 DashSegmentsFD（原生分片下载）—— 单轨 & DASH 直播回放多轨

**分派路径**（满足以下全部条件）：

```
场景 A：单轨 DASH
─────────────────
protocol = http_dash_segments / http_dash_segments_generator（单协议，len==1）
  │
  ├─> ① 无分段截取需求
  ├─> ② 未指定支持 DASH 的外部下载器（或指定了 native）
  ├─> ③ 不是 stdout 多格式合并
  ├─> ④ 非 is_live（或强制 native，后者进入 DashSegmentsFD 后 report_error 报错退出，不兜底 FFmpeg）
  │
  ▼
len(downloaders) == 1 → 返回 PROTOCOL_MAP 中的 DashSegmentsFD
```

```
场景 B：DASH 直播回放多轨（--live-from-start）
──────────────────────────────────────────────
protocol = http_dash_segments_generator + http_dash_segments_generator + ...
  │
  ├─> set(downloaders) == {DashSegmentsFD} ✅
  ├─> set(protocols) == {'http_dash_segments_generator'} ✅
  ├─> 非 (to_stdout && len(protocols) > 1) ✅
  │
  ▼
特殊规则 2 → 返回 DashSegmentsFD（唯一能多协议统一的非 FFmpeg 场景）
```

> **⚠️ 重要反例（纠正之前错误）**：
> - `protocol = http_dash_segments + http_dash_segments`（普通 DASH 音视频分轨）
>   - `set(protocols) == {'http_dash_segments'}`，**不是** `{'http_dash_segments_generator'}`
>   - 不满足特殊规则 2
>   - `len(downloaders) == 2 ≠ 1`，不满足特殊规则 3
>   - → **返回 `None` → 走 YoutubeDL 路径拆分**，逐轨道独立下载！

**适用场景**：
- 非直播 DASH 分片视频（`http_dash_segments`，单轨）
- DASH 直播回放（`http_dash_segments_generator`，即 `--live-from-start`，单轨或多轨）

**内部多轨处理**（DashSegmentsFD.real_download）：
位置：`yt_dlp/downloader/dash.py` L17-L68

```python
# DashSegmentsFD 内部本身支持多轨（通过 requested_formats）
requested_formats = [{**info_dict, **fmt} for fmt in info_dict.get('requested_formats', [])]
args = []
for fmt in requested_formats or [info_dict]:
    ...
    args.append([ctx, fragments_to_download, fmt])
# 同时下载多个轨道的片段
return self.download_and_append_fragments_multiple(*args, is_fatal=lambda idx: idx == 0)
```

但这段多轨逻辑**只有在 DashSegmentsFD 被调用时才会执行**，而多轨场景下它被调用的**唯一机会**是所有协议都是 `http_dash_segments_generator`（`--live-from-start` 回放）。

**内部再分派逻辑**：
1. 如果是 `http_dash_segments_generator`（直播回放/`--live-from-start`），明确不使用外部下载器
2. 普通 DASH：尝试寻找支持 `dash_frag_urls` 的外部下载器（目前内置均不支持）
3. 找到 → 将所有片段 URL 列表放入 `info_dict['fragments']`，委托外部下载器批量下载
4. 未找到 → 使用 `FragmentFD` 机制，内部用 `HttpFD` 逐片段下载

---

## 四、多格式下载器无法统一时的路径拆分方法

当 `get_suitable_downloader` 返回 `None` 时，由 `YoutubeDL.process_info` 中的下载逻辑做**路径拆分**。

### 4.1 触发 `None` 的完整条件清单

`get_suitable_downloader` 返回 `None` **当且仅当全部以下条件同时成立**：

1. `len(protocols) >= 2`（协议是 `A+B+C...` 的多协议组合，即多轨视频）
2. **不满足** FFmpeg 直接合并：
   - 要么 `set(downloaders) ≠ {FFmpegFD}`（轨道间下载器不同）
   - 要么 `FFmpegFD.can_merge_formats(...)` 为假
3. **不满足** DASH 回放统一：
   - 要么 `set(downloaders) ≠ {DashSegmentsFD}`
   - 要么 `set(protocols) ≠ {'http_dash_segments_generator'}`
   - 要么是 `to_stdout` 多协议场景

**典型触发场景**：

| 场景 | protocols | 下载器集合 | 命中哪条 None 条件 |
|------|-----------|-----------|-------------------|
| 普通 DASH 音视频分轨 | `http_dash_segments+http_dash_segments` | `{DashSegmentsFD}` | 协议不是 generator，不满足特殊规则 2 |
| DASH 视频 + HTTP 音频 | `http_dash_segments+https` | `{DashSegmentsFD, HttpFD}` | 下载器不一致 |
| HLS 视频 + DASH 音频 | `m3u8_native+http_dash_segments` | `{HlsFD, DashSegmentsFD}` | 下载器不一致 |
| HLS 视频 + HTTP 音频 | `m3u8_native+https` | `{HlsFD, HttpFD}` | 下载器不一致 |
| 多 HTTP 直链分轨（罕见） | `https+https` | `{HttpFD}` | 非 FFmpeg，非 Dash generator，无特殊规则 |

---

### 4.2 路径拆分算法（YoutubeDL.process_info）

位置：`yt_dlp/YoutubeDL.py` L3482-L3563

```
有 requested_formats（多格式）?
  │
  ├─> get_suitable_downloader 返回了 fd（非 None）?
  │   │是
  │   ├─> fd == FFmpegFD? ──是──> 直接交给 FFmpeg 多输入合并下载
  │   │       （FFmpegFD._call_downloader 内部遍历 requested_formats，逐个 -i url）
  │   │否
  │   └─> 为每个 format 预分配 f{format_id} 前缀的独立文件路径
  │       然后把所有 url 用 \n 拼接（供外部下载器如 aria2c 使用）
  │       然后调用 fd.download 一次（只有 DashSegmentsFD 多轨回放会到这里）
  │
  └─> fd is None（下载器无法统一）?
      │
      ├─> allow_unplayable_formats? ──是──> 警告：不合并防数据损坏
      │
      ├─> FFmpeg 不可用? ──是──> 警告：不合并
      │
      └─> 核心拆分逻辑：遍历每个 requested_format
          │
          ├─> 对每个 format f：
          │   ├─> new_info = dict(info_dict)
          │   ├─> del new_info['requested_formats']  # 剥离多格式信息
          │   ├─> new_info.update(f)                   # 填入单轨道信息（含独立 protocol/url/fragments）
          │   ├─> 生成独立文件名: f{format_id}.{ext}
          │   └─> 递归调用 self.dl(fname, new_info)
          │           └─> 内部重新调用 get_suitable_downloader 为单轨道选下载器
          │               （此时 protocol 只有 1 个，len==1，一定返回非 None）
          │
          └─> 所有轨道下载完成后 → 交给 FFmpegMergerPP 后处理合并
```

### 4.3 关键代码片段

**统一分支（fd 非 None，且非 FFmpegFD）**（`yt_dlp/YoutubeDL.py` L3518-L3527）：
```python
elif fd:
    if fd != FFmpegFD and temp_filename != '-':
        # 为每个格式分配独立文件名（DashSegmentsFD 内部会用 fmt['filepath']）
        for f in info_dict['requested_formats']:
            f['filepath'] = fname = prepend_extension(
                correct_ext(temp_filename, info_dict['ext']),
                'f{}'.format(f['format_id']), info_dict['ext'])
            downloaded.append(fname)
    # 把所有 URL 用 \n 拼接（HttpFD/CurlFD/Aria2cFD 等只看 info_dict['url']）
    info_dict['url'] = '\n'.join(f['url'] for f in info_dict['requested_formats'])
    success, real_download = self.dl(temp_filename, info_dict)  # 一次性调用
```

> **注意**：这里的 `\n` 拼接 URL 是给**单 URL 下载器**（如 Aria2cFD）设计的。但实际上 HttpFD、CurlFD、Aria2cFD、AxelFD、WgetFD、HttpieFD 的 `_make_cmd` 都只读取 `info_dict['url']` 作为**单个字符串**，不会按 `\n` 拆分。真正的多轨统一处理只在：
> - **FFmpegFD**：遍历 `requested_formats`，每个 fmt 加一个 `-i` 参数
> - **DashSegmentsFD**：遍历 `requested_formats`，内部 `download_and_append_fragments_multiple` 多轨并行

**路径拆分分支（fd 为 None）**（`yt_dlp/YoutubeDL.py` L3549-L3563）：
```python
for f in info_dict['requested_formats']:
    new_info = dict(info_dict)
    del new_info['requested_formats']   # 去掉多轨标记
    new_info.update(f)                  # 注入单轨 info（含独立 protocol/url/fragments）
    if temp_filename != '-':
        fname = prepend_extension(
            correct_ext(temp_filename, new_info['ext']),
            'f{}'.format(f['format_id']), new_info['ext'])
        if not self._ensure_dir_exists(fname):
            return
        f['filepath'] = fname
        downloaded.append(fname)
    # 递归调用 dl：此时 new_info 的 protocol 只有 1 个，get_suitable_downloader 一定返回非 None
    partial_success, real_download = self.dl(fname, new_info)
    info_dict['__real_download'] = info_dict['__real_download'] or real_download
    success = success and partial_success
```

**FFmpeg 直接合并 vs 后处理合并的区别**：
| 方式 | 触发条件 | 执行流程 | 中间文件 | 适用下载器 |
|------|---------|---------|---------|-----------|
| FFmpegFD 直接合并 | 所有轨道都选了 FFmpegFD + `can_merge_formats` | FFmpeg 多 `-i` 输入同时读取，输出单文件 | 无（或临时 ffmpeg 管道） | 仅 FFmpegFD |
| DashSegmentsFD 多轨统一 | 所有协议都是 `http_dash_segments_generator` + 非 stdout 多轨 | DashSegmentsFD 内部多轨并行下载片段 | f{id}.ext 多文件 | 仅 DashSegmentsFD（`--live-from-start`） |
| 后处理合并（FFmpegMergerPP） | 下载器无法统一 / 普通 DASH 多轨 / 其他多协议组合 | 各轨道独立下载完成 → FFmpeg 后合并 | 多个 `f{id}.ext` 文件 | HttpFD / HlsFD / DashSegmentsFD 等任意 |

---

## 五、特殊伪协议：`m3u8_frag_urls` 与 `dash_frag_urls`

这两个是**伪协议**（不出现在 `PROTOCOL_MAP` 中），仅用于检测外部下载器是否支持接管 HLS/DASH 的**批量片段下载**。

### 5.1 检测机制

在 `_get_suitable_downloader` 中（`yt_dlp/downloader/__init__.py` L118-L120）：
```python
elif protocol == 'm3u8_native' and get_suitable_downloader(
        info_dict, params, None, protocol='m3u8_frag_urls', to_stdout=info_dict['to_stdout']):
    return HlsFD
```

**关键点**：
- 传入 `default=None`（而非 `HttpFD` 默认兜底）
- 递归调用 `get_suitable_downloader` 查询 `m3u8_frag_urls` 协议
- 如果**没有任何**外部下载器的 `SUPPORTED_PROTOCOLS` 包含该伪协议，返回 `None`，条件不成立
- 返回 HlsFD 的意义：确认 HlsFD 会用 FragmentFD 机制，且存在外部下载器可接管片段

### 5.2 DashSegmentsFD 中的伪协议查询

位置：`yt_dlp/downloader/dash.py` L23-L24
```python
real_downloader = get_suitable_downloader(
    info_dict, self.params, None, protocol='dash_frag_urls', to_stdout=(filename == '-'))
```

同样传 `default=None`。若返回外部下载器类，则：
1. `DashSegmentsFD` 将所有片段解析为 URL 列表
2. 放入 `info_dict['fragments']`
3. 创建该外部下载器实例，委托其 `real_download` 批量下载

### 5.3 外部下载器支持现状

目前内置**没有任何外部下载器**在 `SUPPORTED_PROTOCOLS` 中包含这两个伪协议：
| 类 | SUPPORTED_PROTOCOLS | 支持伪协议? |
|----|-------------------|------------|
| `ExternalFD`（基类默认） | `('http', 'https', 'ftp', 'ftps')` | ❌ |
| `Aria2cFD` | `('http', 'https', 'ftp', 'ftps')` | ❌ |
| `FFmpegFD` | `('http', 'https', ..., 'http_dash_segments')` | ❌（直接支持 manifest 级协议，不走伪协议） |

**设计意图**：这是为**第三方插件**预留的扩展点。理论上一个支持批量 URL 列表输入的外部下载器（如 aria2c 的 `--input-file`）可以通过插件添加伪协议支持，从而避免 yt-dlp 逐片段串行下载的开销。

---

## 六、YoutubeDL.py 中的调用点

### 6.1 单轨下载入口 (`dl` 方法)

位置：`yt_dlp/YoutubeDL.py` L3283-L3318

```python
def dl(self, name, info, subtitle=False, test=False):
    # ... 处理 test 参数
    fd = get_suitable_downloader(info, params, to_stdout=(name == '-'))(self, params)
    # ... progress_hook 注册
    new_info = self._copy_infodict(info)
    if new_info.get('http_headers') is None:
        new_info['http_headers'] = self._calc_headers(new_info)
    return fd.download(name, new_info, subtitle)
```

> **注意**：`dl` 方法假设 `get_suitable_downloader` 永远返回非 `None` 值。若返回 `None` 会在调用时抛 `TypeError`。因此 `None` 情况必须在上层 `process_info` 中（见第四节）处理完才会调用 `dl`。
>
> 路径拆分后逐轨调用 `self.dl(fname, new_info)` 时，`new_info` 已经被剥离了 `requested_formats` 并注入了单轨 info，此时 `protocol` 只有一个，`len(downloaders) == 1` 一定成立，所以一定返回非 None。

### 6.2 多轨下载 + 路径拆分 (`process_info` 中)

位置：`yt_dlp/YoutubeDL.py` L3472-L3563

核心流程（与第四节对应）：
1. 先调用一次 `get_suitable_downloader` 尝试找到统一下载器
2. 非 FFmpegFD + 分段截取需求 → 报错（分段必须用 FFmpeg）
3. `requested_formats` 存在时：
   - fd 存在（下载器统一）→ 一次性调用（FFmpegFD 直接合并或 DashSegmentsFD 多轨回放）
   - fd 为 `None`（下载器不统一 / 普通 DASH 多轨）→ **逐轨拆分下载**，FFmpegMergerPP 后合并
4. 无 `requested_formats` → 普通单文件下载

### 6.3 合并格式时的 FFmpeg 强制校验

位置：`yt_dlp/YoutubeDL.py` L3475-L3480

```python
if fd != FFmpegFD and 'no-direct-merge' not in self.params['compat_opts'] and (
        info_dict.get('section_start') or info_dict.get('section_end')):
    msg = ('This format cannot be partially downloaded' if FFmpegFD.available()
           else 'You have requested downloading the video partially, but ffmpeg is not installed')
    self.report_error(f'{msg}. Aborting')
    return
```

### 6.4 下载器名称记录（用于调试/输出）

位置：`yt_dlp/YoutubeDL.py` L3630-L3631

```python
downloader = get_suitable_downloader(info_dict, self.params) if 'protocol' in info_dict else None
downloader = downloader.FD_NAME if downloader else None
```

---

## 七、完整调用链示例

### 7.1 普通 HTTP 视频下载

```
YoutubeDL.process_info
  └─> YoutubeDL.dl(temp_filename, info)
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'https'
          └─> _get_suitable_downloader(info, 'https', ...)
              ├─> ①~⑤ 均不匹配
              └─> PROTOCOL_MAP 无 'https' → 默认 HttpFD
      └─> HttpFD.download(filename, info)
          └─> HttpFD.real_download → 直接 HTTP 下载 + 断点续传 + 限流
```

### 7.2 非直播 HLS 视频（原生下载）

```
YoutubeDL.process_info
  └─> YoutubeDL.dl
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'm3u8_native'
          └─> _get_suitable_downloader(info, 'm3u8_native', ...)
              ├─> is_live? → 否
              ├─> 无外部下载器
              └─> PROTOCOL_MAP['m3u8_native'] → HlsFD
      └─> HlsFD.download(filename, info)
          └─> HlsFD.real_download
              ├─> 下载 m3u8 manifest
              ├─> can_download 检查（DRM/AES/直播）→ 通过
              ├─> 尝试 m3u8_frag_urls 外部下载器 → 无（传 default=None）
              └─> FragmentFD 循环下载每个片段
                  └─> FragmentFD._download_fragment
                      └─> HttpQuietDownloader(HttpFD).download(片段URL)
                          └─> 必要时 AES-128 解密
                          └─> 按序追加到目标文件
```

### 7.3 HLS 直播流（FFmpeg）

```
YoutubeDL.process_info
  └─> YoutubeDL.dl
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'm3u8' (因为 is_live=True)
          └─> _get_suitable_downloader(info, 'm3u8', ...)
              ├─> ⑤ is_live=True → 返回 FFmpegFD
      └─> FFmpegFD.download(filename, info)
          └─> FFmpegFD.real_download
              └─> ffmpeg -y [-ss/-t] -i <m3u8_url> -c copy <output>
```

### 7.4 DASH 非直播单轨（原生分片）

```
YoutubeDL.process_info
  └─> YoutubeDL.dl
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'http_dash_segments' (len=1)
          └─> len(downloaders) == 1 → 返回 DashSegmentsFD
      └─> DashSegmentsFD.download(filename, info)
          └─> DashSegmentsFD.real_download
              ├─> 尝试 dash_frag_urls 外部下载器（default=None）→ 无
              └─> requested_formats 为空 → [info_dict] 单轨
                  └─> download_and_append_fragments_multiple(单轨 args)
                      └─> 内部 HttpFD 逐段下载 + 追加
```

### 7.5 DASH 非直播音视频分轨（⚠️ 纠正之前错误：走路径拆分，不是统一 DashSegmentsFD）

```
YoutubeDL.process_info
  └─> protocol = 'http_dash_segments+http_dash_segments'
  │
  └─> get_suitable_downloader(info)
      ├─> protocols = ['http_dash_segments', 'http_dash_segments'], len=2
      ├─> downloaders = [DashSegmentsFD, DashSegmentsFD]
      ├─> 特殊规则1 (FFmpeg)? → set != {FFmpegFD} ❌
      ├─> 特殊规则2 (Dash generator)?
      │   ├─> set(downloaders) == {DashSegmentsFD} ✅
      │   └─> set(protocols) == {'http_dash_segments_generator'}?
      │       └─> 实际是 {'http_dash_segments'} ❌
      ├─> 特殊规则3 (len==1)? → len==2 ❌
      └─> → 返回 None
  │
  └─> fd is None 分支（路径拆分）
      ├─> 遍历 requested_formats[0] (视频 DASH)
      │   ├─> new_info: 剥离 requested_formats，填入单轨道 info
      │   ├─> new_info['protocol'] = 'http_dash_segments'（单轨，len=1）
      │   ├─> 文件名: f137.mp4
      │   └─> self.dl('f137.mp4', new_info)
      │           └─> get_suitable_downloader(new_info) → len==1 → DashSegmentsFD
      │           └─> DashSegmentsFD 原生逐片段下载 → f137.mp4
      ├─> 遍历 requested_formats[1] (音频 DASH)
      │   ├─> new_info: 剥离 requested_formats，填入单轨道 info
      │   ├─> new_info['protocol'] = 'http_dash_segments'（单轨，len=1）
      │   ├─> 文件名: f140.m4a
      │   └─> self.dl('f140.m4a', new_info)
      │           └─> get_suitable_downloader(new_info) → len==1 → DashSegmentsFD
      │           └─> DashSegmentsFD 原生逐片段下载 → f140.m4a
      └─> 全部完成 → FFmpegMergerPP: ffmpeg -i f137.mp4 -i f140.m4a -c copy 合并
```

### 7.6 DASH 直播回放多轨（--live-from-start，唯一的 DashSegmentsFD 多轨统一场景）

```
YoutubeDL.process_info
  └─> protocol = 'http_dash_segments_generator+http_dash_segments_generator'
  │
  └─> get_suitable_downloader(info)
      ├─> set(downloaders) == {DashSegmentsFD} ✅
      ├─> set(protocols) == {'http_dash_segments_generator'} ✅
      ├─> 非 (to_stdout && len>1) ✅
      └─> 特殊规则 2 → 返回 DashSegmentsFD
  │
  └─> fd != FFmpegFD 分支
      ├─> 为每个 format 预分配 f{id}.ext 文件名，存入 format['filepath']
      ├─> info_dict['url'] = '\n'.join(urls)（DashSegmentsFD 实际不用这个）
      └─> self.dl(temp_filename, info_dict)
          └─> DashSegmentsFD.download
              └─> DashSegmentsFD.real_download
                  ├─> protocol 含 generator → real_downloader = None（禁用外部下载器）
                  ├─> 遍历 info_dict['requested_formats']（2 个轨道）
                  │   ├─> 视频轨道：ctx['filename'] = format['filepath'] = f137.mp4
                  │   └─> 音频轨道：ctx['filename'] = format['filepath'] = f140.m4a
                  └─> download_and_append_fragments_multiple(视频args, 音频args)
                      └─> 内部 HttpFD 多轨并行下载片段
  └─> 后处理：FFmpegMergerPP 合并 f137.mp4 + f140.m4a
```

### 7.7 DASH 直播（FFmpeg 直接处理）

```
YoutubeDL.process_info
  └─> YoutubeDL.dl
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'http_dash_segments'
          └─> _get_suitable_downloader(info, 'http_dash_segments', ...)
              ├─> ④ protocol==http_dash_segments && is_live && !native
              └─> → 返回 FFmpegFD
      └─> FFmpegFD.download(filename, info)
          └─> FFmpegFD.real_download
              └─> ffmpeg -y -re -i <mpd_url> -c copy <output>
                      ↑
                    直播 DASH 专用 -re 参数
```

### 7.8 DASH(视频) + HTTP(音频) 下载器不统一 → 路径拆分

```
YoutubeDL.process_info
  └─> protocol = 'http_dash_segments+https'
  │
  └─> get_suitable_downloader(info)
      ├─> http_dash_segments → DashSegmentsFD
      ├─> https → HttpFD
      ├─> set = {DashSegmentsFD, HttpFD}，len(set)>1
      └─> → 返回 None
  │
  └─> fd is None 分支（路径拆分）
      ├─> 遍历 requested_formats[0] (视频 DASH)
      │   ├─> new_info: 剥离 requested_formats，填入单轨道 info
      │   ├─> 文件名: f137.mp4
      │   └─> self.dl('f137.mp4', new_info)
      │           └─> get_suitable_downloader(new_info) → DashSegmentsFD
      │           └─> DashSegmentsFD 原生逐片段下载
      ├─> 遍历 requested_formats[1] (音频 HTTP)
      │   ├─> new_info: 剥离 requested_formats，填入单轨道 info
      │   ├─> 文件名: f140.m4a
      │   └─> self.dl('f140.m4a', new_info)
      │           └─> get_suitable_downloader(new_info) → HttpFD
      │           └─> HttpFD 直链下载
      └─> 全部完成 → FFmpegMergerPP: ffmpeg -i f137.mp4 -i f140.m4a -c copy 合并
```

### 7.9 多 HLS 音轨 + 视频轨（全 FFmpeg 直接合并）

```
YoutubeDL.process_info
  └─> protocol = 'm3u8+m3u8+m3u8'（或混合 m3u8_native 但都被改写为 FFmpegFD）
  │
  └─> get_suitable_downloader(info)
      ├─> downloaders = [FFmpegFD, FFmpegFD, FFmpegFD]
      ├─> set(downloaders) == {FFmpegFD} ✅
      ├─> can_merge_formats(info, params) ✅
      └─> → 返回 FFmpegFD
  │
  └─> fd == FFmpegFD 分支
      └─> self.dl(temp_filename, info_dict)
          └─> FFmpegFD.download
              └─> FFmpegFD._call_downloader
                  └─> 遍历 requested_formats: 每个 fmt 加 -ss/-t + -cookies + -headers + -i url
                  └─> 最后加 -c copy -map 0:v:0 -map 1:a:0 ... → 输出单文件
  └─> 无需后处理合并，已一步完成
```

### 7.10 DASH 直播 + `--downloader native`（⚠️ 异常路径：只报错，不兜底 FFmpeg）

```
用户参数: --downloader native  (强制原生下载器)

YoutubeDL.process_info
  └─> protocol = 'http_dash_segments'，is_live=True
  │
  └─> get_suitable_downloader(info)
      └─> _get_suitable_downloader(info, 'http_dash_segments', ...)
          ├─> ① 无分段需求
          ├─> ② external_downloader='native' → 跳过外部下载器检测
          ├─> ③ 非 stdout 合并
          ├─> ④ protocol==http_dash_segments && is_live=True
          │   └─> 但 (external_downloader or '').lower() == 'native'
          │   └─> 条件不成立 ❌ → 不返回 FFmpegFD
          ├─> ⑤ 非 HLS，跳过
          └─> ⑥ PROTOCOL_MAP['http_dash_segments'] → DashSegmentsFD
  │
  └─> self.dl(temp_filename, info_dict)
      └─> DashSegmentsFD.download(filename, info_dict)
          └─> DashSegmentsFD.real_download (dash.py L17-L24)
              ├─> protocol 不含 generator → 进入 else 分支
              ├─> info_dict.get('is_live') → True ⚠️
              ├─> self.report_error('Live DASH videos are not supported')  ← 只报错
              │   （无 FFmpegFD fallback，无降级逻辑）
              ├─> 继续执行: real_downloader = get_suitable_downloader(..., protocol='dash_frag_urls')
              ├─> 后续 fragments 解析/下载因直播无固定片段列表失败
              └─> 最终返回 False（下载失败）

最终结果: 用户看到 "ERROR: Live DASH videos are not supported"，下载中断。
         不会自动切换到 FFmpeg，也没有任何兜底降级。
```

> **关键教训**：分派层的 `!= 'native'` 判断保护了默认路径和 ffmpeg 等外部下载器，但 `'native'` 作为特殊字符串被硬编码排除在外。用户若错误地使用 `--downloader native` 下载 DASH 直播，不会得到 FFmpeg 兜底，只会得到一条报错。

---

## 八、关键设计模式与决策点

### 8.1 三次分派（非两次）设计

1. **协议层分派**：`determine_protocol` 根据 URL/元数据确定传输协议（直播 vs 非直播 HLS）
2. **下载器层分派**：`get_suitable_downloader` → `_get_suitable_downloader` 选择下载器类
3. **运行时再分派**：下载器内部 `real_download` 获取 manifest 后再次决策（HlsFD 降级 FFmpeg、DASH/HLS 委托外部伪协议下载器）

这种设计允许：
- 运行时检测能力（如下载 m3u8 manifest 后才知道是否有 DRM、是否缺 pycryptodomex）
- 有条件降级（HlsFD 原生不支持就用 FFmpeg，但强制 native 或缺 FFmpeg 时例外）
- 外部下载器接管片段级批量下载（伪协议扩展点）
- 兜底策略但非万无一失（DASH 直播 + 强制 native 会直接报错失败，不降转 FFmpeg）

### 8.2 多协议聚合的严格漏斗（7 条路径）

多协议（`len(protocols) >= 2`）时的返回值判断：

```
多协议
  │
  ▼
所有下载器都是 FFmpegFD + can_merge_formats?
  ├─> 是 → FFmpegFD（FFmpeg 多 -i 直接合并）
  │否
  ▼
所有下载器都是 DashSegmentsFD + 所有协议都是 http_dash_segments_generator + 非 stdout 多轨?
  ├─> 是 → DashSegmentsFD（内部通过 requested_formats 多轨下载）
  │否
  ▼
len(downloaders) == 1?
  ├─> 是 → 不可能（多协议时 len>=2）
  │否
  ▼
返回 None → YoutubeDL 路径拆分
```

> **纠正之前的错误理解**：`len(downloaders) == 1` 只代表**协议数量为 1**，不代表"多协议但下载器相同"。多协议但下载器相同的情况（如普通 DASH 多轨、多 HTTP 直链分轨）在当前代码中**没有统一规则**，全部返回 None 走路径拆分。

### 8.3 优先级排序（7 层漏斗）

单协议下载器选择优先级从高到低：
1. 分段下载需求 (`section_start/end`) → FFmpegFD
2. 用户指定外部下载器 → 相应 `ExternalFD`（需 `SUPPORTED_PROTOCOLS` 包含协议）
3. stdout 输出 + 多格式可合并 → FFmpegFD
4. DASH 直播特殊规则 → FFmpegFD
5. HLS 直播/原生偏好规则 → HlsFD / FFmpegFD
6. 协议映射表 `PROTOCOL_MAP`
7. 兜底默认 `HttpFD`

### 8.4 递归调用

`get_suitable_downloader` 内部会递归调用自身：
- 查询 `m3u8_frag_urls` / `dash_frag_urls` 伪协议支持时（传 `default=None`）
- 多协议组合时分别调用 `_get_suitable_downloader` 再聚合
- YoutubeDL 路径拆分后逐轨调用（上层递归，剥离 requested_formats 后保证单协议）

### 8.5 扩展点设计

- 新协议 → 添加到 `PROTOCOL_MAP` + 必要条件到 `_get_suitable_downloader`
- 新外部下载器 → 继承 `ExternalFD` 并定义 `SUPPORTED_PROTOCOLS` + `_make_cmd`
- 伪协议接管片段 → 在自定义外部下载器 `SUPPORTED_PROTOCOLS` 中加入 `m3u8_frag_urls` / `dash_frag_urls`
- 新的多协议统一下载器 → 在 `get_suitable_downloader` 的特殊规则 1/2 之后新增分支（目前只有 FFmpeg 和 Dash generator 多轨统一）

---

## 九、代码优化建议

### 9.1 `get_suitable_downloader` 多协议聚合规则不完整

当前代码对多协议情况只覆盖了 2 个特例（FFmpeg 合并和 Dash generator 回放），但像「普通 DASH 多轨」「多 HTTP 直链分轨」这类同类下载器场景却返回 None 走路径拆分。而 DashSegmentsFD 和 HttpFD 本身内部有能力处理多轨（DashSegmentsFD 用 requested_formats，HttpFD 理论上可支持多文件）。

**建议**：增加同类下载器聚合分支：
```python
# 新增特殊规则：多协议但下载器全相同且下载器自身支持多轨
elif len(set(downloaders)) == 1:
    only_downloader = list(set(downloaders))[0]
    if hasattr(only_downloader, 'can_handle_multiple_formats') and only_downloader.can_handle_multiple_formats(info_copy):
        return only_downloader
    # 或者更简单：如果是 FragmentFD 子类，默认支持多轨
    if issubclass(only_downloader, FragmentFD):
        return only_downloader
```

### 9.2 `_get_suitable_downloader` 函数过长

当前实现（约 40 行）包含多个 if-elif 分支，可读性较差。建议重构为策略模式：

```python
# 优化后结构
DOWNLOADER_STRATEGIES = [
    # (条件函数, 策略函数)，按优先级排序
    (lambda i, p, pa: (i.get('section_start') or i.get('section_end')) and FFmpegFD.can_download(i),
     lambda i, p, pa: FFmpegFD),
    (lambda i, p, pa: p == 'http_dash_segments' and i.get('is_live') and (pa.get('external_downloader') or '').lower() != 'native',
     lambda i, p, pa: FFmpegFD),
    # ... 其他策略
]

def _get_suitable_downloader(info_dict, protocol, params, default):
    for condition, strategy in DOWNLOADER_STRATEGIES:
        if condition(info_dict, protocol, params):
            return strategy(info_dict, protocol, params)
    return PROTOCOL_MAP.get(protocol, default or HttpFD)
```

### 9.3 伪协议检测逻辑不清晰

`m3u8_frag_urls` 和 `dash_frag_urls` 的检测目的不明显，建议添加注释明确说明这是**留给外部下载器的扩展点**，并在 `ExternalFD` 文档中说明用法。

### 9.4 直播流判定重复

- `determine_protocol` 中已根据 `is_live` 将 m3u8 分流为 `m3u8`（直播）和 `m3u8_native`（非直播）
- 但 `_get_suitable_downloader` L114 中又再次判断 `is_live` → 返回 FFmpegFD
- 存在冗余判断（`m3u8` 协议本身已经意味着直播，除非 extractor 错误设置）

### 9.5 DashSegmentsFD 直播错误与分派层重复

- 分派层（`_get_suitable_downloader` ④）已经拦截 DASH 直播 → FFmpegFD
- DashSegmentsFD 内部 L21-22 又做了一次 `is_live` 检查并报错
- 双重检查是合理的防护措施，但 DashSegmentsFD 报错信息应提示用户移除 `--downloader native`（因为这是唯一能绕过分派层拦截的路径）

---

## 十、总结：HTTP、HLS、DASH 完整选择图谱

### 10.1 协议与下载器映射矩阵

| 场景 | 协议 | 下载器 | 内部片段下载 |
|------|------|--------|-------------|
| 普通直链 | http/https/ftp | **HttpFD** | - |
| HLS 非直播 + 无 DRM | m3u8_native | **HlsFD**（原生） | HttpFD |
| HLS 非直播 + 有 DRM / 原生不支持 | m3u8_native | **FFmpegFD**（降级） | ffmpeg 内部 |
| HLS 直播 | m3u8 | **FFmpegFD**（强制） | ffmpeg 内部 |
| HLS + `--downloader native` | m3u8_native | **HlsFD**（强制） | HttpFD |
| HLS AES-128 + Cryptodome 缺失 + FFmpeg 可处理 ⚠️ | m3u8_native | **FFmpegFD**（降级） | ffmpeg 内部 |
| HLS AES-128 + Cryptodome 缺失 + FFmpeg 不可处理 ⚠️ | m3u8_native | **HlsFD**（纯 Python AES 解密，极慢） | HttpFD |
| DASH 非直播（单轨） | http_dash_segments | **DashSegmentsFD** | HttpFD |
| DASH 非直播（音视频分轨）⚠️ | http_dash_segments+http_dash_segments | **None → 路径拆分** → 逐轨 DashSegmentsFD | 逐轨 HttpFD → FFmpegMergerPP 合并 |
| DASH 直播回放多轨（--live-from-start） | http_dash_segments_generator+... | **DashSegmentsFD**（唯一多轨统一非 FFmpeg） | 多轨并行 HttpFD |
| DASH 直播（默认） | http_dash_segments | **FFmpegFD**（强制） | ffmpeg 内部 (-re) |
| DASH + `--downloader native` + 直播 ⚠️ | http_dash_segments | DashSegmentsFD → **report_error 报错，不兜底 FFmpeg，下载失败** | - |
| DASH + `--downloader ffmpeg` | http_dash_segments | **FFmpegFD**（用户指定） | ffmpeg 内部 |
| 多格式下载器不统一 | A+B | None → **路径拆分** | 逐轨各自 |
| 多格式全指向 FFmpeg 且可合并 | 任意组合 | **FFmpegFD**（直接合并） | ffmpeg 多输入 |
| 分段截取（任意协议） | 任意 | **FFmpegFD**（强制） | ffmpeg -ss/-t |

> ⚠️ 表格中标注 ⚠️ 的行是**本次或之前纠正的错误结论**：
> 1. 普通 DASH 音视频分轨不会统一走 DashSegmentsFD，而是返回 None 走路径拆分
> 2. HLS AES-128 + pycryptodomex 缺失**不一定**降级到 FFmpeg，有 4 种组合（详见 3.2.3）
> 3. DASH 直播 + `--downloader native` **不会兜底到 FFmpeg**，DashSegmentsFD 只报错返回 False（下载失败）

### 10.2 核心层次关系

```
                    ┌──────────────────────────────┐
                    │  FFmpegFD (万能降级/合并器)    │
                    │  支持: http/m3u8/dash 等全部   │
                    │  用途: 直播/DRM/分段/多轨合并  │
                    └──────────────▲───────────────┘
                                   │ 降级/直接接管
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
┌───────▼───────┐          ┌───────▼───────┐          ┌───────▼────────┐
│    HttpFD     │          │    HlsFD      │          │ DashSegmentsFD │
│ (基础下载器)  │          │  (HLS 原生)    │          │  (DASH 原生)    │
└───────▲───────┘          └───────▲───────┘          └───────▲────────┘
        │ 逐片段下载               │ 委托 FragmentFD           │ 委托 FragmentFD
        │                          │                          │
        └──────────────────────────┴──────────────────────────┘
                    FragmentFD (分片下载调度基类)

多协议 (A+B+C...)?
  │
  ├─> 全 FFmpeg + 可合并 → FFmpegFD（直接合并）
  ├─> 全 DashSegmentsFD + 全是 generator → DashSegmentsFD（多轨统一）← 仅此一例
  └─> 其他所有 → None → YoutubeDL 路径拆分（逐轨独立下载）
```

**一句话总结**：
> **HTTP 是基础**（所有分片最终都落到 HttpFD），**HLS 有原生/FFmpeg 两条路**（取决于直播/DRM/用户偏好，AES-128 加密在 pycryptodomex 缺失时**不一定降级**，仅 FFmpeg 可处理且可用时才降级，否则走纯 Python 原生解密极慢路径），**DASH 非直播单轨走原生、直播默认强制 FFmpeg、但 `--downloader native` 会绕过分派层导致 DashSegmentsFD 只报错不兜底、普通多轨返回 None 走路径拆分**（只有 `--live-from-start` 的 generator 多轨才统一 DashSegmentsFD），**FFmpeg 是万能降级/合并器但存在盲点**（强制 native DASH 直播不会兜底），**下载器不统一/普通 DASH 多轨时逐轨拆分后分别下载再合并**。三者不是互斥关系，而是**层次 + 有条件降级 + 路径拆分**的组合关系。
