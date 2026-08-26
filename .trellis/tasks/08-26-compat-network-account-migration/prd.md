# 实现兼容网络与安全账号迁移

## Goal

作为中间父任务，协调可选兼容网络和安全剪贴板账号迁移，保持两者都不削弱默认TLS与账号存储。

## Confirmed Facts

- 该范围已拆为restricted-compat-network与secure-clipboard-account-migration。
- 兼容网络是后续能力，不阻塞Normal主链；剪贴板迁移保留可见复制/粘贴UX但不能沿用硬编码AES。
- 两项都跨越Android平台与安全边界，必须分别威胁建模和验收。

## Dependencies

- 08-26-pixiv-network-token-refresh、oauth-pkce-webview-login、secure-account-store与android-platform-parity完成。

## Requirements

- R1: 本中间父任务不直接实现产品代码，只维护两个叶子任务的默认关闭、严格TLS、秘密生命周期和集成验收。
- R2: restricted-compat-network不得成为默认或通用代理，不能影响非Pixiv流量。
- R3: secure-clipboard-account-migration不得把transfer payload/secret持久化或声称无法提供的端到端安全。
- R4: 两个叶子分别完成威胁模型、实时能力核验、用户审批、实现、验证、提交和归档。
- R5: 完成前验证Login Normal/Compatibility、WebView代理清理、账号导入/过期/篡改和clipboard自动清除。

## Acceptance Criteria

- [ ] 两个叶子均归档且安全边界有证据。
- [ ] Normal始终系统DNS+严格HTTPS；Compatibility只在用户显式选择时影响允许域。
- [ ] 剪贴板payload有版本/expiry/nonce/完整性检查，导入后只进入CredentialStore并及时清除。
- [ ] TLS/cert/host/clipboard攻击测试没有安全降级或伪成功。
- [ ] 全量analyze/test/debug build和API36真机Login/clipboard集成通过。

## Out of Scope

- 全局VPN/通用代理。
- 二维码/公钥配对。
- 后台账号云同步。

## Risks and Deferred Items

- 零额外秘密的跨设备clipboard无法防御恶意clipboard reader；叶子任务必须明确威胁模型和残余风险。

## Source Anchors

- beta56 Android PlatformWebView、本地reverse proxy、Encrypt/LoginController
- 两个叶子task artifacts与父PRD R3/R5

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
