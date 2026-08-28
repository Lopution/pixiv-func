# Android 平台边界与 WebView 生命周期 — Implementation Plan

## Start Gate

- 依赖 `08-26-restricted-compat-network`、OAuth/account contract，并在本叶子 planning review 后启动。
- 不改写 `08-26-android-platform-parity` archive；widgets/download 只通过公开 adapter 接入。

## Steps

1. 复核 `android_platform_interfaces.dart`、`android_platform.dart`、`intent_router.dart`、`login_webview_page.dart` 和 intent/login/navigation/platform tests，记录 channel、manifest、lifecycle 和 URI grant 现状。
2. 添加 capability/route、invalid-intent、double-dispose、background/rotation 和 pending cleanup fixtures，先固定当前失败边界。
3. 实现 typed WebView session 和 exact-host loopback gate；将 route 创建/清理与 account/network revision 绑定。
4. 强化 OAuth callback、deep link、SEND、FileProvider、MediaStore 和 permission/error mapping；保持 exported 与 TLS 安全控制。
5. 补充 API 36 emulator/device、WebView 版本和无系统 proxy/VPN 的出口证据；更新 integration Android evidence。

## Validation

```bash
/opt/flutter-3.47.0/bin/flutter test test/intent_router_test.dart test/login_navigation_test.dart
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter build apk --debug
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-android-platform-boundary-hardening
git diff --check
```

若平台插件或 manifest 变更，增加 merged manifest、API 36 真机、OAuth callback、FileProvider/MediaStore 和 WebView route 记录；未实测的能力保持 blocker。

## Completion Gate

- [x] WebView capability/session/loopback 仅在 exact Pixiv host 和 active lifecycle 内成立，结束后清理。
- [x] 非法 intent/callback、权限拒绝、rotation/background、FileProvider/MediaStore pending 均安全失败或 exactly-once 清理。
- [x] 无 TLS/SNI/Host/IP/反代绕过，归档前 integration evidence 已回填；API 36 设备证据保留 blocker。
