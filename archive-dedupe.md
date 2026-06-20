# yt-dlp 下载归档（Download Archive）去重语义分析

## 0. 核心决策流程总览

每个视频的下载格式（format）通过 `__write_download_archive` 字段传递写入意愿，最终在多格式决策层决定是否写入归档。

**标记取值**：
- `False`（默认）：不写入
- `True`：可以写入
- `'ignore'`：被过滤规则排除，不参与决策

**最终决策逻辑**（process_video_result 末尾）：
```
write_archive = {所有 format 的 __write_download_archive 标记}
if True in write_archive and False not in write_archive:
    record_download_archive(info_dict)  # 写入归档
```

即：**全部格式成功（没有 False），且至少一个格式明确要求写入（有 True）→ 写入归档**。只要有一个格式标记为 False，整个视频就不会被记录。

---

## 1. 归档记录生成

### 1.1 归档 ID 格式

由 `make_archive_id(ie, video_id)` 生成，格式为：
```
{extractor_key小写} {video_id}
```
例如：`youtube dQw4w9WgXcQ`

### 1.2 归档 ID 构造流程（_make_archive_id）

1. 从 `info_dict['id']` 获取 video_id，为空则返回 None
2. extractor_key 来源优先级：
   - `info_dict['extractor_key']`（正常提取结果）
   - `info_dict['ie_key']`（playlist 中的条目）
   - 若都为空，通过 `info_dict['url']` 匹配已注册的 extractor 推导
3. 调用 `make_archive_id(extractor, video_id)` 生成最终 ID

### 1.3 归档写入操作（record_download_archive）

1. 若 `download_archive` 参数为 None，直接返回
2. 构造 vid_id（断言非空）
3. 若 `download_archive` 是文件路径：
   - 使用 `locked_file` 以追加模式（`'a'`）打开文件
   - 写入 `vid_id + '\n'`
4. 同时将 vid_id 添加到内存中的 `self.archive` 集合（用于同一次运行中的快速去重）

### 1.4 归档预加载（preload_download_archive）

YoutubeDL 初始化时执行：
1. 若 fn 为 None，返回空 set
2. 若 fn 不是路径（即已是 set 对象），直接返回
3. 否则逐行读取文件，`strip()` 后加入 set
4. 文件不存在（ENOENT）时忽略，其他 OSError 向上抛出

---

## 2. 重复跳过逻辑

### 2.1 in_download_archive 判定

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
- 除了当前 ID，还检查 `_old_archive_ids` 列表中的旧版兼容 ID
- 任一 ID 存在于归档中即判定为重复

`_old_archive_ids` 由各 extractor 自行设置，用于兼容 extractor 升级后 video_id 规则变化的场景。

### 2.2 两处跳过检查点

**检查点 1：URL 级早期检查（提取前）**

在 `extract_info` 中匹配到 extractor 后：
```python
temp_id = ie.get_temp_id(url)
if temp_id is not None and self.in_download_archive({'id': temp_id, 'ie_key': key}):
    # 输出已归档提示，若 break_on_existing 则抛出 ExistingVideoReached
    break
```
- 使用 extractor 的 `get_temp_id(url)`（无需完整提取）从 URL 快速解析临时 ID
- 命中归档时直接跳过，避免不必要的网络请求和信息提取

**检查点 2：info_dict 级检查（提取后）**

在 `_match_entry` 流程中，完整提取 info_dict 并填充字段后：
- 调用 `in_download_archive(info_dict)`，使用完整的 extractor_key 和 id，以及 `_old_archive_ids`
- 同样支持 `break_on_existing` 终止整个队列

---

## 3. force_write_download_archive 的覆盖时机

`force_write_download_archive` 是强制写入归档的开关，但它的作用有明确边界，关键在于 **process_info 是否提前 return**。

### 3.1 两处设置位置

**位置 A（路径中设置）**：
- simulate 路径：`info_dict['__write_download_archive'] = force_write_download_archive`
- skip_download 路径：`info_dict['__write_download_archive'] = force_write_download_archive`

**位置 B（兜底覆盖）**：
在 process_info 函数的最后（L3670-3671）：
```python
if self.params.get('force_write_download_archive'):
    info_dict['__write_download_archive'] = True
```

**关键边界**：位置 B 只有在 process_info **没有提前 return** 时才会执行。

---

## 4. 四大场景详细分析

### 场景 1：跳过下载（skip_download=True）

**执行路径**（process_info）：
```
→ 前期准备（文件名、目录、描述、字幕等，任一失败则 return）
→ 进入 skip_download 分支
   → 执行 MoveFilesAfterDownloadPP
      → 若 PP 抛出异常且 ignoreerrors≠True → 异常向上抛出 → process_info 异常终止 → 标记保持 False
   → 设置 __write_download_archive = force_write_download_archive
   → （无 return，继续往下执行）
→ 执行位置 B（兜底覆盖）：
   若 force_write_download_archive=True → 标记强制设为 True
→ process_info 正常返回
→ 多格式决策
```

**写入结果**：

| 子场景 | force_write | 标记值 | 多格式决策 | 是否写入 |
|---|---|---|---|---|
| PP 成功 | False/None | False | 有 False | ❌ 不写入 |
| PP 成功 | True | True（兜底覆盖） | 无 False，有 True | ✅ 写入 |
| PP 失败（ignoreerrors=True） | True | True（兜底覆盖） | 无 False，有 True | ✅ 写入 |
| PP 失败（ignoreerrors≠True） | 任意 | False（异常退出） | 有 False | ❌ 不写入 |

---

### 场景 2：模拟下载（simulate=True）

**执行路径**：
```
→ 前期准备（文件名、目录、描述、字幕等，任一失败则 return）
→ 进入 simulate 分支
   → 设置 __write_download_archive = force_write_download_archive
   → check_max_downloads()
   → 【直接 return】 ← 关键点！
→ 【不执行位置 B（兜底覆盖）】
→ 多格式决策
```

**写入结果**：

| 子场景 | force_write | 标记值 | 多格式决策 | 是否写入 |
|---|---|---|---|---|
| 正常模拟 | False/None | False | 有 False | ❌ 不写入 |
| 正常模拟 | True | True | 无 False，有 True | ✅ 写入 |

**注意**：simulate 路径会 `return`，无法享受位置 B 的兜底覆盖，完全依赖位置 A 的设置。

---

### 场景 3：下载失败

下载失败分两种子场景：

#### 子场景 3a：下载抛出异常（network_exceptions、ContentTooShortError）

**执行路径**：
```
→ 前期准备（均成功）
→ 进入下载分支
   → try 块内执行下载
   → 捕获 network_exceptions / ContentTooShortError
   → report_error → 【return】 ← 提前退出
→ 【不执行位置 B】
→ 标记保持默认 False
→ 多格式决策：存在 False → 不写入
```

#### 子场景 3b：下载返回 success=False（无异常但失败，如重试耗尽）

**执行路径**：
```
→ 前期准备（均成功）
→ 进入下载分支
   → try 块内执行下载，返回 success=False
   → 不进入 if success 块（不设置 True）
   → try 块正常结束，无 return
→ 执行位置 B（兜底覆盖）：
   若 force_write_download_archive=True → 标记强制设为 True
→ process_info 正常返回
→ 多格式决策
```

**写入结果**：

| 子场景 | force_write | 标记值 | 多格式决策 | 是否写入 |
|---|---|---|---|---|
| 下载抛出异常 | 任意 | False（提前 return） | 有 False | ❌ 不写入 |
| 下载返回 success=False | False/None | False | 有 False | ❌ 不写入 |
| 下载返回 success=False | True | True（兜底覆盖） | 无 False，有 True | ✅ 写入 |

---

### 场景 4：后处理失败

包括 post_process 抛出 PostProcessingError、post_hooks 抛出异常。

**执行路径**：
```
→ 前期准备（均成功）
→ 进入下载分支
   → 下载成功（success=True）
   → 进入 if success 块
      → 执行 post_process
         → 抛出 PostProcessingError → 捕获 → report_error → 【return】
      → 或执行 post_hooks
         → 抛出 Exception → 捕获 → report_error → 【return】
→ 【不执行 L3667 的 True 设置】
→ 【不执行位置 B（兜底覆盖）】
→ 标记保持默认 False
→ 多格式决策：存在 False → 不写入
```

**写入结果**：

| 子场景 | force_write | 标记值 | 多格式决策 | 是否写入 |
|---|---|---|---|---|
| post_process 失败 | 任意 | False（提前 return） | 有 False | ❌ 不写入 |
| post_hooks 失败 | 任意 | False（提前 return） | 有 False | ❌ 不写入 |

---

## 5. after_video 阶段与归档写入的先后关系

### 5.1 执行顺序

在 `process_video_result` 中，当 `download=True` 时的关键执行顺序：

```
1. 循环处理每个 format，调用 process_info(new_info)
   → 每个 format 独立设置 __write_download_archive 标记

2. 多格式决策 & 写入归档（L3140-3143）
   write_archive = {f.get('__write_download_archive', False) for f in downloaded_formats}
   if True in write_archive and False not in write_archive:
       self.record_download_archive(info_dict)   ← 归档落盘

3. 运行 after_video 后处理（L3145-3146）
   info_dict['requested_downloads'] = downloaded_formats
   info_dict = self.run_all_pps('after_video', info_dict)  ← after_video 阶段

4. 若 max_downloads_reached 则抛出异常
```

**结论：归档写入发生在 after_video 阶段之前。**

### 5.2 after_video 阶段包含什么

after_video 是视频级别的最后一个后处理时机，在所有格式都下载处理完成后执行一次。

默认没有内置的 after_video 后处理器，用户可通过配置添加自定义 PP。

`run_all_pps('after_video', info)` 内部执行：
1. `_forceprint('after_video', info)` — 打印 after_video 时机的模板
2. 遍历所有注册在 `after_video` 时机的 PP，依次调用 `run_pp`

### 5.3 after_video 阶段失败时的归档状态

**after_video 阶段失败，归档已经落盘。**

具体分析：

| 失败场景 | 归档是否已写入 | 异常是否向上抛出 | 归档是否回滚 |
|---|---|---|---|
| after_video PP 抛出 PostProcessingError（ignoreerrors≠True） | ✅ 已写入 | ✅ 向上抛出 | ❌ 不回滚 |
| after_video PP 抛出 PostProcessingError（ignoreerrors=True） | ✅ 已写入 | ❌ 被 run_pp 捕获，继续 | ❌ 不回滚 |
| max_downloads_reached 抛出 MaxDownloadsReached | ✅ 已写入 | ✅ 向上抛出 | ❌ 不回滚 |

**关键原因**：
- `record_download_archive` 是同步写入文件的（locked_file + write + close），执行完毕即落盘
- 写入后没有任何回滚机制
- 即使 after_video 阶段随后失败并抛出异常，归档文件和内存中的 `self.archive` set 都已经包含了该记录

### 5.4 设计含义

这种"先归档、后 after_video"的顺序意味着：

1. **at-least-once 语义**：只要下载成功就归档，after_video 阶段的失败不影响归档记录。下次运行时该视频会被跳过。
2. **after_video 适合做"锦上添花"的操作**：如元数据上报、统计、通知等。这些操作失败不应影响视频已下载的事实。
3. **如果 after_video 失败需要重试**：不能依赖归档去重，需要手动清理归档记录或使用其他机制。

---

## 6. 其他提前 return 导致不写入的场景

以下场景均在位置 B 之前 return，无法享受兜底覆盖，标记保持 False：

| 场景 | 说明 |
|---|---|
| `_match_entry` 排除 | 被 match_filter 等规则过滤，标记设为 'ignore' |
| `full_filename` 为 None | 准备文件名失败 |
| 最终文件目录创建失败 | `_ensure_dir_exists(full_filename)` 返回 False |
| 临时文件目录创建失败 | `_ensure_dir_exists(temp_filename)` 返回 False |
| 描述文件写入失败 | `_write_description` 返回 None |
| 字幕写入失败 | `_write_subtitles` 返回 None |
| 缩略图写入失败 | `_write_thumbnails` 返回 None |
| info.json 写入失败 | `_write_info_json` 返回 None |
| 互联网快捷方式写入失败 | `_write_link_file` 返回 False |
| 分段下载需 ffmpeg 但不可用 | 非 FFmpegFD 且有 section_start/end |
| 多格式合并 ffmpeg 不可用 | 且 ignoreerrors≠True |
| 多格式单独下载目录创建失败 | `_ensure_dir_exists(fname)` 返回 False |
| 下载抛出网络异常 | network_exceptions |
| 下载内容过短 | ContentTooShortError |
| 后处理失败 | PostProcessingError |
| post_hooks 异常 | 任意 Exception |

---

## 7. 扁平提取（extract_flat）场景

在 `process_ie_result` 中，当使用 `extract_flat` 时：
```python
if self.params.get('force_write_download_archive', False):
    self.record_download_archive(info_copy)
```

- 即使未实际下载，也会直接将扁平提取的 url 条目写入归档
- 这里是**直接调用** `record_download_archive`，不经过 `__write_download_archive` 标记和多格式决策
- 仅当 `force_write_download_archive=True` 时执行

---

## 8. 总结：归档写入边界表

| 场景 | 无 force_write | 有 force_write | 能否被兜底覆盖 |
|---|---|---|---|
| 下载+后处理全部成功 | ✅ 写入 | ✅ 写入 | - |
| skip_download | ❌ 不写入 | ✅ 写入 | ✅ 能（除非 PP 抛异常） |
| simulate | ❌ 不写入 | ✅ 写入 | ❌ 不能（依赖位置 A） |
| 下载抛出异常 | ❌ 不写入 | ❌ 不写入 | ❌ 不能（提前 return） |
| 下载返回 success=False | ❌ 不写入 | ✅ 写入 | ✅ 能 |
| 后处理失败 | ❌ 不写入 | ❌ 不写入 | ❌ 不能（提前 return） |
| after_video 阶段失败 | ✅ 已写入 | ✅ 已写入 | -（归档已落盘） |
| 前期准备失败 | ❌ 不写入 | ❌ 不写入 | ❌ 不能（提前 return） |
| extract_flat | ❌ 不写入 | ✅ 写入 | -（直接调用） |

**核心原则**：
1. **正常流程**："全成功才写入"——所有格式的下载+后处理+post_hooks 全部成功才写入
2. **强制写入**：force_write_download_archive 能覆盖"无异常但失败"（success=False）和 skip_download 场景
3. **强制写入边界**：只要 process_info 因异常或错误提前 return，即使 force_write 也无法写入
4. **simulate 特殊**：提前 return 导致无法兜底，必须依赖路径中的显式设置
5. **after_video 在归档之后**：归档写入先于 after_video 阶段执行，after_video 失败不影响归档记录（已落盘，无回滚）
