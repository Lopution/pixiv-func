# 补强 Bookmark、Follow、Comments 写操作归属

## Goal

补强 Bookmark、Follow、Comments 及相关资料写操作的账号归属、去重、revision、限流分类和取消语义，确保失败写操作不会跨账号、跨页面或隐式后台重放。

## Scope and current facts

- 目标历史任务：`08-26-bookmark-state-sync`、`08-26-user-profile-follow`、`08-26-comments-replies`、`08-26-profile-edit`；具体代码以当前 `bookmark`, `comments`, `user` 和 profile modules 为准。
- 服务器确认结果仍是唯一 authoritative state；乐观 UI 只能表达 pending，不能把请求发出当作已收藏、已关注或已评论。
- 网络调用依赖 `08-26-restricted-compat-network` 的 account-aware client、strict TLS、network revision 和统一错误分类。

## Requirements

- R1：每个 mutation envelope 固定 `accountId`、credential revision、entity id、operation、client mutation id、createdAt 和 cancellation owner；切换账号或 dispose 后不可提交。
- R2：以 `(accountId, entityType, entityId, operation)` 去重；同一对象的相反操作标记 superseded，旧响应不得覆盖新 revision。
- R3：状态至少区分 `idle/pending/confirmed/failed/cancelled/superseded`；只有 server-confirmed 才能写共享 store，失败必须恢复可操作的 UI 状态。
- R4：401/invalid refresh、403、404、429（含 `Retry-After`）、网络不可达和可重试 5xx 分类可观察；非幂等评论/资料写操作不得被后台隐式重放。
- R5：退出、账号切换、token refresh 失败和页面生命周期结束时取消属于旧 owner 的未完成任务；禁止把 pending mutation 跨账号持久化恢复。
- R6：保持 beta56 的短按/长按收藏和评论体验，不把 optimistic shortcut 扩大为离线队列或全局重试服务。

## Acceptance Criteria

- [ ] 同一账号同一对象的快速重复收藏/关注只产生一次有效提交，反向操作能标记 superseded 且旧响应不回滚新状态。
- [ ] 账号 A 发起写操作后切换到账号 B，迟到响应不能更新 B 的 store/UI；退出或 dispose 后任务终态为 cancelled/stale。
- [ ] 429 的 `Retry-After`、认证失效和永久失败分别可观测；刷新 token 只按共享 policy 执行，不触发非幂等操作重放。
- [ ] server-confirmed、失败恢复、取消、跨页面同步和多账号隔离均有聚焦测试；真实 API/device 证据按四层状态记录。
- [ ] 归档目录保持无 diff，integration release 明确消费本叶子的 mutation ledger。

## Dependencies and Out of Scope

- 依赖：`08-26-restricted-compat-network`；资料编辑的 beta56 字段和权限仍由 `08-26-profile-edit` 约束。
- 不负责新增社交功能、改变 API 合约、建立离线 durable mutation queue 或复制第三方客户端的状态库。
