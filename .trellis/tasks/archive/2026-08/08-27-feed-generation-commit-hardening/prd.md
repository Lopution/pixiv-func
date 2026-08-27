# 补强 Feed generation 结果提交边界

## Goal

修复并验证 Recommended、Ranking、New、Search、Profile feed 的 generation-scoped 网络、解析、共享实体和 cursor 提交边界，防止 refresh、筛选或账号切换后旧请求污染当前状态。

## Scope and current facts

- 目标历史任务：`08-26-recommended-feed-paging`、`08-26-ranking-feed`、`08-26-new-content-feeds`、`08-26-search-catalog`、`08-26-user-profile-follow`。
- 复核重点是当前 `PagedFeedController`、各 feed controller/repository、`IllustStore`/`UserStore` 以及分页测试；归档目录只作缺口证据，不直接修改。
- 这是网络消费者，依赖 `08-26-restricted-compat-network` 的 exact-host、direct-first、严格 TLS 和 network revision 契约。

## Requirements

- R1：每个请求携带不可伪造的 `FeedRequestContext`，至少包括 feed key、account/credential revision、generation、page/cursor 和 cancellation handle。
- R2：网络响应、解析结果、共享 entity merge、去重、cursor 更新和 loading/error 状态必须以同一 context 提交；context 非 active 时只能丢弃并记录可观察的 stale/cancel 结果。
- R3：refresh 必须使旧 append 失效并赢得最终状态；账号、搜索词、筛选、排序或 endpoint 改变时不能复用旧 cursor、entity ownership 或 loading 状态。
- R4：同 ID 作品/用户去重保持稳定顺序；服务端同 ID 更新、删除、重排和新增都必须反映在当前 generation，不产生幽灵项。
- R5：`next_url`/cursor 仅接受 repository 的 allowlist/解析结果；恶意 host、过期 revision、重复 cursor 和取消不应改变当前 feed。
- R6：保留 beta56 的可见分页、错误提示和空态；不引入无界缓存、全局静态 store 或隐式重试。

## Acceptance Criteria

- [ ] refresh 与 append 并发时，旧响应不能写入当前列表、`IllustStore`/`UserStore`、cursor 或 terminal error。
- [ ] 账号切换、退出、dispose、搜索词/筛选切换后，旧请求只产生可观察 cancellation/stale 记录，不改变新 context。
- [ ] 覆盖相同 ID 更新、重复 ID、删除、重排和 disjoint page 的 merge 测试，结果顺序和 cursor 可预测。
- [ ] 分页错误独立于已有内容；重试不会重复写入实体或推进错误 cursor。
- [ ] 目标 feed 的聚焦测试、`flutter analyze` 和适用的 debug/device 验证分层记录；未达成的层级保留 blocker。
- [ ] 归档任务无 diff，且最终集成矩阵引用本叶子的 evidence ledger。

## Dependencies and Out of Scope

- 依赖：`08-26-restricted-compat-network`；在 shared transport contract 未验收前不启动本叶子。
- 不负责重新设计 feed UI、缓存策略、Pixiv endpoint 发现或直接修复归档源码；跨层共享 API 变更须回写本叶子 design 和集成任务。
