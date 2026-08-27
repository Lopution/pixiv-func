# 实现受限兼容网络模式 — Design

## Objective

以统一、host-scoped、direct-first 的 transport policy 覆盖 API、OAuth、pximg、下载和 WebView，使大陆用户无需外部代理/VPN，同时把 DNS 污染、SNI 阻断、平台能力缺失和 TLS 失败区分为不同问题，不用安全降级制造“可用”。

## Architecture and Boundaries

- `PixivDestinationRegistry`：把 canonical hostname 映射到 `api/oauth/accountsWeb/pixivWeb/image/live` purpose、允许端口和可用 route；只接受 exact host，不向 feature 暴露任意 host 注册接口。
- `NetworkAccessPolicy`：默认 `automatic`（direct-first），并提供 `directOnly`；Q1 已批准 App 内部的 Pixiv 域自动 fallback，但每个候选仍需通过自身 capability/严格 TLS Gate。每个请求生成 `NetworkRevision + RouteAttemptId`。
- `TransportFailureClassifier`：稳定区分 cancel、DNS、timeout、connect/reset、TLS/certificate、HTTP/auth/rate-limit/parse；只有允许类别进入下一个仍严格验证的 route。
- `SecureResolver`：系统 DNS 与 approved DoH 分开采样，验证 A/AAAA、公网地址、answer/TTL/size 上限，结果绑定 host/network revision；DoH bootstrap 也必须严格 TLS且不能递归依赖被污染的同一解析。
- `NativeStrictConnector`：实现 spike 时优先用 `HttpClient.connectionFactory` 连接 candidate IP，再以 canonical URL host 创建 `SecureSocket`；产出可注入 `IOClient`、OAuth/API、Download 与 image file service 的共享 factory。不得设置全局 `HttpOverrides`。
- `EchTransportAdapter`：可选、capability-gated 的 host-scoped transport。只有 endpoint ECH、`require ECH`、system trust、stream/cancel/pool 和 API 36 device spike 全部通过才进入依赖图；否则 adapter 不存在于 production graph。
- `WebViewRouteSession`：平台默认直连优先；在 Q1 已批准的 App 内部范围且 AndroidX WebKit reverse-bypass 可证明时，才为精确 host 设置 loopback `CONNECT`。listener 只做 DNS/目标 IP steering，不终止 TLS、不读取 HTTP body，也不声称隐藏 SNI。
- `NetworkDiagnostics`：只输出 route/failure/latency/capability/network revision 的脱敏记录，不保存请求 URL、query、header、token、cookie、BODY 或完整用户地址。

## Route Data Flow

读取型原生请求：

`destination validation → system direct HTTPS → eligible transport failure → secure DNS candidates → candidate IP + original-host TLS → optional require-ECH transport → response/error`

Mutation/OAuth token exchange：

`destination validation → current host/network health or strict preflight chooses one route → send body once → response/error`

请求体可能已发送后不得把 mutation 自动重放到另一条 route。GET/HEAD 的 route retry 也有 attempt 上限和同一 request cancellation。

WebView：

`platform direct → (approved scope + capability + exact host only) WebView reverse-bypass → loopback CONNECT:443 → secure resolver candidate → opaque TLS tunnel`

loopback 只改变解析/连接 IP。端到端 ClientHello 仍来自 WebView，所以没有 WebView/endpoint ECH 证据时，SNI 阻断必须显示为明确限制。

## Shared Transport Integration

- `PixivHttpClient` 与 `OAuthService` 注入同一 `PixivHttpClientFactory`，Token refresh 不另建默认 client。
- `HttpDownloadTransport` 复用相同 connector、route health 和 destination registry，同时保留每跳 redirect allowlist。
- `CachedNetworkImage` 使用 project-owned `HttpFileService/ImageProvider` 注入受限 transport；不靠全局 override 影响其他图片域。
- Ugoira ZIP、头像/插画、Profile edit、Live 与 headless Widget 消费同一 policy；Updater、翻译、反向搜图 provider 永远绕过 Pixiv compatibility policy并按自己的 allowlist direct。
- Host set 只能来自集中 registry 与经 owning task 证实的 endpoint；`next_url`、redirect 或远端 JSON 不能扩大它。

## Failure, Health, and Lifecycle

- health key 为 `(networkRevision, canonicalHost, ipFamily, routeKind)`，含正/负 TTL、最近严格握手结果与 circuit cooldown；不保存账号或请求内容。
- Network callback 变化、模式变化、账号退出和 dispose 会推进 revision、取消 probe/resolver、关闭旧 pool 并清除 WebView override。
- certificate/hostname mismatch 永不标记为“候选不健康后换不验证路径”；可尝试另一个仍严格验证的 endpoint/route，但所有候选失败后返回 TLS error。
- captive portal、private/reserved DoH answer、过长 DNS/CONNECT 输入和 response rebinding fail closed。

## Compatibility, Security, and Migration

- beta56 的可见网络入口可映射为诊断/模式入口；默认 `Automatic` 与 loopback 已获产品允许，但分别受 secure transport/WebKit capability Gate 约束。不会保留“SSL 错误继续”或旧 AAR。
- 从现有独立 clients 迁移时先引入 factory/registry，再逐个接入 API/OAuth/download/image/WebView；每一步测试未接入出口，避免出现一半兼容、一半默认 client 的伪完成。
- 不迁移第三方固定 IP、SNI 候选、OAuth identity、代理 URL、BODY 日志或 trust manager。ECH 依赖失败时回滚整个 optional adapter，不改 strict direct baseline。

## Important Trade-offs

- 自动 direct-first 能最大化零配置可用性，但失败时会把少量 Pixiv 域 DNS 查询交给经审计的加密 DNS；因此提供 `DirectOnly` 和脱敏状态，默认行为已按 Q1 决议固定。
- 原生 custom connector 能解决 DNS 污染，但不能解决所有 SNI 阻断；ECH 是更安全的候选，但 endpoint/transport/WebView 支持不可假设。
- 不用 SNI 替换、证书关闭或第三方反代意味着部分网络仍会失败；这是严格 TLS 与真实完成声明的边界。

## Rollback

- route factory、resolver、optional ECH 与 WebView session 分层提交；任一 capability 失败可移除对应 adapter，`DirectOnly` 仍使用原始严格 HTTPS。
- 关闭 compatibility 会清除 health/DoH/WebView state并重建 direct pools；不迁移或删除账号、下载和业务数据。
- 不重写历史、不覆盖当前 Ugoira 或其他无关工作树修改。
