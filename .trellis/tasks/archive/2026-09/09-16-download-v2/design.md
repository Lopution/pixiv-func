# 设计：下载体系增强

## 总览

```
caller ─ submit ─> DownloadManager ─ open(url, headers+Range?) ─> transport
                        │                                            │ 206/200
                        ├─ resumeOwned/beginOwned ─> sink(append) ─ write
                        │                                            │
                        └─ record{resumeBytes,resumeAnchor,pendingId} ─┴─> finalize
```

四块增量按 implement.md 顺序落地；「续传锚点」是唯一新概念——每个可续传任务持有一个平台侧半成品（MediaStore pending 行 / SAF `.part` 文档 / 桌面 `.part` 文件）+ 一条记录里的 `resumeBytes` 偏移。

## 1. Range 续传

**传输**：manager 在 `_run` 里对可续传 job 向 headers 注 `Range: bytes=<resumeFrom>-`（`PixivHeaders.image` 原样加头）。响应 `206` → 走 append sink；`200` → 服务器忽略 Range → abort 旧锚点、全新写；`416` → 旧锚点与服务器长度不一致 → abort 全新写（视为正常重下，非错误）。

**sink 契约**（`download_sink.dart`）：

- `DownloadSink` 新增只读 `storedBytes`（已冲刷到平台的字节数；`_CoalescingChannelWriter` 的 `_pendingLength` 不参与计数，flush 后才计入）。
- 可选接口 `ResumableDownloadSink`：`Future<ResumeAnchor> detach()` —— flush、保持半成品、返回 `{anchorId, storedBytes}`；之后 sink 关闭。
- 可选接口 `ResumableDownloadSinkFactory`：`Future<ResumedSink> resumeOwned(request, displayName, owner, ResumeAnchor anchor, {destination})` —— 返回 `(sink, existingBytes)`；`existingBytes != 期望` 时 manager abort 并退回全新 begin。
- `ResumeAnchor` = `{kind: mediaStorePending|safDoc|stagedFile, id/uri/path, bytes}`——恢复记录持久化的就是它。

**manager 行为**：

- `pause(taskId)`：running → `detach()` → 记录 `resumeBytes+anchor` → `retryable`；queued → `retryable`（无字节可续，retry 即重排）。
- 网络失败（非 cancel）→ 同 detach 路径落 `failed` + resume 字段；`retry` 优先走 resume 分支。
- 语义不变量：`cancel` 仍 abort 删除；`finalize` 成功后锚点失效（MediaStore 行转可见 / `.part` 改名）。
- `_recover`：带 `resumeBytes>0` 且 anchor 存活校验通过的非终态记录 → `retryable` 且**跳过** pending 清理；anchor 丢失/尺寸不符 → 清 pending + `retryable`（重下）。

**平台侧**：

- MediaStore：`resumePending(id, ownerId)` 通道——校验 pending+owner、`openOutputStream(uri,"wa")` 入 streams、`statSize` 回传。`listPending` 已给 owner 匹配。
- SAF：`rename(uri, displayName)` 通道（`DocumentsContract.renameDocument`）+ `resume(uri)`（"wa" 打开+大小）。begin 时落 `<final>.part`、finalize rename 去 `.part`——部分文件在用户树里始终带 `.part` 后缀，可见语义清晰。
- 桌面：`_StagedFileSink` 的 `.part` 已是 staging；resume 走 append 打开（现 `create` 会删旧 `.part`——resume 分支不删）。

**不可续传 sink**（MemorySink/legacy）：接口不实现即退化为原 abort 语义，测试与第三方适配层零改动。

## 2. 批量任务与组管理

- `IllustDownloadCoordinator.downloadAuthorWorks(userId, type)` → `AuthorWorksEnumerator`：`fetchWorks` 循环到 `nextUrl==null`，yield 每页 illust；跳过 `ugoira`/无 originalUrl 的作品；每作品展开 `metaPages[].original`（无 metaPages 时用 `metaSinglePageOriginalUrl`）。
- 入口：作者作品区 toolbar/overflow 「下载全部」——先拉 `total_illusts` 控制可见（Shaft 同），点按 → 枚举进度对话框（可取消枚举）→ 完成弹确认（作品数/页数）→ `submitGroup`（illust+manga 各一组）。硬上限 2000 作品，超出截断并在确认文案明示。
- manager 组操作：`pauseGroup(id)`/`resumeGroup(id)`/`cancelGroup(id)` = 逐子任务 pause/retry/cancel。
- 任务页：group 卡片（组名=任务名/首个 displayName 前缀，聚合进度=Σreceived/Σtotal，状态聚合用现有 `_aggregateGroupStatus`），操作：暂停/恢复/全部取消。

## 3. caption 导出

- `CaptionExporter`（core）：`export({required IllustEntity illust, required DownloadSubmissionContext ctx, required String pageZeroStem})`。
  - 去重：prefs set `caption_exported:{accountId}:{illustId}:{destination.identity}`；命中跳过。
  - 文件名：page-0 渲染名去 `.ext` + `.txt`（`12345_p0.txt`）。
  - 落盘：`DestinationAwareSinkFactory.beginRaw(displayName, 'text/plain', destination)`——新加穿透方法（MediaStore `begin` / SAF `create` / desktop 写文件），utf8 bytes → finalize；失败 abort 并向调用方抛错（页面 snackbar 可见，不静默）。
  - 内容：标题/作者/作者ID/作品ID/链接/标签/简介（Shaft `buildContent` 结构，标签取 `IllustTag.name`，简介取 `caption`）。
- 开关：`AppSettings.downloadCaption`（默认 false）+ 下载设置页开关；`downloadCaptionProvider`。
- 时机：提交时（downloadPage/downloadAllPages/author batch 路径在 coordinator 层统一触发），仅 `target==illustPage` 参与。

## 4. 命名变量扩展

`NamingRule.supportedVariables += {author_id, pages, page1, series, series_order, chapters, w, h, created}`；`DownloadRequest` 增可选元数据 `{authorId,totalPages,seriesTitle,seriesOrder,seriesTotal}`，宽高经现有 `width/height`（新增可选字段，page 级 metaPages 优先）。

- `{page1}`=`pageIndex+1`；`{pages}`=totalPages；`{created}`=createDate 解析 → `yyyyMMdd_HHmmss`；`{series}/{series_order}/{chapters}` 无元数据 → 空串（Shaft 同，空段由清理逻辑折叠）。
- 约束不变：`{name}` 白名单、无正则、`_clean` 去 `/`。

## 风险与回滚

- MediaStore pending 行跨进程保留依赖 IS_PENDING 行存活（官方支持自己 pending 行续写）；尺寸校验失败即弃旧重下，最坏退化为现状。
- SAF `.part` 命名改变落盘形态：旧版本写出半成品可见最终名，新版见 `.part`——语义更清楚，不破坏契约；rename 通道失败则该文档保留 `.part` 名并标失败可见。
- 每勾独立 revert；resume 相关全部为能力接口，旧 sink 行为不变。
