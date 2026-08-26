# 实现 Pixiv OAuth PKCE WebView 登录 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 实时核验 authorize/token endpoint、客户端身份与 WebView 支持矩阵，记录主源和时间。
2. 实现 PKCE 生成、session 生命周期和 redirect parser 的纯 Dart 核心。
3. 引入/配置 WebView，连接现有 Login shell 的注册/登录操作和状态 UI。
4. 实现严格 TLS token exchange 与 AccountStore 成功/失败事务。
5. 增加 PKCE、callback、安全失败、取消、重复消费和 Widget 导航测试。
6. 在 Android 设备上完成真实登录、取消、证书失败和重启恢复验证。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-oauth-pkce-webview-login
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- RFC PKCE 向量、随机 verifier、TTL 与 one-use。
- redirect URI 白名单和恶意 query corpus。
- token exchange success/error/cancel/timeout，不发生半写入。
- Login UI help 与按钮可见行为不回归。
- 真实 WebView 登录及日志/存储泄漏审计。

## Risky Files and Rollback Points

- pubspec.yaml、lib/core/auth/oauth_service.dart、lib/features/login/、WebView Android 配置、账号写入边界

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

