# 实现 Pixiv OAuth PKCE WebView 登录

## Goal

在保留 beta56 Login 可见体验的同时，用严格 TLS 和一次性 PKCE 完成真实 Pixiv 登录并安全写入 AccountStore。

## Confirmed Facts

- 当前 LoginPage 的注册、登录和剪贴板回调默认为空操作。
- beta56 LoginWebViewController 构造 app-api authorize URL 并截获 pixiv://account，但 verifier 生命周期、callback 白名单和现代 WebView 安全需要重做。
- 客户端身份和 OAuth endpoint 具有时效性；实现开始时必须从当前可信主源重新核验，不把历史值写进规划文件。

## Dependencies

- 08-26-secure-account-store 完成。
- 正常 token exchange 可以先使用专用严格 TLS transport；完整业务网络客户端由后续任务接管。

## Requirements

- R1: 用密码学安全随机数生成 RFC 7636 合规 verifier，计算 SHA-256、base64url 无填充的 S256 challenge。
- R2: 每次登录只有一个带 TTL 的 PKCE session；verifier 只能消费一次，并在成功、失败、取消、超时和 dispose 后清除。
- R3: WebView 打开经核验的 Pixiv authorize URL，只拦截 scheme=pixiv、host=account 且唯一有效 code 的回调；拒绝重复 code、额外歧义值和其他 URI。
- R4: TLS/证书错误必须失败；不得通过 JavaScript、bridge、日志或剪贴板读取/暴露账号密码。
- R5: token exchange 使用严格 host allowlist、超时和错误分类；成功后先安全保存凭据，再更新 AccountStore/current account 并导航 Home。
- R6: 注册/登录按钮、help 展开和首次登录标题位置保持当前 Replica shell；加载、取消、失败和重试状态必须可见。

## Acceptance Criteria

- [ ] PKCE challenge 对标准测试向量正确，verifier 随机性、长度、一次性和 TTL 测试通过。
- [ ] 只有精确 pixiv://account?code=... 可以进入 exchange；恶意、重复、空 code 和非白名单导航均被拒绝。
- [ ] 证书错误、用户取消、超时、exchange 失败都清理 session，且不会创建半成品账号。
- [ ] 真实 Android WebView 完成一次登录，重启后 AccountStore 恢复当前账号并进入 Home。
- [ ] 登录流程中日志、WebView bridge 和普通存储不出现密码、verifier、token 或 cookie。

## Out of Scope

- 兼容网络代理。
- 剪贴板账号迁移。
- 业务 API 请求和自动 Token refresh。
- 验证码或绕过 Pixiv 安全控制。

## Risks and Deferred Items

- Pixiv OAuth URL、客户端身份或 WebView 页面会变化；开始实现前的实时核验是硬前置，失效时明确阻塞而非恢复旧抓密逻辑。

## Source Anchors

- beta56 lib/pages/login/login.dart、lib/pages/login/web_view/controller.dart、lib/app/api/auth_client.dart
- 当前 lib/features/login/login_page.dart 与账号任务契约

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
