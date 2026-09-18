# Feed 基础设施：快照冷启动与离线动作队列

## Goal

对齐 Shaft `RoomFeedCacheBackend` + `actionqueue`：feed 快照持久化让冷启动首帧直出上次内容、后台刷新静默替换；收藏/关注等变更离线入队持久化，重连后经同一 repository 路径自动重放。

## Confirmed Facts

- `PagedFeedController`（`core/paging/`）是 15 个 feed 的共享基类；`PagedFeedState.ids` 是 `List<int>`，实体不进 state，走各类型 store（illust→`illustStore`，novel→`novelStore`）。`feedKey` 已是稳定标识。
- `IllustEntity` 有 `toJson()`/`fromJson` 双向序列化（API 字段形状往返）；`NovelEntity` 同型。
- 本仓 SQLite 惯例：一特性一库（`history.db` schemaVersion 2、`watchlater.db`），`HistoryDatabase` 提供 factory 注入测试模式。
- `pixiv_history_outbox` 已有离线持久化先例：`(account_id, content_type, content_id)` PK + `attempts`/`next_attempt_at`，串行 `flushOutbox`。
- `MutationLedger`（`core/mutation/`）只管在途身份/去重/丢弃 telemetry，**明确不持久化**——离线队列是其持久化补层，不重写。
- Shaft `actionqueue` 语义已确认：`ActionRequest(type, dedupeKey, payload, coalesce)`；同 owner+dedupeKey 的 PENDING 行入队时合并（收藏→取消→收藏只发最后一次意图）；`RetryScope` 分两档——429/5xx 整队冷却，4xx 只退避该行；owner=登录 uid，跨账号行永不重放。
- 仓库无 `connectivity_plus` 依赖。
- `FeedCommitGate.discardEvents`/`MutationDiscardEvent` 是既有 telemetry 范式（有界列表，可观测计数）。

## Requirements

- `feeds.db`（独立库，同一特性一库惯例）：`feed_snapshots` 表 `(account_id, feed_key)` 主键，列 `ids`(JSON)、`entities`(JSON)、`cursor`、`saved_at`、`snapshot_version`。
- `FeedSnapshotStore`：读/写/LRU 逐出；**过期阈值 24h**、**每 feed 最多 60 个 id**、**全库最多 64 个 feed 行**（超出按 saved_at 最旧逐出）。
- 冷启动路径：`PagedFeedController.build()` 在首个 fetch 前先查快照——命中则把实体 merge 进 store、`state` 直出快照 ids（`initialPhase: idle`），随后调度一次后台 `refresh()` 静默替换；过期/损坏快照删除后走正常首载。
- 快照写入：初始提交与 refresh 提交成功后各写一次（取 ids 前 60 + 对应实体）。
- 快照覆盖：**所有实体可序列化的 `PagedFeedController` 子类**（illust/novel id feed），通过可选 codec 挂钩接入；comment/spotlight 等不适配的子类默认关闭。
- `action_queue` 表 + `ActionQueue` 引擎（`core/actionqueue/`）：Shaft 语义——owner=accountId、dedupeKey coalesce、串行 drain、RetryScope 分档（网络/429/5xx→整队冷却；4xx→行级退避，`maxAttempts` 后标 failed 不再重放）。
- 首批入队动作：**bookmark add/delete + follow/unfollow**（PRD「收藏/关注等」）。调用路径：store 发 mutation → 仓库调用抛网络类 `ApiError` → 包成 envelope 入队并保 pending 态；重放走同一 repository 方法 → 确认/失败经既有 MutationLedger 状态路径。
- 重放触发：**不引入 connectivity_plus**——应用启动/回前台 + 任意同账号 repository 成功调用后 pump 一次（成功即连通证据）。
- Telemetry：`snapshotRestored`/`snapshotDiscarded`/`actionEnqueued`/`actionReplayed`/`actionDropped` 计数，走 discardEvents 同款有界列表范式。
- 账号边界：快照与队列行都按 accountId 隔离；切账号不重放、不读他人快照。

## Acceptance Criteria

- [ ] 冷启动有网：首帧渲染上次快照内容，后台 refresh 完成后替换为新数据。
- [ ] 冷启动无网：快照内容可见，初始阶段不报全屏错误。
- [ ] 快照过期（>24h）/JSON 损坏：行被删除，回退正常首载不崩。
- [ ] 快照按 (accountId, feedKey) 隔离；切账号后看不到另一账号快照。
- [ ] 离线点收藏/关注：动作入队可见 pending 态；恢复连通后经同一 repository 重放成功；重复重放幂等。
- [ ] 同一目标连点（收藏→取消→收藏）coalesce 后只发最后一次意图。
- [ ] 429/5xx 触发整队冷却；单条 4xx 不冻结队列；超 maxAttempts 标 failed 可见。
- [ ] fake-clock + 假传输测试覆盖恢复/过期/重放/账号边界/coalesce/退避分档。

## Out of Scope

- history outbox 与 action_queue 合并（历史上报已有自己的 outbox，保留）。
- 搜索词/筛选态快照（feedKey 已含查询参数，自然生效，不特化）。
- 快照实体的图片字节缓存（只存元数据，图片走既有 image cache）。
- 失败队列的用户级管理 UI（telemetry 计数即可观测，不做专门页面）。
- ugoira/评论等非收藏关注类变更的入队。

## Dependencies

无硬依赖。`watchlist-local-library` 的离线 watchlist 变更可复用 `ActionQueue`（该 child 排在后面自动受益，不阻塞）。

## 默认决策（待用户确认）

| 项 | 默认 | 备选 |
|---|---|---|
| 过期阈值 | 24h | 更短/可配置 |
| 容量 | 60 ids/feed、64 feeds LRU | 其他数值 |
| 快照覆盖 | 全部可序列化 PagedFeedController | 仅推荐+新着 |
| 重放触发 | 启动/前台+成功调用 pump | 引入 connectivity_plus |
| 首批动作 | bookmark + follow | 缩小到 bookmark |
