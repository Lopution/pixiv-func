# Feed generation 结果提交边界 — Design

## Context and owner

所有 feed controller 使用同一类请求上下文，但由各 controller 保留 beta56 的排序和显示策略。建议引入不可变 `FeedRequestContext`：`feedKey`、`accountId`、`credentialRevision`、`generation`、`page`、`cursor`、`networkRevision` 和取消信号。context 由 refresh/filter/account 变更生成，不能由迟到响应回填。

## Commit sequence

1. controller 创建 context 并发起 repository 请求。
2. repository 在 shared network policy 下请求并解析成 page DTO；不得在不知道 context 是否 active 时写 store。
3. controller 先检查 generation、account/revision、feed key、cursor 和 lifecycle，再以一次 commit transaction 合并 `IllustStore`/`UserStore`、去重当前 page、推进 cursor 并更新 UI 状态。
4. 检查失败的响应只产生 stale/cancel telemetry；不得清空当前列表，也不得推进 error cursor。

refresh 递增 generation 并取消旧 page；append 只能使用当前 generation 的 cursor。相同 ID 更新按服务端顺序覆盖字段，稳定排序由当前 page 保持，删除/重排不会留下旧 index。

## Failure and cancellation

- 取消与 stale 不显示为网络错误；真实网络、认证、限流和 malformed cursor 走 shared failure taxonomy。
- dispose、logout、account switch 都使 context inactive；repository 请求可取消，无法取消时 commit gate 仍必须阻断。
- cursor 只接受 allowlist host/route 和非重复值，拒绝 `next_url` 改 host 或跨 account revision。

## Test design

使用 fake repository 控制响应顺序，覆盖 refresh-before-append、append-before-refresh、same ID update/delete/reorder、account/filter switch、duplicate cursor 和 dispose。测试必须同时断言 controller state、shared store 和 cursor，而不只断言列表长度。

## Integration and rollback

公开一个最小 commit gate 或 request context API，供 ranking/new/search/profile 复用；不引入全局缓存。若 rollout 发现兼容性问题，可关闭新 commit gate 回到旧 UI 行为，但保留 stale telemetry 和测试以便定位。
