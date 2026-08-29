# 实现兼容网络与安全账号迁移 — Design

## Objective

作为中间父任务，协调可选兼容网络和安全剪贴板账号迁移，保持两者都不削弱默认TLS与账号存储。

## Architecture and Boundaries

- 中间父任务只拥有NetworkRoutePolicy和TransferEnvelope安全契约，不作为实现task。
- 两个叶子共享Android capability/secure storage但不共享payload或代理内部状态。
- 安全审查和残余风险作为父acceptance evidence。

## Data Flow

Normal主链稳定 → compat leaf + clipboard leaf → login/account security integration → parent归档。

## Compatibility, Security, and Migration

- 保持beta56开关和复制粘贴入口，不恢复SSL proceed 或硬编码AES；固定IP 不作默认路由，只作解析失败后的兜底。
- 父任务不回滚独立叶子提交。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

