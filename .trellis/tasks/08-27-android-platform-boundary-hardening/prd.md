# 补强 Android 平台边界与 WebView 生命周期

## Goal

补强已归档 Android platform parity 的 WebView 能力探测、深链/intent、FileProvider、MediaStore、权限和生命周期边界，并为大陆无外部代理目标下的 WebView route 提供可验证的平台契约。

## Scope and current facts

- 目标历史任务：`08-26-android-platform-parity`；相关消费者包括 OAuth/login、restricted network、downloads、widgets 和 navigation，但本叶子只拥有平台边界，不重复实现业务页面。
- 当前代码范围包括 `android_platform_interfaces.dart`、`android_platform.dart`、`intent_router.dart`、`login_webview_page.dart` 及 intent/login/navigation/platform tests。
- Q1 允许 App 内 Pixiv 域限定的 WebView loopback `CONNECT`，但只有能力探测通过且 exact-host、严格 TLS、生命周期隔离成立时才可使用；loopback 不能被描述为 SNI 隐藏或通用代理。

## Requirements

- R1：提供版本化 WebView capability/route session，明确支持/不支持、network revision、exact host allowlist、创建/清理 owner 和 fallback failure；能力不足时可见失败。
- R2：OAuth WebView 仅接受预期 `pixiv://account` callback 和已注册 state/PKCE session；深链、SEND image/*、intent extras、content URI 和 MIME 必须输入验证，拒绝任意 scheme/host/path。
- R3：loopback route 仅在 WebView session 生命周期内创建，按 owner/refcount 清理；页面销毁、账号切换、认证失败和 app background 不得留下可复用 listener。
- R4：FileProvider URI、MediaStore pending item、权限请求和 result callback 有明确所有权和 exactly-once cleanup；不扩大 exported component 或 grant 权限。
- R5：predictive back、双击退出、rotation/background/restore 与现有 beta56 行为兼容；插件/平台异常必须返回真实错误，不静默 no-op。
- R6：所有网络出口继续复用 shared `NetworkAccessPolicy`；不添加固定 IP、证书关闭、SNI/Host hack、第三方反代或全局代理。

## Acceptance Criteria

- [ ] API 36 模拟器/真机覆盖 capability success/failure、WebView session cleanup、OAuth callback 拒绝、非法 deep link/SEND、rotation/background 和 predictive back。
  - 当前明确 blocker：可验证的 MuMu 实例为 API 35；没有 API 36 MuMu 镜像，未将 API 35 结果冒充 API 36，也没有物理设备覆盖。
- [ ] FileProvider/MediaStore 在成功、取消、权限拒绝、进程重启和错误回调后没有 orphan grant、pending URI 或临时文件。
- [x] loopback 只对允许的 Pixiv host 生效，session 结束立即清理；非 Pixiv URL 和能力不足不走该路径。
- [x] Android platform、intent、login/navigation 聚焦测试通过；实际设备和 WebView 版本证据按 `Device-tested` 分层记录。
- [ ] 归档任务无 diff，且 widgets/download/integration 只消费本叶子公开的平台契约，不复制实现。

## Dependencies and Out of Scope

- 依赖：`08-26-restricted-compat-network`、OAuth/account contract 和现有 `08-26-android-home-widgets` 规划；不改变 widgets 功能范围。
- 不负责新建代理服务、绕过 WebView TLS/证书校验、重做 Android UI 或扩大 intent/exported 权限。
