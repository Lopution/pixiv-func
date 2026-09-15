# Windows 桌面客户端支持

## Goal

让 Pixiv-func 能以 Windows 桌面客户端构建、登录并运行主要功能，使其同时作为 UI 重构任务（09-11-ui-redesign）的桌面验收载体和一个真实可用的客户端形态。优先 Windows，不做 Linux/macOS 桌面。

## Background and confirmed facts

- 项目现状只有 `android/` runner；无 `windows/`、`linux/`、`macos/`、`web/` 目录。
- 本地 Rust 插件 `plugins/rhttp`（cargokit + flutter_rust_bridge）已在 pubspec 声明 `windows: ffiPlugin: true`；根 `rust-toolchain.toml` 只 pin 了 4 个 Android target，需追加 `x86_64-pc-windows-msvc` 并在 Windows 侧准备 MSVC/Rust 工具链。
- `Platform.isAndroid/isIOS` 调用点只有 3 处文件；`sqflite` 已按 `Platform.isAndroid||isIOS` 走 `databaseFactoryFfi`（`lib/core/history/history_database.dart:47`），桌面 DB 路径已就绪。
- `flutter_secure_storage` 支持 Windows（DPAPI）；`path_provider`/`shared_preferences`/`go_router`/`cached_network_image` 等均支持 Windows。
- `webview_flutter` **不支持 Windows**：`login_webview_page` 用它拦截 `session.authorizeUrl` 完成 OAuth；`reverse_image_search_page` 用它承载 SauceNAO 结果页。两处是硬平台依赖。
- 账号迁移链路已存在，是 OAuth 之外的补充登录路径：`AccountTransferService.exportCurrentToClipboard()` 导出 payload，`PixivTransferCredentialVerifier` 先验 access token、过期则走 `OAuthService` refresh（`lib/core/auth/account_transfer_service.dart`）。
- Android 专属面需要守卫/降级：APK 自更新（`lib/core/updater/`）、`external_intent_bridge`（intent 路由）、下载目录选择（Android SAF 语义）、登录 WebView、SauceNAO WebView。
- 历史会话记录：本机 Windows 侧外网流量走 Clash 代理（`127.0.0.1:7897` 或网关），网络验收需在代理下验证；参考实现 PixEz 已有 `build_windows.yml` MSIX  nightly CI。
- 本机环境是 WSL2（Ubuntu-24.04），Flutter 3.47.2 在 `/opt/flutter-3.47.2`；Windows 构建必须在 Windows 侧执行（WSL 不能交叉编译 Windows runner）。

## Requirements

- R1 `flutter create --platforms=windows` 落地 runner，`flutter build windows` 在 Windows 侧产出可运行 exe；rhttp 增加 `x86_64-pc-windows-msvc` target 并通过 cargokit 链入。
- R2 Windows 上提供完整 OAuth 登录（应用内 WebView2 拦截 `pixiv://account?code=` 回跳，PKCE/verify 链路复用现有 `OAuthService`），账号迁移导入作为既有补充路径保留。
- R3 Android 专属能力显式守卫或降级（自更新、intent、SAF 目录、WebView 页面），不静默坏掉。
- R4 主要浏览链路（推荐→详情→作者→我的、搜索、历史、下载任务）在 Windows 桌面可用，服务 UI 重构任务的代表宽度/键盘/减少动效验收。
- R5 分发与 CI 形态明确（zip portable / MSIX / GitHub Actions windows job）。

## Acceptance Criteria

- [ ] Windows 侧 `flutter build windows --release` 成功产出可运行产物。
- [ ] Windows 客户端能完成登录并拉取真实 Pixiv 数据（走本机代理）。
- [ ] 浏览样板链路在桌面宽度下无溢出/不可达操作；移动端行为无回归。
- [ ] 不支持的功能有可见降级而不是崩溃或静默失败。

## Confirmed decisions

- 登录走完整 OAuth（用户明确选择，非 MVP 旁路）。技术形态：应用内 WebView（`flutter_inappwebview`，Windows 实现经 CDP `Fetch.requestPaused` 支持 `shouldOverrideUrlLoading`），拦截 `pixiv://account` 后取消导航并走既有 `exchangeCode`。
- 分发：先做 zip portable + GitHub Actions `windows-latest` 产物；MSIX/签名延后。
- 不做 Linux/macOS 桌面（用户明确）。

## Notes

- 本任务由 UI 重构的验收需求触发，但 Windows 客户端本身是独立交付物，故为独立任务而非子任务。
