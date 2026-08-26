# 实现受限兼容网络模式

## Goal

在用户显式启用时，为受阻网络提供Pixiv限定的兼容路由，同时保持原始hostname端到端TLS且绝不成为通用代理。

## Confirmed Facts

- beta56本地reverse proxy AAR和WebView ProxyController会对本地代理配置addDirect，并在SSL错误时proceed；这些安全行为禁止沿用。
- 父PRD目标是system DNS + DoH race→选可用IP→local HTTP CONNECT tunnel→end-to-end TLS。
- 允许域仅*.pixiv.net与*.pximg.net，端口仅443；静态IP只可作为显式emergency fallback。

## Dependencies

- 08-26-pixiv-network-token-refresh与android-platform-parity完成。
- OAuth WebView使用统一NetworkRoutePolicy。

## Requirements

- R1: Normal为默认且始终system DNS/direct HTTPS；Compatibility必须用户显式开启并可随时关闭/清理。
- R2: 解析只接受规范化允许域，防止suffix/IDN/IP/点尾/重绑定绕过；CONNECT严格只允许443。
- R3: system DNS与经核验DoH resolver并发/竞速，结果按域/TTL缓存并健康探测；DoH失败不泄漏到任意resolver。
- R4: local tunnel仅bind loopback/随机端口，解析CONNECT request上限并拒绝非allowlist；不记录URL query、cookie或token。
- R5: 上游可连接选定IP，但TLS握手/SNI/Host和证书验证始终使用原始hostname；任何cert error失败。
- R6: WebView通过AndroidX WebKit ProxyController设置受限proxy与reverse bypass；能力不足或无法避免代理其他流量时安全失败提示。
- R7: 关闭/dispose/crash恢复时清除WebView override、停止listener和缓存敏感状态；多WebView引用计数正确。
- R8: 静态Pixiv IP仅在显式emergency选项且健康/证书验证后使用，不成为安全核心。
- R9: 实现host/port/rate/connection limits，防止本地代理被滥用或资源耗尽。

## Acceptance Criteria

- [ ] Normal流量路径与启用前一致，非Pixiv域在Compatibility下仍direct且不能通过CONNECT tunnel。
- [ ] allowlist绕过语料、CONNECT非443、恶意header/超长请求被拒绝。
- [ ] 连接IP时证书/SNI仍验证原host，错误证书/host mismatch必失败。
- [ ] WebView capability不足时不设置全局proxy；set/clear/refcount和进程重启验证通过。
- [ ] DNS/DoH race、TTL、失败/取消和静态emergency路径可测试且不泄密。
- [ ] analyze、全量test、debug build及受控网络/API36 WebView真机验证通过。

## Out of Scope

- 通用HTTP/SOCKS/VPN代理。
- MITM/自签CA/TLS bypass。
- 代理非Pixiv流量。
- 默认固定IP。

## Risks and Deferred Items

- WebView ProxyController reverse bypass能力随WebView版本变化；不能保证隔离时任务应block，不采用全局代理退化。

## Source Anchors

- beta56 Android platform/webview/PlatformWebView.kt与pixiv_local_reverse_proxy.aar行为
- 父PRD R5、NetworkRoutePolicy和AndroidWebKitCapabilities

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
