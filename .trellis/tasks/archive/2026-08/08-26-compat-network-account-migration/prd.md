# 实现兼容网络与安全账号迁移

## Goal

作为中间父任务，协调大陆无外部代理 P0 网络能力和安全剪贴板账号迁移，保持两者都不削弱严格 TLS 与账号存储；网络 leaf 是产品基础 Gate，clipboard 仍是独立可选迁移能力。

## Confirmed Facts

- 该范围已拆为restricted-compat-network与secure-clipboard-account-migration。
- 用户已把大陆用户无需外部代理/VPN使用 Pixiv 列为产品目标；兼容网络因此从后置可选项提升为当前 Ugoira 边界之后的下一项基础任务。剪贴板迁移保留可见复制/粘贴UX但不能沿用硬编码AES。
- 两项都跨越Android平台与安全边界，必须分别威胁建模和验收。

## Dependencies

- 08-26-pixiv-network-token-refresh、oauth-pkce-webview-login、secure-account-store与android-platform-parity完成。

## Requirements

- R1: 本中间父任务不直接实现产品代码，只维护两个叶子任务的严格TLS、秘密生命周期、决策 Gate 和集成验收。
- R2: `restricted-compat-network` 使用 exact-host shared transport、direct-first 与 DoH/original-host TLS，覆盖 API/OAuth/image/download 四个 Pixiv 自有主机；不得成为通用代理或影响非Pixiv流量。
- R3: secure-clipboard-account-migration不得把transfer payload/secret持久化或声称无法提供的端到端安全。
- R4: 两个叶子分别完成威胁模型、实时能力核验、用户审批、实现、验证、提交和归档。
- R5: 完成前验证 system proxy/VPN off 下的 OAuth/API/pximg/download/WebView、direct-first/route isolation/WebView清理，以及账号导入/过期/篡改和clipboard自动清除。
- R6: 第三方的 TLS override、反代和 BODY 日志仅作为拒绝语料；省 SNI 在自行完成链验证与 SAN 核对时允许，固定 IP 仅作兜底。Clipboard 信封是透明传输格式，只承诺 accidental corruption 检测，不宣称 authenticity/confidentiality——真实边界是服务端凭据校验。

## Acceptance Criteria

- [ ] 两个叶子均归档且安全边界有证据。
- [ ] `DirectOnly` 始终系统DNS+严格HTTPS；默认 `Automatic` 只在 eligible failure 后影响 exact Pixiv host，并始终保持原 hostname TLS。非Pixiv流量不进入 compatibility。
- [ ] 剪贴板payload有版本与SHA-256损坏检测，导入后只进入CredentialStore并及时清除。
- [ ] TLS/cert/host/clipboard攻击测试没有安全降级或伪成功。
- [ ] 网络反例审计和clipboard reader/writer威胁矩阵明确；checksum可重算这一事实与Pixiv服务端credential验证没有被包装成端到端安全。
- [ ] 系统 proxy/VPN off、无外部代理 App 的 API 36 真机按 OAuth/API/image/download/WebView 分出口记录；三运营商证据不足时只声明已测范围。
- [ ] 全量analyze/test/debug build和API36真机Login/clipboard集成通过。

## Out of Scope

- 全局VPN/通用代理。
- 二维码/公钥配对。
- 后台账号云同步。
- 通用代理/VPN、远程中继、第三方 Pixiv API/图片反代或大陆 100% 可用承诺。

## Risks and Deferred Items

- 零额外秘密的跨设备clipboard无法防御恶意clipboard reader；叶子任务必须明确威胁模型和残余风险。

## Source Anchors

- beta56 Android PlatformWebView、本地reverse proxy、Encrypt/LoginController
- 两个叶子task artifacts与父PRD R3/R5

## Open Questions

- Q1（已决议 2026-08-27，2026-08-29 修订）：允许 App 默认 direct-first、在 eligible failure 时自动使用仅 Pixiv 域的 DoH 与严格连接；“不使用代理”指不需要外部代理/VPN，范围为登录后使用。exact-host 与完整证书验证不变；ECH 与 WebView loopback 已从方案中删除。
