# 技术设计：Windows 桌面客户端支持

只描述实现边界与契约；不改 Pixiv API、OAuth 语义、下载恢复、阅读位置等业务行为。登录安全边界保持"PKCE 会话 + 精确 `pixiv://account` 回调匹配"不变。

## 1. 平台选择机制（先决）

业务层不散落 `Platform.isWindows`：新增单一探测点（如 `lib/core/platform/platform_caps.dart` 或等价位置），提供 `isDesktop`/`supportsX` 语义化判定；所有 provider/构造器注入点接受显式 override，使 `flutter test`（Linux 上 `Platform.isWindows==false`）也能覆盖 Windows 分支。沿用现有风格：每个平台能力已是 `abstract interface class` + `MethodChannel*` 实现，Windows 侧新增对应实现类，在选择处按 caps 挑选。

## 2. 构建链

- `flutter create --platforms=windows .` 生成 `windows/` runner（在 WSL 里生成模板即可；**构建只能在 Windows 侧**：VS 2022 "Desktop development with C++"、Windows SDK、rustup target `x86_64-pc-windows-msvc`、可选 nuget）。
- `rust-toolchain.toml` 追加 `x86_64-pc-windows-msvc`；cargokit 负责链入 `rhttp.dll`。`plugins/rhttp` 已声明 `windows: ffiPlugin: true`，无需改插件清单。
- `main()` 的 `RhttpGate.ready = rhttp.Rhttp.init()` 语义不变；rhttp 加载失败经既有 gate 上报，不静默。
- sqflite：桌面已走 `databaseFactoryFfi`（`history_database.dart`），无需改。
- `webview_flutter` 无 Windows 实现——仅运行时缺插件，编译不受影响；Windows 路径不实例化它。

## 3. 登录（OAuth over WebView2）

- 抽薄接口 `OAuthWebView`（load authorizeUrl / redirect 回调 / progress / error / dispose），Android/iOS 沿用 `webview_flutter` 现有 `login_webview_page.dart` 实现，Windows 用 `flutter_inappwebview` 的 `InAppWebView`（`useShouldOverrideUrlLoading: true`，拦截到 `pixiv://account` → `NavigationActionPolicy.CANCEL` → `OAuthService.exchangeCode`）。
- 选择点集中在路由/页面构建处：Windows 用 inappwebview 版页面，其他平台保持原页；`OAuthService.beginSession/exchangeCode/discardSession` 原样复用。
- 风险预案：若 CDP 拦截对服务端 302 到 `pixiv://` 的回跳漏触发，退化到 `onUpdateVisitedHistory`/url 流检测 + `stop()` + 照常 exchange（code 已在 URL 中，`pixiv://` 在 WebView2 内无法真正加载，时序宽容）。
- 注册（`create: true` → accounts.pixiv.net/signup）同页通用，Windows 版同样承载。
- `webview_windows`/`webview_flutter_windows`（tomars fork，自有 API）缺导航取消能力，不采用。

## 4. 平台能力矩阵（Windows 实现）

| 能力 | 接口 | Windows 策略 |
| --- | --- | --- |
| 自更新 | `UpdatePlatform` | `UnsupportedUpdatePlatform`：capability 报 disabled，设置页隐藏更新入口 |
| 外部 intent 进/出 | `AndroidIntentSource` / `_OutboundUrlOpener` | inbound: no-op（现有 MissingPluginException 已容错，仍给显式空实现）；outbound: `url_launcher`（新增 endorsed 依赖，Windows 支持打开外部链接） |
| 下载落盘 | `SafDocumentSinkFactory`/`SafTreePicker`、`MediaStoreSink` | 桌面文件 sink：`dart:io` 流式写入选定目录；目录选择用 `file_selector`（endorsed，`getDirectoryPath`），默认 `Downloads/PixivFunc` |
| 账号迁移剪贴板 | `TransferClipboard` | Windows 用 Flutter `Clipboard`（payload 是文本）；超大 payload 边界按现有 validator 处理 |
| 以图搜图输入/外链 | `ReverseImageInputPlatform` / external launcher | `file_selector` 选图 + `url_launcher` 打开外链 |
| 资料页 web session | `WebProfileSession` | 现有 MissingPluginException→null 降级已可用；资料编辑入口在 Windows 显示不可用态 |
| 分享图片 | `SharedImage`(ACTION_SEND) | Windows 无对应语义：分享入口隐藏或降级为"保存到下载目录"（按现有调用点最小改动） |
| SauceNAO 结果页 | webview_flutter | 换 `InAppWebView` 承载（与登录同插件），复用 `SauceNaoNavigationPolicy` |
| 返回手势/退出提示 | `RootBackCoordinator` + shell `PopScope` | Windows 无系统返回；守卫为 no-op，退出语义由窗口管理 |
| 后台 widget worker | `widgetBackgroundMain` | 仅 Android 注册；Windows 不注册 |

## 5. 桌面形态与壳层

- runner 定制：窗口标题 `Pixiv Func`、图标（复用现有应用图标资产转 .ico）、合理初始/最小尺寸（如 400×600 min，默认 390×844 附近以贴近手机验收宽度）。
- 键盘/鼠标：桌面验收依赖 ui-redesign 的响应式与焦点工作；本任务不重复实现，只保证窗口可缩放、滚动/右键/文本选择行为不被破坏。
- 字体：Windows 无系统级思源/CJK 兜底差异，视觉复核项记录进验收清单，不预修。

## 6. 网络与代理

- rhttp + DoH + SNI 策略为纯 Rust/Dart，平台无关，不改。
- 验收环境走 Clash（`127.0.0.1:7897`）；确认 rhttp 是否需显式代理设置或跟随系统——若 rhttp 不读系统代理，网络层补一个"使用系统代理"开关属于既有设置范围，不在本任务新增（先实测再定）。

## 7. 测试策略

- 平台选择可注入 → Windows 分支逻辑可在 Linux `flutter test` 下覆盖（能力矩阵、sink、剪贴板、登录页的 Windows 变体 widget 测试可跑，inappwebview 本体不渲染但控制器 mock 层可测拦截逻辑——拦截判定抽成纯函数）。
- 既有套件在 Linux 全量回归：Android 语义路径必须零变化。
- Windows 侧人工验收清单：登录（含 signup 入口）、代理下真实数据、样板链路、下载落盘、迁移导入兜底、窗口缩放/横宽比、WebView2 runtime 缺失提示。

## 8. CI 与分发

- `.github/workflows/` 增加 windows job：`windows-latest` + Flutter 3.47.x + rust msvc target + `flutter build windows --release` → zip 上传 artifact。
- MSIX/签名/商店延后；zip 产物先满足验收与真实用户侧载。

## 9. 关键风险与停止条件

- `flutter_inappwebview_windows` 0.6.0 的 CDP 拦截对 Pixiv 回跳不可靠 → 用 §3 退化方案兜底；仍不行则停止登录移植，先交付其余平台支持并如实记录。
- cargokit/MSVC 在 Windows 侧构建失败 → 单独隔离构建问题，不让 Dart 改动带病推进。
- WebView2 Runtime 缺失 → 登录页给出可操作的安装指引（`getWebViewVersion()` 探测）。
- 桌面端被 Pixiv 风控/WAF 差异对待 → 记录现象，不在本任务内对抗。

## 10. 与 09-11-ui-redesign 的关系

本任务交付的 Windows 客户端是 ui-redesign R6"真实设备验证"的桌面载体；UI 重构的响应式/可访问性实现仍在那个任务里，本任务只保证客户端可用。
