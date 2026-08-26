# 建立安全多账号与启动状态

## Goal

建立可恢复、可测试且不会泄露秘密的账号域，使冷启动和后续认证请求由真实账号状态驱动。

## Confirmed Facts

- 当前 lib/app/app.dart 固定向 StartupGate 传入 hasAccount: false，登录成功后无法进入真实账号态。
- beta56 的 AccountService 把包含 cookie/token 的账号 JSON 存入 SharedPreferences；该模式只用于理解多账号 UX，不能沿用。
- 父 PRD 明确要求普通 metadata 与 credential 分离，并用 Android 安全存储保存秘密。

## Dependencies

- 08-26-flutter-android-scaffold 完成。
- OAuth 和网络任务将依赖本任务稳定的账号/凭据接口。

## Requirements

- R1: 定义不可变 Account metadata、认证状态和稳定 account ID；token、cookie 等秘密不得进入该对象的普通序列化或日志。
- R2: 提供 CredentialStore 接口及基于 flutter_secure_storage/Android Keystore 的实现，包含版本化 key namespace、读写、更新和删除。
- R3: 提供 AccountStore，支持 hydration、账号列表、当前账号、添加、更新、切换、退出、删除和 reauthRequired。
- R4: metadata 与 credential 的写入顺序必须可恢复；遇到缺失、损坏或 Keystore 失效时返回明确状态，不静默假定无账号。
- R5: StartupGate 等待 Settings 与 AccountStore hydration，并严格选择 Welcome、Login 或 Home；错误状态可观察且可重试。
- R6: 测试使用明确注入的内存 CredentialStore，不得在生产路径加入隐藏 mock，也不得把秘密写入测试快照。

## Acceptance Criteria

- [ ] guide false、guide true/no account、account exists 三个冷启动分支均由真实 store 状态通过 Widget 测试。
- [ ] 两个以上账号可以添加、切换、退出和删除；当前账号持久化后可在重启 hydration 中恢复。
- [ ] access token、refresh token、cookie 不出现在 SharedPreferences、日志、task artifacts 或普通 Account JSON。
- [ ] 安全存储损坏/读取失败进入明确错误或 re-auth 流程，不显示空白页且不伪装成成功退出。
- [ ] flutter analyze、全量 flutter test 和 debug APK 构建通过。

## Out of Scope

- OAuth authorize/token exchange。
- Token 自动刷新。
- 剪贴板账号迁移。
- 服务器账号资料编辑。

## Risks and Deferred Items

- Android Keystore 在锁屏变更、备份恢复或 app 重装后可能使密文不可用；必须设计为可重新认证而非崩溃。

## Source Anchors

- beta56 lib/models/account.dart、lib/app/services/account_service.dart
- 当前 lib/app/app.dart、lib/features/onboarding/startup_gate.dart、lib/core/settings/*

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
