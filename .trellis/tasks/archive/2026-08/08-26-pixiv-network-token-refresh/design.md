# 建立 Pixiv 网络客户端与 Token 刷新 — Design

## Objective

提供所有业务 feature 共用的严格、可取消、可观测 Pixiv 网络层，并保证并发认证失败只触发一次按账号刷新。

## Architecture and Boundaries

- PixivClientIdentity 是不可变配置提供者；时效值来自显式配置模块并附核验来源，不暴露给 feature。
- PixivHttpClient 包装 transport、headers、AccountStore snapshot、错误映射和 refresh/retry 协调。
- TokenRefreshGate 使用 Map<AccountId, Future<RefreshResult>>，在 finally 中仅清理自身 Future，避免旧任务删除新 gate。
- NextPageRequestParser 将 URL 降解为允许的 endpoint + typed query，由 client 重新构造 base URL。

## Data Flow

feature repository → PixivHttpClient(account/token snapshot) → response → auth classifier → compare current token → per-account gate → retry once | re-auth/error。

## Compatibility, Security, and Migration

- Compatibility transport 后续通过 NetworkRoutePolicy 注入，不改变 feature/client public contract。
- 取消信号贯穿 refresh 等待；取消业务请求不取消被其他请求共享的 refresh。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

