# Bookmark、Follow、Comments 写操作归属 — Design

## Operation envelope

每次写操作创建不可变 `MutationEnvelope`：`accountId`、`credentialRevision`、`entityType`、`entityId`、`operation`、`clientMutationId`、创建时间、policy/network revision 和 cancellation owner。store 维护 `(accountId, entityType, entityId)` 的 server state，UI 维护 mutation state，不用 pending 值冒充 confirmed。

## State machine

`idle -> pending -> confirmed`；失败转 `failed`，用户明确取消转 `cancelled`，被同一 owner 的相反操作取代转 `superseded`。每个 envelope 只能进入一次 terminal；迟到响应须比较 mutation id/revision 后丢弃。去重键按账号、实体和操作组合，避免多账号共享内存 key。

## Error and retry boundary

401/invalid refresh 交给 account-aware token policy，不能自动重放非幂等 body；403/404 为明确业务失败；429 保存 `Retry-After` 并等待用户/显式调度；网络/5xx 只按 operation 幂等性决定可重试。评论和 profile edit 默认不建立 durable offline queue。

账号切换、logout、页面 dispose 会取消 owner 下的 pending，并使旧 credential revision 失效。server-confirmed response 才能更新 Bookmark/Follow/Comments store；失败回滚 pending marker，不回滚其他账号的 state。

## Test design

fake transport 控制重复、反序、延迟和 token refresh；覆盖快速 toggle、相反操作 supersede、A/B 账号切换、dispose、429、401、网络失败和跨页面观察。测试同时断言 request count、mutation terminal 和 shared store。

## Compatibility and rollback

对既有 bookmark API 保持 adapter，先让新 envelope 包裹现有 repository；若 rollout 失败可关闭 optimistic rendering，但不能关闭 owner/revision 校验或恢复隐式重放。
