# 补齐 Android API 36 平台行为

## Goal

在 API 36 上建立 Replica 所需的统一 Android 平台契约，使导航、外部 intent、文件与媒体操作安全且可真实验证。

## Confirmed Facts

- 当前 Android scaffold 只包含 launcher intent 和 Flutter embedding，未配置 deep links、SEND、FileProvider 或媒体桥接。
- beta56 Manifest 提供原版 URI/intent/Widget 行为参考，但包含旧存储权限、全局 cleartext 和安装权限，不能直接复制。
- 目标 applicationId/namespace 为 io.github.lopution.pixivfunc，FileProvider authority 必须使用 applicationId.fileProvider。

## Dependencies

- 08-26-flutter-android-scaffold 完成。
- OAuth callback 与 app navigation contract 在 08-26-oauth-pkce-webview-login 中稳定。

## Requirements

- R1: 实现 Android 16 edge-to-edge，并核对状态栏、导航栏、IME 与现有 Replica scaffold 的 inset 行为。
- R2: 使用 Flutter PopScope/当前预测返回 API 统一页面回退；根页面两次 back 间隔约 1 秒退出，首次提示且计时不跨生命周期泄漏。
- R3: 声明并解析 http/https pixiv.net|www.pixiv.net、pixiv://users|illusts|account、pixivfunc://users|illusts；外部参数严格校验后映射 typed route。
- R4: 接收 ACTION_SEND image/* 和 content URI grant，用于后续反向搜图；拒绝非图片、不可读、超大或无权限输入并给出明确错误。
- R5: 配置非导出的 FileProvider，authority=applicationId.fileProvider，paths 最小化且不暴露任意内部目录。
- R6: 提供 MediaStore 流式写入/提交/失败清理的平台接口，不申请旧式广域存储权限。
- R7: 准备 AndroidX WebKit capability adapter，供 OAuth/Compatibility task 查询 ProxyController 等能力；能力不足时安全失败。
- R8: 基础 Manifest 不启用 usesCleartextTraffic=true、REQUEST_INSTALL_PACKAGES 或全局 TLS 放宽。

## Acceptance Criteria

- [ ] API 36 设备上 edge-to-edge 无关键内容被 system bars/IME 遮挡，predictive back 动画和页面结果正确。
- [ ] 根页第一次 back 提示、约 1 秒内第二次退出；超时、切后台、push route 后不会误退出。
- [ ] 所有允许 deep link 均到达正确 typed route，恶意/缺参/未知 URI 不崩溃也不误导航。
- [ ] SEND image/* 对 content URI、权限丢失、错误 MIME 和大文件路径有测试及设备验证。
- [ ] FileProvider merged manifest authority 正确且不能读取未授权路径；MediaStore pending item 失败时被清理。
- [ ] merged manifest 不含禁止的全局 cleartext、旧存储或安装权限；debug APK 构建通过。

## Out of Scope

- 反向搜图 API/UI。
- 完整下载队列。
- Compatibility CONNECT 服务。
- Updater flavor 的安装权限。
- Widgets。

## Risks and Deferred Items

- 预测返回与 Flutter engine/API 36 组合可能有版本差异；必须以真实设备行为为验收，不能只看 targetSdk。

## Source Anchors

- beta56 android/app/src/main/AndroidManifest.xml、lib/app/url_scheme/url_scheme.dart、Android platform plugins
- 当前 android/app/src/main/AndroidManifest.xml、MainActivity.kt、Replica navigation shell

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
