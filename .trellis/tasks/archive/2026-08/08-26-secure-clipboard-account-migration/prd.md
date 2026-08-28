# 实现安全剪贴板账号迁移

## Goal

保留beta56复制账号数据→另一设备粘贴登录的入口，同时去除硬编码AES并诚实约束clipboard传输威胁。

## Confirmed Facts

- beta56使用固定AES key/ECB加密整个Account JSON，Settings长按复制、Login帮助页粘贴导入。
- 父PRD要求credential at rest secure storage、version/timestamp/expiry/nonce/integrity、Android sensitive clipboard flag和短时自动清除。
- 在不增加独立秘密/配对通道时，clipboard内容无法抵御具有读取权限的恶意应用；不得声称端到端安全。

## Dependencies

- 08-26-secure-account-store、oauth-pkce-webview-login与android-platform-parity完成。

## Requirements

- R1: 保持Settings账号卡长按复制及Login help中的粘贴登录入口、反馈和导航节奏。
- R2: 定义版本化TransferEnvelope，包含createdAt/expiresAt、nonce、payload type/schema、最小账号metadata/credential和 corruption checksum；字段命名不得暗示不存在的认证加密。
- R3: 不使用固定/内置对称密钥、ECB或自制加密；在零额外交互下，unkeyed checksum 只能检测意外损坏/截断，nonce store 只能在目标设备上检测已导入 payload，不能证明发送者真实性或抵御恶意clipboard writer。
- R4: UI/文档必须明确clipboard reader/writer残余风险；账号真实性最终由导入后的Pixiv认证确认。若要求密码学机密性/真实性，必须新增口令/配对/公钥UX并重新取得产品批准，不能把密钥和密文一起放进clipboard后宣称安全。
- R5: Android写clipboard时设置sensitive flag，显示短期有效期，并在超时/成功导入/应用生命周期允许时只清除仍等于本app payload的内容。
- R6: 导入先严格base64/size/version/time/nonce/schema/checksum检查，拒绝未知字段膨胀、过期、该目标设备已消费nonce和恶意JSON；错误文案区分corrupt、expired、replayed-on-this-device与credential-invalid。
- R7: 成功导入通过Pixiv endpoint验证credential与authoritative profile，再原子写CredentialStore/AccountStore；失败不留下半账号。
- R8: payload、credential、nonce不写日志、SharedPreferences、task artifacts或crash report；粘贴读取只在用户明确操作时发生。

## Acceptance Criteria

- [ ] 复制/粘贴可在两台受支持Android设备完成一次真实账号迁移，成功后目标端重启可恢复。
- [x] 过期、篡改、截断、超大、未知version、重复nonce和无效credential均被拒绝且无半写入（单元及服务边界已覆盖）。
- [x] clipboard具有sensitive标记并按策略自动清理，不误清用户后来复制的其他内容（API 35 channel/UI 路径与 conditional-clear 测试已覆盖）。
- [x] 源码/配置不含beta56硬编码key或新的共享静态secret；UI不宣称无法提供的机密性。
- [x] 篡改 checksum 可被攻击者重算的威胁边界在 research/UI 中明确；测试只声称 accidental corruption 与本机 replay detection，不把它写成 authenticity。
- [x] 导入后account metadata来自服务端验证而非clipboard盲信，账号切换正确（生产 verifier 与 atomic store 单元边界已覆盖）。
- [ ] analyze、全量test、debug build及API36跨设备/过期/clear真机验证通过。

### Evidence boundary

API 35 MuMu 已完成真实登录账号的 Settings 导出、Login help 风险说明和
剪贴板迁移入口验证，并在同一 emulator 上做了真实 credential 的识别导入
尝试及重启恢复检查。该证据明确写作
`MuMu emulator-tested, not physical-device-tested`；不能替代两台独立设备或
API 36。API 36 镜像和第二台独立 Android 设备当前不可用，因此对应验收项
保持未勾选。全量测试另有既有 `test/icon_font_test.dart` 的 VM
`MissingPluginException`，详见 implementation evidence；这不是账号登录链路
blocker。

## Out of Scope

- 二维码/公钥配对。
- 云端账号同步。
- 保证抵御已获clipboard读取权限的恶意应用而又不增加用户秘密。

## Risks and Deferred Items

- Replica UX与强端到端机密性存在物理约束；实现前安全评审必须记录残余风险，若用户要求更强威胁模型需重新审批UX。

## Source Anchors

- beta56 lib/app/encrypt/encrypt.dart、pages/login/controller.dart、pages/settings/settings.dart
- CredentialStore/AccountStore与Android clipboard APIs

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
