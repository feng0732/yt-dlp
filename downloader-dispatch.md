# yt-dlp 协议下载器分派机制完整分析

## 一、核心架构与类层次

### 1.1 类继承关系

```
FileDownloader [common.py]
    ├── HttpFD [http.py]                    # HTTP 直链下载
    │   └── HttpQuietDownloader [fragment.py]
    ├── FragmentFD [fragment.py]            # 分片下载基类
    │   ├── ExternalFD [external.py]        # 外部下载器基类
    │   │   ├── CurlFD [external.py]
    │   │   ├── AxelFD [external.py]
    │   │   ├── WgetFD [external.py]
    │   │   ├── Aria2cFD [external.py]
    │   │   ├── HttpieFD [external.py]
    │   │   └── FFmpegFD [external.py]      # FFmpeg 下载/合并
    │   ├── HlsFD [hls.py]                  # HLS 原生下载
    │   ├── DashSegmentsFD [dash.py]        # DASH 分片下载
    │   ├── F4mFD [f4m.py]
    │   └── IsmFD [ism.py]
    └── RtmpFD [rtmp.py]
```

### 1.2 关键文件定位

| 文件 | 核心作用 |
|------|---------|
| [downloader/__init__.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/__init__.py) | 分派入口 `get_suitable_downloader` |
| [downloader/common.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/common.py) | 基类 `FileDownloader` |
| [downloader/http.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/http.py) | `HttpFD` HTTP 下载 |
| [downloader/hls.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/hls.py) | `HlsFD` HLS 原生下载 |
| [downloader/dash.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/dash.py) | `DashSegmentsFD` DASH 下载 |
| [downloader/fragment.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/fragment.py) | `FragmentFD` 分片基类 |
| [downloader/external.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/external.py) | 外部下载器族 |
| [utils/_utils.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/utils/_utils.py) | `determine_protocol` 协议判定 |
| [YoutubeDL.py](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/YoutubeDL.py) | 调用下载器的业务逻辑 |

---

## 二、完整分派流程（三层分派）

### 2.1 第一层：协议判定 (`determine_protocol`)

位置：[utils/_utils.py#L3178-L3197](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/utils/_utils.py#L3178-L3197)

```python
def determine_protocol(info_dict):
    protocol = info_dict.get('protocol')
    if protocol is not None:
        return protocol  # 已指定协议直接使用

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
位置：[downloader/__init__.py#L41-L61](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/__init__.py#L41-L61)

| 协议 | 下载器 | 说明 |
|------|--------|------|
| `http` / `https` / `ftp` | `HttpFD` (默认) | 普通 HTTP 下载 |
| `m3u8_native` | `HlsFD` | HLS 原生下载 |
| `m3u8` | `FFmpegFD` | HLS 通过 FFmpeg |
| `http_dash_segments` | `DashSegmentsFD` | DASH 分片下载 |
| `http_dash_segments_generator` | `DashSegmentsFD` | DASH 直播回放 |
| `rtmp` | `RtmpFD` | RTMP 协议 |

---

### 2.2 第二层：主分派 (`get_suitable_downloader`)

位置：[downloader/__init__.py#L4-L20](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/__init__.py#L4-L20)

```python
def get_suitable_downloader(info_dict, params={}, default=NO_DEFAULT, protocol=None, to_stdout=False):
    info_dict['protocol'] = determine_protocol(info_dict)
    info_copy = info_dict.copy()
    info_copy['to_stdout'] = to_stdout

    # 支持多协议组合（如 http+http_dash_segments）
    protocols = (protocol or info_copy['protocol']).split('+')
    downloaders = [_get_suitable_downloader(info_copy, proto, params, default) for proto in protocols]

    # 特殊规则1：全部可由 FFmpeg 处理且支持合并 -> FFmpegFD
    if set(downloaders) == {FFmpegFD} and FFmpegFD.can_merge_formats(info_copy, params):
        return FFmpegFD
    # 特殊规则2：全部是 DASH 生成器且非多协议 stdout -> DashSegmentsFD
    elif (set(downloaders) == {DashSegmentsFD}
          and not (to_stdout and len(protocols) > 1)
          and set(protocols) == {'http_dash_segments_generator'}):
        return DashSegmentsFD
    # 特殊规则3：单一下载器直接返回
    elif len(downloaders) == 1:
        return downloaders[0]
    return None  # 多种下载器无法统一，后续分别处理
```

---

### 2.3 第三层：单协议分派 (`_get_suitable_downloader`)

位置：[downloader/__init__.py#L87-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/__init__.py#L87-L126)

这是最核心的分派逻辑，按优先级判断：

```
开始
  │
  ▼
默认下载器 = HttpFD
  │
  ▼
有 section_start/section_end 且 FFmpeg 可用? ──是──> FFmpegFD
  │否
  ▼
用户指定了外部下载器? ──是──> 检查该外部下载器是否支持此协议
  │否                     │是
  │                       ▼
  │                     返回外部下载器
  │
  ▼
输出到 stdout 且 FFmpeg 支持合并? ──是──> FFmpegFD
  │否
  ▼
协议是 http_dash_segments?
  │是
  ├─> 是直播 且 非强制native? ──是──> FFmpegFD
  │否
  ▼
协议是 m3u8 / m3u8_native?
  │是
  ├─> 是直播? ──是──> FFmpegFD
  │
  ├─> 强制 native (external_downloader='native')? ──是──> HlsFD
  │
  ├─> 协议是 m3u8_native 且 有支持 m3u8_frag_urls 的外部下载器? ──是──> HlsFD
  │
  ├─> hls_prefer_native=True? ──是──> HlsFD
  │
  └─> hls_prefer_native=False? ──是──> FFmpegFD
  │
  ▼
最后查 PROTOCOL_MAP，找不到用默认 HttpFD
```

**关键判断点解析**：

1. **分段下载优先 FFmpeg**：只要设置了 `section_start` 或 `section_end`，优先用 FFmpegFD（因为原生下载器不支持分段截取）。

2. **外部下载器介入点**：用户通过 `--downloader` 指定外部下载器时，在检查完分段需求后立即判断。

3. **HLS 的两条路径**：
   - `m3u8` 协议 → 默认走 FFmpegFD
   - `m3u8_native` 协议 → 走 HlsFD 原生下载，但会被多个条件改写

4. **HLS 直播强制 FFmpeg**：HLS 直播流 (`is_live=True`) 始终使用 FFmpegFD，原生下载器不支持。

5. **片段 URL 协议检测**：通过 `m3u8_frag_urls` 和 `dash_frag_urls` 伪协议检测外部下载器是否支持接管片段下载。

---

## 三、HTTP、HLS、DASH 三者选择关系

### 3.1 HTTP 下载 (HttpFD)

**适用场景**：
- 普通直链下载（http/https/ftp）
- HLS/DASH 内部下载单个片段

**分派路径**：
1. `determine_protocol` 返回 `http`/`https`/`ftp`
2. `_get_suitable_downloader` 中无特殊条件匹配
3. 从 `PROTOCOL_MAP` 找不到对应项，使用默认 `HttpFD`

**关键特性**：
- 支持断点续传（通过 `Range` 头）
- 支持分块下载（`http_chunk_size`）
- 支持速率限制
- 支持重试机制

**代码位置**：[http.py#L23-L376](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/http.py#L23-L376)

---

### 3.2 HLS 下载 (HlsFD vs FFmpegFD)

#### 3.2.1 HlsFD（原生下载）

**适用场景**：
- 非直播 HLS (`m3u8_native` 协议)
- 无 DRM 保护
- 仅 AES-128 加密（需要 pycryptodomex）
- 用户设置 `--downloader native` 或 `--hls-prefer-native`

**分派路径**：
```
protocol = m3u8_native
  │
  ├─> is_live? → 否
  ├─> external_downloader='native'? → 可选
  ├─> 有支持 m3u8_frag_urls 的外部下载器? → 可选
  └─> hls_prefer_native=True? → 可选
  │
  ▼
返回 HlsFD
```

**内部再分派**（HlsFD.real_download）：
位置：[hls.py#L74-L140](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/hls.py#L74-L140)

1. 下载并解析 m3u8 manifest
2. 检查是否支持（`can_download`）：
   - 不是直播
   - 无 DRM（FairPlay/PlayReady/Flash Access）
   - 加密方式仅为 NONE 或 AES-128
3. 不支持时降级到 `FFmpegFD`
4. 尝试寻找支持 `m3u8_frag_urls` 协议的外部下载器接管片段下载
5. 无外部下载器时，使用 `FragmentFD` 机制，内部用 `HttpFD` 逐片段下载

**片段下载机制**：
- `HlsFD` 继承自 `FragmentFD`
- `FragmentFD._download_fragment` 创建 `HttpQuietDownloader`（即 `HttpFD`）下载每个片段
- 下载后解密（如果是 AES-128）
- 按顺序追加到目标文件

---

#### 3.2.2 FFmpegFD（FFmpeg 处理 HLS）

**适用场景**：
- HLS 直播流 (`is_live=True`)
- 有 DRM 保护
- 不支持的加密方式
- 用户设置 `--hls-prefer-native=False`
- 原生下载器检测到不支持的特性

**分派路径**：
```
protocol = m3u8 或 m3u8_native
  │
  ├─> is_live? → 是 → FFmpegFD
  ├─> hls_prefer_native=False? → 是 → FFmpegFD
  └─> 原生下载器 can_download 失败 → 降级到 FFmpegFD
```

**FFmpegFD 支持的协议**：
`('http', 'https', 'ftp', 'ftps', 'm3u8', 'm3u8_native', 'rtsp', 'rtmp', 'rtmp_ffmpeg', 'mms', 'http_dash_segments')`

---

### 3.3 DASH 下载 (DashSegmentsFD)

**适用场景**：
- DASH 分片视频 (`http_dash_segments` 协议)
- DASH 直播回放 (`http_dash_segments_generator` 协议)

**分派路径**：
```
protocol = http_dash_segments / http_dash_segments_generator
  │
  ├─> is_live 且 非强制native? → 是 → FFmpegFD
  │
  ▼
查 PROTOCOL_MAP → DashSegmentsFD
```

**内部再分派**（DashSegmentsFD.real_download）：
位置：[dash.py#L17-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/downloader/dash.py#L17-L68)

1. 如果是 `http_dash_segments_generator`（直播回放），不使用外部下载器
2. 普通 DASH 尝试寻找支持 `dash_frag_urls` 的外部下载器
3. 找到则将所有片段 URL 传给外部下载器批量下载
4. 否则使用 `FragmentFD` 机制，内部用 `HttpFD` 逐片段下载

**DASH 多轨道处理**：
- 支持同时下载音视频多个轨道（`requested_formats`）
- 调用 `download_and_append_fragments_multiple` 并行/串行下载多个轨道
- 后续由 FFmpeg 合并

---

## 四、特殊协议：`m3u8_frag_urls` 与 `dash_frag_urls`

这两个是**伪协议**，用于检测外部下载器是否支持接管 HLS/DASH 的片段下载。

### 4.1 检测机制

在 `_get_suitable_downloader` 中：
```python
elif protocol == 'm3u8_native' and get_suitable_downloader(
        info_dict, params, None, protocol='m3u8_frag_urls', to_stdout=info_dict['to_stdout']):
    return HlsFD
```

**关键点**：传入 `default=None`，如果没有外部下载器支持该协议，返回 `None`，条件不成立。

### 4.2 外部下载器支持情况

目前代码中**没有任何外部下载器**在 `SUPPORTED_PROTOCOLS` 中包含这两个协议：
- `ExternalFD`: `('http', 'https', 'ftp', 'ftps')`
- `Aria2cFD`: `('http', 'https', 'ftp', 'ftps')`
- `FFmpegFD`: 不包含这两个

**设计意图**：这是一个扩展点，允许第三方外部下载器通过插件机制支持批量下载 HLS/DASH 片段，而无需 yt-dlp 逐个下载。

---

## 五、YoutubeDL.py 中的调用点

### 5.1 主下载流程

位置：[YoutubeDL.py#L3304](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/YoutubeDL.py#L3304)

```python
fd = get_suitable_downloader(info, params, to_stdout=(name == '-'))(self, params)
# ...
return fd.download(name, new_info, subtitle)
```

### 5.2 合并格式时的下载器选择

位置：[YoutubeDL.py#L3474-L3480](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/YoutubeDL.py#L3474-L3480)

```python
fd = get_suitable_downloader(info_dict, self.params, to_stdout=temp_filename == '-')
if fd != FFmpegFD and 'no-direct-merge' not in self.params['compat_opts'] and (
        info_dict.get('section_start') or info_dict.get('section_end')):
    # 分段下载必须用 FFmpeg，否则报错
    self.report_error(f'{msg}. Aborting')
    return
```

### 5.3 下载器名称记录

位置：[YoutubeDL.py#L3630](file:///d:/fz/0601-2/solo-dogfeeding/code/85-yt-dlp/yt_dlp/YoutubeDL.py#L3630)

```python
downloader = get_suitable_downloader(info_dict, self.params) if 'protocol' in info_dict else None
downloader = downloader.FD_NAME if downloader else None
```

---

## 六、完整调用链示例

### 6.1 普通 HTTP 视频下载

```
YoutubeDL.process_info
  └─> YoutubeDL.download
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'https'
          └─> _get_suitable_downloader(info, 'https', ...)
              └─> PROTOCOL_MAP 无 'https' → 默认 HttpFD
      └─> HttpFD.download(filename, info)
          └─> HttpFD.real_download → 直接 HTTP 下载
```

### 6.2 非直播 HLS 视频（原生下载）

```
YoutubeDL.process_info
  └─> YoutubeDL.download
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'm3u8_native'
          └─> _get_suitable_downloader(info, 'm3u8_native', ...)
              ├─> is_live? → 否
              ├─> 无外部下载器
              └─> PROTOCOL_MAP['m3u8_native'] → HlsFD
      └─> HlsFD.download(filename, info)
          └─> HlsFD.real_download
              ├─> 下载 m3u8 manifest
              ├─> can_download 检查 → 通过
              ├─> 尝试 m3u8_frag_urls 外部下载器 → 无
              └─> FragmentFD 循环下载每个片段
                  └─> FragmentFD._download_fragment
                      └─> HttpQuietDownloader(HttpFD).download(片段URL)
```

### 6.3 HLS 直播流（FFmpeg）

```
YoutubeDL.process_info
  └─> YoutubeDL.download
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'm3u8' (因为 is_live=True)
          └─> _get_suitable_downloader(info, 'm3u8', ...)
              ├─> is_live? → 是 → 返回 FFmpegFD
      └─> FFmpegFD.download(filename, info)
          └─> FFmpegFD.real_download → 调用 ffmpeg 命令行下载
```

### 6.4 DASH 视频（音视频分离）

```
YoutubeDL.process_info
  └─> YoutubeDL.download
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'http_dash_segments+http_dash_segments'
          ├─> 对每个协议调用 _get_suitable_downloader → [DashSegmentsFD, DashSegmentsFD]
          └─> set(downloaders) == {DashSegmentsFD} → 返回 DashSegmentsFD
      └─> DashSegmentsFD.download(filename, info)
          └─> DashSegmentsFD.real_download
              ├─> 尝试 dash_frag_urls 外部下载器 → 无
              └─> download_and_append_fragments_multiple
                  ├─> 下载音频轨道片段（内部 HttpFD）
                  └─> 下载视频轨道片段（内部 HttpFD）
  └─> 后处理：FFmpeg 合并音视频
```

---

## 七、关键设计模式与决策点

### 7.1 两次分派设计

1. **高层分派**：`get_suitable_downloader` 根据协议类型选择下载器类
2. **低层分派**：下载器内部 `real_download` 再次决定是否委托给其他下载器

这种设计允许：
- 运行时检测能力（如下载 manifest 后才知道是否有 DRM）
- 优雅降级（原生不行就用 FFmpeg）
- 外部下载器接管片段下载

### 7.2 优先级排序

下载器选择优先级从高到低：
1. 分段下载需求 → FFmpegFD
2. 用户指定外部下载器 → 相应 ExternalFD
3. stdout 输出且支持合并 → FFmpegFD
4. 协议特殊规则（HLS 直播、DASH 直播等）
5. 协议映射表 PROTOCOL_MAP
6. 默认 HttpFD

### 7.3 递归调用

`get_suitable_downloader` 内部会递归调用自身：
- 检查 `m3u8_frag_urls` / `dash_frag_urls` 支持时
- 多协议组合时分别调用

### 7.4 扩展点

- 新协议只需添加到 `PROTOCOL_MAP`
- 新外部下载器只需继承 `ExternalFD` 并定义 `SUPPORTED_PROTOCOLS`
- 通过 `m3u8_frag_urls` / `dash_frag_urls` 伪协议扩展片段下载能力

---

## 八、代码优化建议

### 8.1 `_get_suitable_downloader` 函数过长

当前实现（约 40 行）包含多个 if-elif 分支，可读性较差。建议重构为策略模式：

```python
# 优化后结构
DOWNLOADER_STRATEGIES = [
    (lambda d, p, i: i.get('section_start') or i.get('section_end'), 
     lambda d, p, i: FFmpegFD if FFmpegFD.can_download(i) else None),
    (lambda d, p, i: p in ('m3u8', 'm3u8_native') and i.get('is_live'),
     lambda d, p, i: FFmpegFD),
    # ... 其他策略
]

def _get_suitable_downloader(info_dict, protocol, params, default):
    for condition, strategy in DOWNLOADER_STRATEGIES:
        if condition(info_dict, protocol, params):
            result = strategy(info_dict, protocol, params)
            if result:
                return result
    return PROTOCOL_MAP.get(protocol, default or HttpFD)
```

### 8.2 伪协议检测逻辑不清晰

`m3u8_frag_urls` 和 `dash_frag_urls` 的检测目的不明显，建议添加注释说明这是扩展点。

### 8.3 直播流判定重复

`determine_protocol` 中已根据 `is_live` 决定 `m3u8` vs `m3u8_native`，但 `_get_suitable_downloader` 中又再次判断 `is_live`，存在重复逻辑。

---

## 九、总结

yt-dlp 的下载器分派机制是一个**三层递归决策系统**：

1. **协议层**：根据 URL 和元数据确定传输协议
2. **下载器层**：根据协议、用户配置、内容特性选择主下载器
3. **片段层**：HLS/DASH 下载器内部再次决策是自己下载还是委托给 HTTP/外部下载器

**HTTP、HLS、DASH 的选择关系**：
- **HTTP** 是基础，所有分片下载最终都落到 HttpFD
- **HLS** 有两条路径：原生 HlsFD（非直播、无 DRM）或 FFmpegFD（直播、DRM）
- **DASH** 始终走 DashSegmentsFD，但内部用 HttpFD 下载每个分片
- **FFmpeg** 是万能降级方案，几乎支持所有协议，负责合并和直播

三者不是互斥关系，而是**层次关系**：HLS/DASH 构建在 HTTP 之上，FFmpeg 可以直接接管任何协议。
