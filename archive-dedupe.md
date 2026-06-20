# yt-dlp 下载归档（Download Archive）去重语义分析

## 1. 归档记录生成

### 1.1 归档 ID 格式

归档 ID 由 [make_archive_id](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/utils/_utils.py#L5299-L5301) 生成，格式为：

```
{extractor_key小写} {video_id}
```

例如：`youtube dQw4w9WgXcQ`

### 1.2 归档 ID 的构造流程

[_make_archive_id](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L3848-L3866) 负责从 info_dict 构造归档 ID：

1. 从 `info_dict['id']` 获取 video_id，为空则返回 None
2. 提取器 key 来源优先级：
   - `info_dict['extractor_key']`（正常提取结果）
   - `info_dict['ie_key']`（playlist 中的条目）
   - 若都为空，尝试通过 `info_dict['url']` 匹配已注册的 extractor 获取 ie_key
3. 调用 `make_archive_id(extractor, video_id)` 生成最终 ID

### 1.3 归档记录的写入

[record_download_archive](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L3876-L3887) 负责写入归档：

1. 若 `download_archive` 参数为 None，直接返回
2. 构造 vid_id（断言非空）
3. 若 `download_archive` 是文件路径：
   - 使用 `locked_file` 以追加模式（`'a'`）打开文件
   - 写入 `vid_id + '\n'`
4. 同时将 vid_id 添加到内存中的 `self.archive` 集合（用于同一次运行中的快速去重）

### 1.4 归档文件的预加载

[preload_download_archive](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L839-L857) 在 YoutubeDL 初始化时执行：

1. 若 fn 为 None，返回空 set
2. 若 fn 不是路径（即已是 set 对象），直接返回
3. 否则逐行读取文件，`strip()` 后加入 set
4. 文件不存在（ENOENT）时忽略，其他 OSError 向上抛出

---

## 2. 重复跳过逻辑

### 2.1 in_download_archive 判定

[in_download_archive](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L3868-L3874)：

```python
def in_download_archive(self, info_dict):
    if not self.archive:
        return False
    vid_ids = [self._make_archive_id(info_dict)]
    vid_ids.extend(info_dict.get('_old_archive_ids') or [])
    return any(id_ in self.archive for id_ in vid_ids)
```

关键点：
- `self.archive` 为空时直接返回 False（不启用归档）
- 除了当前 ID，还会检查 `_old_archive_ids` 列表中的**旧版兼容 ID**
- 任一 ID 存在于归档中即判定为重复

`_old_archive_ids` 由各 extractor 自行设置（如 zdf.py），用于兼容 extractor 升级后 video_id 规则变化的场景。

### 2.2 两处跳过检查点

#### 检查点 1：URL 级早期检查（提取前）

[extract_info](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L1708-L1714)：

```python
temp_id = ie.get_temp_id(url)
if temp_id is not None and self.in_download_archive({'id': temp_id, 'ie_key': key}):
    # 输出已归档提示，若 break_on_existing 则抛出 ExistingVideoReached
    break
```

- 使用 extractor 的 `get_temp_id(url)`（无需完整提取）从 URL 中快速解析出临时 ID
- 命中归档时直接跳过，避免不必要的网络请求和信息提取

#### 检查点 2：info_dict 级检查（提取后）

[_match_entry 流程](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L1645-L1663)：

- 在完整提取 info_dict 并填充字段后，调用 `in_download_archive(info_dict)`
- 会使用完整的 extractor_key 和 id，以及 `_old_archive_ids` 进行匹配
- 同样支持 `break_on_existing` 终止整个队列

---

## 3. 失败写入时机（何时写入/不写入归档）

### 3.1 `__write_download_archive` 标记

每个 format 的 info_dict 通过 `__write_download_archive` 字段传递写入意愿，取值：

| 值 | 含义 |
|---|---|
| `False`（默认） | 不写入 |
| `True` | 可以写入 |
| `'ignore'` | 被过滤规则排除，不参与决策 |

### 3.2 标记 True 的设置时机

在 [process_info](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L3331-L3673) 中：

| 场景 | 代码位置 | 条件 |
|---|---|---|
| 模拟下载 | L3371 | `simulate=True` 且 `force_write_download_archive=True` |
| 跳过下载 | L3457 | `skip_download=True` 且 `force_write_download_archive=True` |
| 正常下载+后处理全部成功 | L3667 | 下载成功、后处理（post_process）无异常、post_hooks 无异常 |
| 强制写入兜底 | L3670-3671 | `force_write_download_archive=True` 覆盖任何情况 |

标记 `'ignore'` 的设置时机：
- [L3340-3342](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L3340-L3342)：`_match_entry` 返回非 None（视频被 match_filter 等规则排除）

### 3.3 多格式下载的最终决策

在 [process_video_result](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L3140-L3143)：

```python
write_archive = {f.get('__write_download_archive', False) for f in downloaded_formats}
assert write_archive.issubset({True, False, 'ignore'})
if True in write_archive and False not in write_archive:
    self.record_download_archive(info_dict)
```

决策逻辑（对所有 requested formats 的投票结果取集合）：
- **必须存在至少一个 `True`**（有格式愿意写入）
- **绝对不能存在 `False`**（所有格式都不能失败）
- `'ignore'` 不影响结果（可存在可不存在）

即：**全部格式成功，且至少一个格式明确要求写入 → 写入归档**。只要有一个格式失败（False），整个视频就不会被记录。

### 3.4 process_info 中导致不写入的提前 return（失败场景）

以下场景在 `__write_download_archive` 被设为 True 之前就 return，对应 format 会保持默认 `False`，进而导致多格式决策时出现 False：

| 场景 | 代码位置 |
|---|---|
| `full_filename` 为 None（准备文件名失败） | L3375-3376 |
| 最终文件目录创建失败 | L3377-3378 |
| 临时文件目录创建失败 | L3379-3380 |
| 描述文件写入失败（返回 None） | L3382-3384 |
| 字幕写入失败（返回 None） | L3386-3388 |
| 缩略图写入失败（返回 None） | L3391-3394 |
| info.json 写入失败（返回 None） | L3398-3404 |
| 互联网快捷方式文件写入失败 | L3445-3447 |
| 分段下载需要 ffmpeg 但不可用 | L3477-3480 |
| 多格式合并时 ffmpeg 不可用且非 ignoreerrors | L3534-3538 |
| 多格式单独下载时目录创建失败 | L3557-3558 |
| 下载过程中网络异常（network_exceptions） | L3587-3589 |
| 下载内容过短（ContentTooShortError） | L3592-3594 |
| 后处理失败（PostProcessingError） | L3658-3660 |
| post_hooks 执行异常 | L3661-3666 |

此外，`success` 标志在下载过程中若变为 False，虽然不会提前 return，但后续的 `if success and full_filename != '-'` 分支（L3597）不会进入，因此也不会到达 L3667 设置 True。

### 3.5 扁平提取（extract_flat）场景

在 [process_ie_result](file:///d:/fz/0601-2/solo-dogfeeding/code/96-yt-dlp/yt_dlp/YoutubeDL.py#L1935-L1936)：

```python
if self.params.get('force_write_download_archive', False):
    self.record_download_archive(info_copy)
```

当使用 `extract_flat` 且 `force_write_download_archive=True` 时，即使未实际下载，也会直接将扁平提取的 url 条目写入归档。

---

## 4. 总结

归档写入遵循 **"全成功才写入"** 原则：

1. **预加载**：启动时将归档文件读入内存 set
2. **双重检查**：提取前（temp_id）和提取后（完整 info_dict + _old_archive_ids）各检查一次
3. **逐格式标记**：每个下载格式独立标记 `__write_download_archive`，只有下载+后处理+hooks 全部成功才标记 True
4. **全局决策**：所有格式的标记取集合，存在 True 且不存在 False → 写入归档（同时写文件和内存 set）
5. **force_write_download_archive**：可在 simulate / skip_download / extract_flat / 兜底覆盖等场景强制写入
