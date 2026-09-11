# 实现计划：Windows 桌面客户端支持

按可独立验证的阶段组织；每阶段小步提交、可单独回滚。**特殊约束：Windows runner 的构建/运行验收必须在 Windows 侧执行**，WSL 内只能完成 Dart 代码、模板生成与 `flutter test`/`analyze`。

## 阶段 0：基线与工具链清单

- [ ] 记录 `flutter analyze --no-pub`、全量 `flutter test` 基线；对照 `baseline-2026-09-07` tag 无需重测。
- [ ] 写清 Windows 侧前置条件清单：VS 2022 C++ 工作负载、Windows SDK、rustup `x86_64-pc-windows-msvc`、WebView2 Runtime、Clash 代理环境。
- [ ] 提交：`docs(windows): record platform baseline and toolchain prerequisites`。

## 阶段 1：runner + 平台能力层（Dart 侧全部可在 WSL 完成）

- [ ] `flutter create --platforms=windows .` 生成 runner；定制标题/图标/最小尺寸；`rust-toolchain.toml` 加 msvc target；pubspec 加 `flutter_inappwebview`、`url_launcher`、`file_selector`（锁定发布 ≥7 天的版本）。
- [ ] 新增平台能力探测点（可注入，非散落 `Platform.isWindows`）；为 updater/intent/mediastore/saf/clipboard/reverse-image/webprofile/shared 写出 Windows 实现或显式 Unsupported 降级；RootBackCoordinator/后台 widget worker 守卫。
- [ ] 提交边界：runner+构建链一个提交；能力层按接口分组提交。每次提交跑 `dart format`、`flutter analyze --no-pub`、聚焦单测。
- [ ] 提交：`feat(windows): add runner and desktop capability layer`。

## 阶段 2：OAuth 登录（Windows）

- [ ] 抽 `OAuthWebView` 薄接口；移动端原页保持不动；Windows 实现 `InAppWebView` 版登录/注册页：拦截 `pixiv://account?code=` → CANCEL → `exchangeCode`；进度/错误/取消/dispose 语义对齐 `login_webview_page.dart`。
- [ ] 拦截判定抽纯函数并补单测；路由处按平台选实现。
- [ ] 提交：`feat(windows): oauth login via in-app webview`。

## 阶段 3：次级 WebView 与输入面

- [ ] SauceNAO 结果页 Windows 版（同 inappwebview），复用 `SauceNaoNavigationPolicy`；以图搜图输入走 `file_selector`、外链走 `url_launcher`。
- [ ] 账号迁移剪贴板 Windows 实现（Flutter `Clipboard`）+ 端到端导入验证点。
- [ ] 下载目录：`file_selector` 目录选择 + `dart:io` 文件 sink；默认 `Downloads/PixivFunc`。
- [ ] 提交：`feat(windows): secondary webview, file io and transfer paths`。

## 阶段 4：质量门与 Windows 侧验收

- [ ] `dart format --set-exit-if-changed .`、`flutter analyze --no-pub`、全量 `flutter test`（Android 语义零回归）。
- [ ] Windows 侧：`flutter build windows --release`、登录（含 signup）、代理下真实数据、样板链路、下载落盘、迁移导入兜底、窗口缩放、WebView2 缺失提示。
- [ ] 提交：`test(windows): desktop verification evidence and remaining limits`。

## 阶段 5（可选）：CI windows job

- [ ] windows-latest + Flutter 3.47.x + rust msvc + `flutter build windows --release` → zip artifact。
- [ ] 提交：`ci(windows): release zip artifact on windows runner`。

## 回滚策略

每阶段独立提交；平台选择走注入，Android 路径不改行为，回滚=revert 对应提交。若 inappwebview 拦截失效，停登录移植保其余交付并记录。
