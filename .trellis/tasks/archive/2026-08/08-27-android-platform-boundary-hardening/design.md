# Android 平台边界与 WebView 生命周期 — Design

## Capability session

WebView route 由版本化 `WebViewRouteSession` 表示：session id、account/credential revision、network revision、exact Pixiv host set、capability result、loopback owner/refcount、created/disposed timestamps。能力探测失败返回 typed blocker；不得在 transport 层猜测 WebView 可用。

Q1 允许的 loopback `CONNECT` 只能作为 WebView 私有、Pixiv 域限定的 route steering。它不隐藏 SNI、不改变证书验证，也不服务非 Pixiv host。页面销毁、background、logout、auth failure 和 app restart 都要清理 session/listener。

## Input and platform boundary

`intent_router` 先验证 action、scheme、host、path、MIME、URI grant 和大小，再映射为 typed route；未知输入显式拒绝。OAuth callback 还要匹配一次性 state/PKCE owner。FileProvider/MediaStore 使用 owner-tagged pending record，prepare/finalize/cancel/revoke 各自幂等，避免 exported/grant 扩大。

## Lifecycle and tests

平台 adapter 只暴露最小接口，业务层不直接操作 Android channel。测试 capability success/failure、double dispose、rotation/background、predictive back、deep link/SEND 非法输入、权限拒绝、pending cleanup 和 process restart；API 36 真机确认 WebView 版本、route 和安全日志。

## Rollback

loopback 与新 capability adapter 可按 capability gate 禁用并保留 direct WebView；不能以关闭证书/SNI 校验作为 fallback。FileProvider/MediaStore schema 变更需版本化，失败时清理新 owner 的记录而不触碰其他账号。
