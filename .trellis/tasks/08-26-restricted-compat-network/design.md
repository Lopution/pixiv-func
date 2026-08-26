# 实现受限兼容网络模式 — Design

## Objective

在用户显式启用时，为受阻网络提供Pixiv限定的兼容路由，同时保持原始hostname端到端TLS且绝不成为通用代理。

## Architecture and Boundaries

- NetworkRoutePolicy选择Normal/Compatibility并输出typed destination，不让feature拼proxyURL。
- ResolverRace并发system/DoH、canonicalize域并返回TTL-bound candidates；TunnelConnector连接IP但保留hostname TLS。
- LoopbackConnectProxy是严格parser/allowlist/rate-limited service，无通用forwarding接口。
- WebViewProxySession以引用计数包装ProxyController set/clear并检查feature/reverse bypass。

## Data Flow

explicit compatibility → canonical host allowlist → DNS/DoH candidates → loopback CONNECT:443 → selected IP TCP → original host TLS/SNI/cert → Pixiv；disable→clear/stop。

## Compatibility, Security, and Migration

- Login UI保留本地反向代理开关，内部替换旧AAR/SSL proceed。
- 静态IP配置版本化且仅emergency，不能写死在业务层。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

