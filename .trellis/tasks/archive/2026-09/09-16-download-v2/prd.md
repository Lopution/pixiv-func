# 下载体系增强

## Goal

对齐 Shaft 下载体验：字节级断点续传、批量下载、caption 导出、命名模板扩展。

## Confirmed Facts

- `DownloadManager` 已有任务/组状态机、owner/destination 校验、durable recovery（记录 `receivedBytes`/`pendingMediaStoreId`/owner/finalUri）、`submitGroup` 组聚合状态。
- `DownloadTransport.open(url, headers, cancelToken)` 透传 headers——`Range` 由 manager 注入即可，无签名变更；206/200 语义在 manager 处理。
- 落点三态已有：MediaStore pending 行（不可见）、SAF 目标树文档（立即可见，abort 删除）、桌面 `<name>.part` staging（`_StagedFileSink`，崩溃不留下半成品最终名）。
- `_CoalescingChannelWriter` 缓冲 ≤256KiB——「已交给 sink 的字节」不等于「已落平台字节」，续传偏移必须以 sink 侧已冲刷数为准。
- 整页套图已有：`IllustDownloadCoordinator.downloadAllPages` → `submitGroup`。
- Shaft 参考：`BulkActions.startAuthorWorksBulkDownload`（作者作品页「下载全部」，先 `total_illusts` 控制入口可见，`FetchProgressDialog` 枚举进度，`bulkEnqueueIllusts` 入队）；`IllustCaptionExporter`（标题/作者/作者ID/作品ID/链接/标签/简介 txt）；`TemplateContext` 变量集。
- `userRepository.fetchWorks(userId, type:illust|manga, cursor)` 分页枚举作者作品；`IllustEntity` 提供 caption/tags/user.id/metaPages(w,h,original)/createDate，series 信息经 `series_store` 探测可选携带。
- `NamingRule` 现有变量 `{artist}{title}{id}{page}{ext}{date}`；「无 / 无正则」安全约束。

## Requirements

- HTTP `Range` 续传：失败/暂停/进程重启后从已落盘字节续传；服务器忽略 Range（回 200）则弃旧字节重来；平台侧实际字节数与记录不符同样弃旧重来。
- 暂停 ≠ 取消：`pause` 保留已写字节转 `retryable`；`cancel` 维持 abort 删除语义。恢复记录与 pending 锚点对续传任务保留（recovery 不再清理可续传记录的 pending 行）。
- 批量下载：作者页「下载全部」走 Shaft 语义——枚举 `user_illusts`+`user_manga` 全量（ugoira 排除），枚举进度可见，确认显示总数后 `submitGroup`；批量任务可暂停/取消/观察（任务页组卡片）。
- caption 导出：同名 `.txt`（page-0 渲染名去扩展）随图落盘，可选开关，按 (账号, illust, 目标) 去重只写一次；内容用 Shaft 信息头结构（标题/作者/作者ID/作品ID/链接/标签/简介）。
- 命名模板变量扩展：`{author_id} {pages} {page1} {series} {series_order} {chapters} {w} {h} {created}`；缺失元数据渲染为空串；保持无 `/` 无正则约束。
- 复用 DownloadManager ownership/recovery/revision 契约；不破坏 MediaStore/SAF 语义。

## Acceptance Criteria

- [ ] 中断（网络失败/用户暂停/进程重启）后重试从已下载字节续传；恢复记录可观测 resume 偏移与锚点。
- [ ] 作者页「下载全部」确认后批量入队；任务页可按组暂停/恢复/取消并看聚合进度。
- [ ] caption `.txt` 在开关开启时随下载落盘到同目标；重复下载同作品不产生重复 txt。
- [ ] 命名模板接受新变量并安全渲染（非法变量仍拒绝，`/` 永不出现）。
- [ ] sink/transport/recovery 测试覆盖续传、取消、暂停与组操作；Kotlin 通道单测覆盖 resume/rename。

## Out of Scope

- 收藏侧批量下载（Shaft `startBookmarkIllustBulkDownload`）——PRD 只要求作者侧。
- ugoira 作品批量（走独立 ugoira 导出流程）。
- 分段限速/多线程下载、aria2 式外部下载器。
- 下载完成通知/系统下载管理器集成。

## Dependencies

无。
