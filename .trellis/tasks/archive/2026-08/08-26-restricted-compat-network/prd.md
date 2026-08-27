# 实现受限兼容网络模式

## Goal

把“中国大陆用户无需安装、购买、配置或保持外部代理/VPN，仍能尽可能使用 Pixiv”实现为 Replica v1 的一等网络目标：所有 Pixiv 原生流量先走严格直连，只在受支持且可审计的条件下切换 Pixiv 域限定的兼容传输，始终保留原 URL hostname 的系统 CA、证书链和主机名校验，绝不成为通用代理。

“尽可能”表示以代表性大陆网络样本持续提高登录、API、图片、下载和 WebView 的成功覆盖，不承诺所有地区、运营商、时间点或 Pixiv 服务变化下 100% 可用。

## Confirmed Facts

- 用户已明确把大陆无外部代理/VPN可用性列为项目目标；“不使用代理”是否允许 App 内部自动使用 Pixiv 域限定的透明兼容传输，仍是本轮唯一产品边界问题。
- 当前仓库的 API/OAuth 使用独立 `http.Client`，下载使用 `dart:io HttpClient`，图片使用 `CachedNetworkImage`，登录使用 Android WebView；兼容能力必须统一覆盖这些真实出口，不能只改一个 HTTP client 后声称全 App 可用。
- 当前 Flutter 3.47 所带 Dart SDK 提供 `HttpClient.connectionFactory`，`package:http` 的 `IOClient` 可注入配置后的 `HttpClient`。实现前仍须用 spike 证明“连接解析出的 IP、再以原 hostname 建立 `SecureSocket`”在连接池、取消和证书校验下正确工作。
- pixez-flutter 的固定版本把 `standard`、ECH 与 compatibility transport 分开；ECH 路径保持证书校验，但 compatibility 路径关闭证书校验/SNI并依赖固定 IP。前者只作为 capability research，后者是明确拒绝语料。
- Pixiv-Shaft 提供稳定的 DNS/connect/timeout/TLS/cancel failure 分类和面向“无代理环境”的分目标诊断思路；其 trust-all、无/替换 SNI、固定 IP 和第三方反代路径不得沿用。
- 仅把 WebView 指向 loopback `CONNECT` 可以替 WebView 控制 DNS/目标 IP，但在端到端 TLS 下不会自动隐藏原始 ClientHello/SNI；若受阻点是 SNI，必须由经核验的 ECH/平台能力解决，否则保持明确失败。
- beta56 的本地 reverse proxy AAR、WebView `addDirect` 和 SSL error `proceed` 只能作为可见 UX 与失败现场参考；任何 TLS 绕过都禁止复刻。

## Dependencies

- `08-26-pixiv-network-token-refresh`、`08-26-android-platform-parity` 与 `08-26-oauth-pkce-webview-login` 已完成。
- 当前 `08-26-ugoira-player-export` 工作边界完成后，本任务提升为下一项基础实现候选；后续 Profile edit、Live 与 Widgets 复用同一 `NetworkAccessPolicy`。

## Requirements

- R1: Replica v1 必须以“无需外部代理/VPN”作为可观察结果；发布说明只能陈述有日期、设备和网络样本支撑的覆盖，不得把设计目标写成已实现或普遍可用。
- R2: 默认路由为 `Automatic`（direct-first），并保留 `DirectOnly`：任何请求先用系统 DNS + 直接 HTTPS；只有匹配 Pixiv destination registry 的请求才可能进入兼容候选。Q1 已允许 eligible failure 后的仅 Pixiv 域自动 fallback 与必要且可隔离的 WebView loopback transport，但实现仍必须通过各自 capability Gate。
- R3: destination registry 使用规范化后的精确 hostname/purpose 集合，至少区分 App API、OAuth、accounts Web、pximg、Pixiv Web 与经 Live 任务当天确认的 host；拒绝 suffix/IDN/IP/trailing-dot/userinfo/redirect/重绑定绕过。非 Pixiv 域、Updater、反向搜图 provider 和用户自定义 URL 永不进入兼容路由。
- R4: 建立稳定 failure taxonomy。DNS、connect、timeout、reset/route failure 可以触发另一个仍保持严格 TLS 的候选；取消、HTTP 状态、认证、限流、解析错误和证书/主机名不匹配不得触发安全降级。Mutation/Token exchange 不得在请求体可能已发送后自动重放，只能预选健康路由或显式重试。
- R5: 原生 API/OAuth/图片/下载/Ugoira 优先使用 host-scoped secure resolver + direct connector：系统 DNS 与经核验的加密 DNS 候选带来源、TTL、network identity 和取消信号；连接候选 IP 后，TLS SNI、Host、证书链与主机名校验仍使用原 URL hostname。
- R6: 把 ECH 作为独立 capability gate，而不是默认依赖或宣传项。实现前必须证明当前 Pixiv endpoint 发布/接受 ECH、候选 Android/Flutter transport 在 API 36 上 `require ECH` 时 fail closed、仍使用系统信任链，并能覆盖 API/OAuth/图片的取消、连接池和流式响应；证明失败则不引入该依赖。
- R7: WebView 先使用平台原生严格 HTTPS。在 Q1 已批准的 App 内部范围内，若 AndroidX WebKit reverse-bypass 能力可证明，loopback `CONNECT` 只能 bind 随机 loopback port、只接受 registry 中的 `:443`、由 secure resolver 选择 IP，并正确 set/clear/refcount；它只被声明为 DNS/route steering，不能被宣传为 SNI bypass。无法隔离或无法覆盖登录必需请求时明确提示/阻塞。
- R8: 任何候选都不得关闭证书校验、设置 `badCertificateCallback => true`、替换/清空 SNI、改写 Host、使用 domain fronting、全局 `HttpOverrides`、开启 cleartext，或把固定 IP/第三方反代当作默认安全核心。静态 endpoint 只有在未来单独批准的签名、过期、kill-switch 配置中才可研究。
- R9: resolver/health/circuit state 同时绑定 canonical host、network identity、IP family、route kind 和 TTL；网络切换、账号退出、模式切换和 dispose 会取消旧候选、关闭连接池并清理过期状态，不能把 Wi-Fi 成功候选带到蜂窝网络。
- R10: `PixivHttpClient`、`OAuthService`、`HttpDownloadTransport`、`CachedNetworkImage` 的 file service、Ugoira、WebView 与受证 headless Widget 必须从同一 policy/factory 取得路由；禁止 feature 自行替换 URL、Host 或 client。外部翻译、Updater 与反向搜图仍 direct。
- R11: 提供不含 URL query、cookie、token、响应 BODY 或完整私有地址的诊断：记录目的类别、route kind、DNS 来源、IP family、failure kind、耗时、WebView/ECH capability 与 network revision；用户可重试 direct、查看当前路径并关闭兼容行为。
- R12: 大陆验证矩阵必须在系统 proxy/VPN 关闭、未运行外部代理 App 的干净 Android 设备上覆盖 OAuth 登录、Token exchange/refresh、Recommended/Detail/Bookmark、pximg 图片与下载/Ugoira、accounts/WebView；记录日期、运营商/网络类型、IPv4/IPv6、Android/WebView 版本和实际 route。广泛“大陆可用”声明至少需要移动/联通/电信各一个当前样本；样本不足时只报告已测范围。

## Acceptance Criteria

- [ ] 干净 API 36 设备在系统 proxy/VPN 关闭且无外部代理 App 的条件下，完成至少一条真实 `Login → Recommended → Detail → Bookmark → image/download` 链路；自动化、单一真机样本和三运营商矩阵分层记录。
- [ ] `Automatic` 始终 direct-first；仅 eligible transport failure 选择下一个严格候选，HTTP/auth/rate-limit/parse/cancel/certificate mismatch 不触发降级，POST/mutation 不发生不确定自动重放。
- [ ] App API、OAuth、图片缓存、下载/Ugoira、WebView 与 Widget headless 的 route contract 有聚焦测试或明确 capability blocker；非 Pixiv/Updater/反向搜图流量从未进入兼容 connector/tunnel。
- [ ] secure DNS 的 bootstrap、响应类型、TTL、取消、network revision、IPv4/IPv6 与恶意回答处理可测试；连接选定 IP 时仍验证原 URL hostname，错误证书/host mismatch 必失败。
- [ ] ECH 只有在端点与 transport 设备 spike 通过后进入产物；`require ECH` 不可满足时 fail closed，不能静默回到关闭验证/SNI hack，也不能复制第三方固定 IP。
- [ ] 在已批准的 WebView loopback 范围内，host/port/header/rate/connection/loopback-only、reverse bypass、set/clear/refcount、进程重启与“不能隐藏 SNI”的边界均有证据；能力不满足时产物不包含该 listener/proxy override。
- [ ] 网络切换、模式切换、账号退出、页面 dispose 和取消会终止旧 resolver/probe/connection；日志与诊断不包含 secret、query、BODY 或完整用户配置地址。
- [ ] source/依赖/配置审计确认没有证书绕过、SNI/Host 替换、domain fronting、全局 `HttpOverrides`、通用代理、第三方图片反代或默认固定 IP。
- [ ] `flutter analyze`、全量 test、debug build、API 36 WebView 真机和有日期的大陆真实网络矩阵按真实结果记录；缺少样本或某类端点失败时保持 blocker/限制，不宣称普遍可用。

## Out of Scope

- 通用 HTTP/SOCKS/VPN、远程中继或用户自定义代理。
- MITM、自签 CA、TLS/certificate bypass、替换/清空 SNI、Host 改写或 domain fronting。
- 代理/改写非 Pixiv 流量，或用第三方图片/API 反代偷换“Pixiv 直连”。
- 绕过账号、地区内容规则、验证码、Cloudflare challenge 或 Pixiv 服务端授权。
- 对所有大陆地区、运营商、设备和未来 Pixiv/WebView 版本作 100% 可用承诺。

## Risks and Deferred Items

- ECH endpoint/transport 与 Android WebView 能力会变化；本任务只能用当日实测能力选择路径，不能从第三方源码推断当前环境一定可用。
- `CachedNetworkImage`、OAuth、下载与 WebView 属于不同出口；任一未接入时都不能声称 App 级兼容完成。
- 大陆运营商真机样本需要外部网络条件；无法取得时实现与自动测试可以完成，但“大陆广泛验证”保持未完成。

## Source Anchors

- 当前仓库：`lib/core/network/pixiv_http_client.dart`、`lib/core/auth/oauth_service.dart`、`lib/core/download/pixiv_download_transport.dart`、`lib/app/pixiv_image.dart`、`lib/features/login/login_webview_page.dart`
- 开源审查：`../08-27-open-source-pixiv-app-plan-audit/research/source-evidence.md` 的大陆无外部代理连通性章节
- beta56 Android `PlatformWebView.kt` 与本地 reverse proxy 行为；只作为 UX/失败现场参考

## Open Questions

- Q1（已决议，2026-08-27）：用户允许无需外部代理/VPN的 App 内部方案：默认 `Automatic` direct-first、eligible failure 后的仅 Pixiv 域 DoH/ECH，以及必要且可隔离的 WebView 本机 loopback `CONNECT`。实现仍须通过 exact-host、能力、严格 TLS 和真实设备 Gate；不满足能力时明确失败，不改用不安全路径。
