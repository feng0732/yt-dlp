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
    # 特殊规则2：全部是 DASH 生成器（直播回放）且非 stdout 多协议 -> 返回 DashSegmentsFD
    elif (set(downloaders) == {DashSegmentsFD}
          and not (to_stdout and len(protocols) > 1)
          and set(protocols) == {'http_dash_segments_generator'}):
        return DashSegmentsFD
    # 特殊规则3：只有一个下载器（单协议 / 多协议但都选了同一个） -> 直接返回
    elif len(downloaders) == 1:
        return downloaders[0]
    # 多协议下载器不一致：返回 None，交给上层 YoutubeDL 做路径拆分
    return None
```

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
- 加密方式仅为 NONE 或 AES-128（若 pycryptodomex 不可用会警告并降级）
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
位置：`yt_dlp/downloader/hls.py` L74-L140

1. 下载并解析 m3u8 manifest（或使用 `hls_media_playlist_data` 中缓存的）
2. `can_download` 能力检查：
   - 不是直播（双重校验）
   - 无 DRM 标签
   - 加密方式仅为 NONE 或 AES-128
3. **不支持时降级**：直接创建 `FFmpegFD` 实例并调用其 `real_download`
4. **AES-128 + 无 pycryptodomex**：若 ffmpeg 可用也降级到 FFmpegFD
5. 尝试寻找支持 `m3u8_frag_urls` 协议的外部下载器接管片段批量下载
6. 无外部下载器时，使用 `FragmentFD` 机制，内部用 `HttpFD` 逐片段下载

**片段下载机制**：
- `HlsFD` 继承自 `FragmentFD`
- `FragmentFD._download_fragment` 创建 `HttpQuietDownloader`（即 `HttpFD`，屏蔽输出）下载每个片段
- 下载后若有 AES-128 加密，用 Cryptodome 或纯 Python 实现解密
- 按顺序追加到目标文件（WebVTT 字幕还要特殊打包）

---

#### 3.2.2 FFmpegFD（FFmpeg 处理 HLS）

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

#### 3.3.1 FFmpeg 处理 DASH 的 5 种情况

| # | 触发条件 | 判断位置 |
|---|---------|---------|
| 1 | 有 `section_start`/`section_end` 分段截取需求 + FFmpeg 可用 | `_get_suitable_downloader` ① |
| 2 | 用户指定 `--downloader ffmpeg`（或其他支持 DASH 的外部下载器） | `_get_suitable_downloader` ② + `FFmpegFD.SUPPORTED_PROTOCOLS` 含 `http_dash_segments` |
| 3 | `is_live=True` + 非强制 `native` | `_get_suitable_downloader` ④ |
| 4 | 多协议（音视频分轨）都返回 FFmpegFD + `can_merge_formats` 满足 | `get_suitable_downloader` 特殊规则 ① |
| 5 | 输出到 stdout + 多格式可合并 | `_get_suitable_downloader` ③ |

---

#### 3.3.2 直播 DASH 走 FFmpeg 的完整条件链

**判断代码**（`yt_dlp/downloader/__init__.py` L109-L111）：
```python
if protocol == 'http_dash_segments':
    if info_dict.get('is_live') and (external_downloader or '').lower() != 'native':
        return FFmpegFD
```

**三个必要条件缺一不可**：
1. `protocol == 'http_dash_segments'`（**注意**：`http_dash_segments_generator` 走 DashSegmentsFD，不走这条）
2. `is_live == True`（由 extractor 设置，不是从 URL 推断）
3. 用户**没有**强制指定 `--downloader native`（即 `external_downloader` 是 None / ffmpeg / aria2c 等非 native 值）

**FFmpeg 侧直播 DASH 特殊处理**（`yt_dlp/downloader/external.py` L484-L489）：
```python
elif protocol == 'http_dash_segments' and info_dict.get('is_live'):
    # 直播 DASH 加 -re 防止 ffmpeg 读超出最新可用分片
    args += ['-re']  # 别名 -readrate 1，但兼容旧版 ffmpeg
```

**DashSegmentsFD 直播兜底报错**（`yt_dlp/downloader/dash.py` L21-L22）：
如果因某种原因（如用户强制 `--downloader native`）直播 DASH 到达了 DashSegmentsFD，内部还会二次检查并报错：
```python
if info_dict.get('is_live'):
    self.report_error('Live DASH videos are not supported')
```

> 结论：DASH 直播有**三道关卡**确保走 FFmpeg（分派层、DashSegmentsFD 内部二次报错、FFmpeg `-re` 参数适配）

---

#### 3.3.3 DashSegmentsFD（原生分片下载）

**分派路径**（满足以下全部条件）：
```
protocol = http_dash_segments / http_dash_segments_generator
  │
  ├─> ① 无分段截取需求
  ├─> ② 未指定支持 DASH 的外部下载器（或指定了 native）
  ├─> ③ 不是 stdout 多格式合并
  ├─> ④ 非 is_live 或 强制 native（后者会报错）
  │
  ▼
PROTOCOL_MAP → DashSegmentsFD
```

**适用场景**：
- 非直播 DASH 分片视频（`http_dash_segments`）
- DASH 直播回放（`http_dash_segments_generator`，即 `--live-from-start`）

**内部再分派**（DashSegmentsFD.real_download）：
位置：`yt_dlp/downloader/dash.py` L17-L68

1. 如果是 `http_dash_segments_generator`（直播回放/`--live-from-start`），明确不使用外部下载器
2. 普通 DASH：尝试寻找支持 `dash_frag_urls` 的外部下载器（目前内置均不支持）
3. 找到 → 将所有片段 URL 列表放入 `info_dict['fragments']`，委托外部下载器批量下载
4. 未找到 → 使用 `FragmentFD` 机制，内部用 `HttpFD` 逐片段下载

**DASH 多轨道处理**：
- 支持 `requested_formats` 音视频多轨道（每个轨道都有独立的 `fragments` 列表）
- 调用 `download_and_append_fragments_multiple(*args)` 逐轨道下载（可并发）
- 若返回 FFmpegFD 合并，则 FFmpeg 用多 `-i` 参数同时读取多个 MPD URL 并 `-c copy -map` 直接合并

---

## 四、多格式下载器无法统一时的路径拆分方法

当 `get_suitable_downloader` 返回 `None`（多协议对应下载器不一致）时，由 `YoutubeDL.process_info` 中的下载逻辑做**路径拆分**。

### 4.1 触发场景

`get_suitable_downloader` 返回 `None` 当且仅当：
- `protocol` 是形如 `A+B+C` 的多协议组合（`split('+')` 后 ≥2 个）
- 各协议对应下载器 `downloaders` 集合中**存在不同类**（即 `set(downloaders)` 大小 ≥2）

典型例子：
- 视频是 `http_dash_segments`（DashSegmentsFD）+ 音频是 `https`（HttpFD）
- 视频是 `m3u8_native`（HlsFD）+ 音频是 `http_dash_segments`（DashSegmentsFD）

### 4.2 拆分算法

位置：`yt_dlp/YoutubeDL.py` L3482-L3563

```
有 requested_formats（多格式）?
  │
  ├─> get_suitable_downloader 返回了 fd（非 None）?
  │   │是
  │   ├─> fd == FFmpegFD? ──是──> 直接交给 FFmpeg 多输入合并下载
  │   │否
  │   └─> 为每个 format 预分配 f{format_id} 前缀的独立文件路径
  │       然后把所有 url 用 \n 拼接后调用 fd.download 一次
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
          │
          └─> 所有轨道下载完成后 → 交给 FFmpegMergerPP 后处理合并
```

### 4.3 关键代码片段

**路径拆分入口**（`yt_dlp/YoutubeDL.py` L3518-L3563）：
```python
elif fd:
    # 所有格式能用同一个下载器（含 FFmpegFD 直接合并）
    if fd != FFmpegFD and temp_filename != '-':
        for f in info_dict['requested_formats']:
            f['filepath'] = fname = prepend_extension(
                correct_ext(temp_filename, info_dict['ext']),
                'f{}'.format(f['format_id']), info_dict['ext'])
            downloaded.append(fname)
    info_dict['url'] = '\n'.join(f['url'] for f in info_dict['requested_formats'])
    success, real_download = self.dl(temp_filename, info_dict)  # 一次性调用
else:
    # 下载器无法统一：拆分逐轨道下载
    for f in info_dict['requested_formats']:
        new_info = dict(info_dict)
        del new_info['requested_formats']
        new_info.update(f)
        if temp_filename != '-':
            fname = prepend_extension(...)  # f{format_id}.ext
            f['filepath'] = fname
            downloaded.append(fname)
        partial_success, real_download = self.dl(fname, new_info)  # 独立调用
        # ... 聚合 success 状态
```

**FFmpeg 直接合并 vs 后处理合并的区别**：
| 方式 | 触发条件 | 执行流程 | 中间文件 |
|------|---------|---------|---------|
| FFmpegFD 直接合并 | 所有轨道的下载器都选了 FFmpegFD + `can_merge_formats` | FFmpeg 多 `-i` 输入同时读取，输出单文件 | 无（或临时 ffmpeg 管道） |
| 后处理合并（FFmpegMergerPP） | 下载器无法统一 / 原生分片下载 | 各轨道独立下载完成 → FFmpeg 后合并 | 多个 `f{id}.ext` 文件 |

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

### 6.2 多轨下载 + 路径拆分 (`process_info` 中)

位置：`yt_dlp/YoutubeDL.py` L3472-L3563

核心流程（与第四节对应）：
1. 先调用一次 `get_suitable_downloader` 尝试找到统一下载器
2. 非 FFmpegFD + 分段截取需求 → 报错（分段必须用 FFmpeg）
3. `requested_formats` 存在时：
   - fd 存在（下载器统一）→ 一次性调用（FFmpegFD 直接合并或原生单下载器多 url）
   - fd 为 `None`（下载器不统一）→ **逐轨拆分下载**，FFmpegMergerPP 后合并
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

### 7.4 DASH 非直播（原生分片，音视频分轨）

```
YoutubeDL.process_info
  └─> YoutubeDL.dl
      └─> get_suitable_downloader(info)
          ├─> determine_protocol(info) → 'http_dash_segments+http_dash_segments'
          ├─> 对每个协议调用 _get_suitable_downloader → [DashSegmentsFD, DashSegmentsFD]
          └─> set(downloaders) == {DashSegmentsFD}，len==1 → 返回 DashSegmentsFD
      └─> DashSegmentsFD.download(filename, info)
          └─> DashSegmentsFD.real_download
              ├─> 尝试 dash_frag_urls 外部下载器（default=None）→ 无
              └─> download_and_append_fragments_multiple(2 个轨道)
                  ├─> 下载音频轨道片段：内部 HttpFD 逐段下载 + 追加
                  └─> 下载视频轨道片段：内部 HttpFD 逐段下载 + 追加
  └─> 后处理：FFmpegMergerPP 合并音视频两个 .mp4 文件
```

### 7.5 DASH 直播（FFmpeg 直接处理）

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

### 7.6 DASH(视频) + HTTP(音频) 下载器不统一 → 路径拆分

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

---

## 八、关键设计模式与决策点

### 8.1 三次分派（非两次）设计

1. **协议层分派**：`determine_protocol` 根据 URL/元数据确定传输协议（直播 vs 非直播 HLS）
2. **下载器层分派**：`get_suitable_downloader` → `_get_suitable_downloader` 选择下载器类
3. **运行时再分派**：下载器内部 `real_download` 获取 manifest 后再次决策（HlsFD 降级 FFmpeg、DASH/HLS 委托外部伪协议下载器）

这种设计允许：
- 运行时检测能力（如下载 m3u8 manifest 后才知道是否有 DRM）
- 优雅降级（原生不行就用 FFmpeg）
- 外部下载器接管片段级批量下载（伪协议扩展点）

### 8.2 优先级排序（7 层漏斗）

下载器选择优先级从高到低：
1. 分段下载需求 (`section_start/end`) → FFmpegFD
2. 用户指定外部下载器 → 相应 `ExternalFD`（需 `SUPPORTED_PROTOCOLS` 包含协议）
3. stdout 输出 + 多格式可合并 → FFmpegFD
4. DASH 直播特殊规则 → FFmpegFD
5. HLS 直播/原生偏好规则 → HlsFD / FFmpegFD
6. 协议映射表 `PROTOCOL_MAP`
7. 兜底默认 `HttpFD`

### 8.3 递归调用

`get_suitable_downloader` 内部会递归调用自身：
- 查询 `m3u8_frag_urls` / `dash_frag_urls` 伪协议支持时（传 `default=None`）
- 多协议组合时分别调用 `_get_suitable_downloader` 再聚合
- YoutubeDL 路径拆分后逐轨调用（上层递归）

### 8.4 扩展点设计

- 新协议 → 添加到 `PROTOCOL_MAP` + 必要条件到 `_get_suitable_downloader`
- 新外部下载器 → 继承 `ExternalFD` 并定义 `SUPPORTED_PROTOCOLS` + `_make_cmd`
- 伪协议接管片段 → 在自定义外部下载器 `SUPPORTED_PROTOCOLS` 中加入 `m3u8_frag_urls` / `dash_frag_urls`

---

## 九、代码优化建议

### 9.1 `_get_suitable_downloader` 函数过长

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

### 9.2 伪协议检测逻辑不清晰

`m3u8_frag_urls` 和 `dash_frag_urls` 的检测目的不明显，建议添加注释明确说明这是**留给外部下载器的扩展点**，并在 `ExternalFD` 文档中说明用法。

### 9.3 直播流判定重复

- `determine_protocol` 中已根据 `is_live` 将 m3u8 分流为 `m3u8`（直播）和 `m3u8_native`（非直播）
- 但 `_get_suitable_downloader` L114 中又再次判断 `is_live` → 返回 FFmpegFD
- 存在冗余判断（`m3u8` 协议本身已经意味着直播，除非 extractor 错误设置）

### 9.4 DashSegmentsFD 直播错误与分派层重复

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
| DASH 非直播 | http_dash_segments | **DashSegmentsFD** | HttpFD |
| DASH 直播（默认） | http_dash_segments | **FFmpegFD**（强制） | ffmpeg 内部 (-re) |
| DASH + `--downloader native` + 直播 | http_dash_segments | DashSegmentsFD → **报错退出** | - |
| DASH + `--downloader ffmpeg` | http_dash_segments | **FFmpegFD**（用户指定） | ffmpeg 内部 |
| DASH 直播回放 (`--live-from-start`) | http_dash_segments_generator | **DashSegmentsFD** | HttpFD |
| 多格式下载器不统一 | A+B | None → **路径拆分** | 逐轨各自 |
| 多格式全指向 FFmpeg 且可合并 | 任意组合 | **FFmpegFD**（直接合并） | ffmpeg 多输入 |
| 分段截取（任意协议） | 任意 | **FFmpegFD**（强制） | ffmpeg -ss/-t |

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
```

**一句话总结**：
> **HTTP 是基础**（所有分片最终都落到 HttpFD），**HLS 有原生/FFmpeg 两条路**（取决于直播/DRM/用户偏好），**DASH 非直播走原生、直播强制 FFmpeg**（用户指定 ffmpeg 时也直接 FFmpeg），**FFmpeg 是万能兜底+合并器**，**下载器不统一时逐轨拆分后分别下载再合并**。三者不是互斥关系，而是**层次 + 降级**的组合关系。
