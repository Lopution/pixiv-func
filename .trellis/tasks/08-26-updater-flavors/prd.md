# 实现 GitHub 与 F-Droid Updater flavors

## Goal

复刻About中的检查更新体验，同时通过flavor彻底隔离GitHub自更新与F-Droid无安装权限构建。

## Confirmed Facts

- beta56 Updater下载APK并调用安装，基础Manifest全局声明REQUEST_INSTALL_PACKAGES。
- 父PRD要求GitHub flavor允许check/download/install，F-Droid禁用self updater且不请求安装权限。
- 远程发布和签名材料不属于本任务自动授权。

## Dependencies

- 08-26-download-manager-mediastore与android-platform-parity完成。
- release/version/签名策略在最终集成任务审查。

## Requirements

- R1: 建立明确GitHub与F-Droid product flavors/compile-time capability，不能仅用runtime隐藏按钮。
- R2: 基础与F-Droid merged manifest不含REQUEST_INSTALL_PACKAGES或self-updater组件；仅GitHub flavor声明最小安装权限。
- R3: GitHub flavor通过HTTPS GitHub Releases/API检查当前repo release，使用语义/versionCode比较并处理prerelease策略。
- R4: 检查/下载/取消/进度/错误/无更新/发现更新和用户确认保持beta56 About体验，不自动静默安装。
- R5: APK下载复用DownloadManager流式写入受控位置/FileProvider，不读取完整bytes；校验content length、SHA-256（若发布提供）和包名/签名兼容。
- R6: 安装Intent使用content URI、临时grant和当前Android未知来源权限流程；失败/拒绝明确并清理临时APK。
- R7: F-Droid构建完全不发GitHub update网络请求且UI不显示空操作；About可说明由商店更新。
- R8: release API/URL/JSON严格解析、redirect host受限，日志不含私有token；本任务不创建release。

## Acceptance Criteria

- [ ] GitHub/F-Droid flavor均可构建，merged manifest权限差异符合要求。
- [ ] F-Droid无REQUEST_INSTALL_PACKAGES、无self-updater代码路径/网络请求/空按钮。
- [ ] GitHub检查版本、无更新、prerelease、malformed/429/offline及取消状态通过测试。
- [ ] APK流式下载、hash/package/signature检查、FileProvider/install拒绝/cleanup通过API36验证。
- [ ] About UI反馈与beta56节奏一致且不会静默安装。
- [ ] analyze、全量test、两个flavor debug/release build及GitHub flavor真机安装流程验证通过。

## Out of Scope

- 实际创建GitHub Release或上传APK。
- F-Droid发布。
- 自动静默安装或绕过用户授权。

## Risks and Deferred Items

- Android安装权限和签名匹配会阻止自更新；必须作为明确错误，不放宽package/signature校验。

## Source Anchors

- beta56 lib/app/updater/updater.dart、pages/about、Android Manifest/FileProvider
- DownloadManager、AndroidPlatform和最终release task contracts

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
