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
- R3: GitHub flavor只把固定公钥验证通过的signed release manifest作为版本/asset信任根；manifest由外部CI签名并包含schema、repo、tag/channel、asset URL、exact size与SHA-256。GitHub API只可用于非安全展示或受签名字段约束的下载位置，不能单独授权安装。
- R4: 检查/下载/取消/进度/错误/无更新/发现更新和用户确认保持beta56 About体验，不自动静默安装。
- R5: APK下载复用DownloadManager流式写入owned受控位置/FileProvider，不读取完整bytes；必须校验manifest exact size、SHA-256、package name和下载APK signing certificate与已安装应用匹配，任一缺失/不匹配均fail closed。
- R6: 安装Intent使用content URI、临时grant和当前Android未知来源权限流程；失败/拒绝明确并清理临时APK。
- R7: F-Droid构建完全不发GitHub update网络请求且UI不显示空操作；About可说明由商店更新。
- R8: manifest/signature/release URL严格解析并限长，redirect只允许manifest声明且通过host policy的目标；无内置公钥、签名无效、未知schema/asset/channel均明确拒绝，日志不含私有token；本任务不创建release。
- R9: check和apply分别single-flight；下载ID/tag/hash状态可恢复，进程重启只附着到完全匹配的既有DownloadManager任务，否则清理owned临时APK后重新请求用户确认。

## Acceptance Criteria

- [ ] GitHub/F-Droid flavor均可构建，merged manifest权限差异符合要求。
- [ ] F-Droid无REQUEST_INSTALL_PACKAGES、无self-updater代码路径/网络请求/空按钮。
- [ ] GitHub检查版本、无更新、prerelease、malformed/429/offline及取消状态通过测试。
- [ ] valid/invalid/missing signature、unknown schema/channel、manifest oversize和asset mismatch测试通过；未验签字段从不进入UpdateAvailable。
- [ ] APK流式下载、exact size/hash/package/installed-signer检查、FileProvider/install拒绝/cleanup通过API36验证。
- [ ] About UI反馈与beta56节奏一致且不会静默安装。
- [ ] analyze、全量test、两个flavor debug/release build及GitHub flavor真机安装流程验证通过。

## Out of Scope

- 实际创建GitHub Release或上传APK。
- F-Droid发布。
- 自动静默安装或绕过用户授权。

## Risks and Deferred Items

- Android安装权限、manifest signing key和APK signer材料可能阻止自更新；必须作为明确配置/发布blocker，不放宽signature/hash/package校验。私钥不进入仓库或App。

## Source Anchors

- beta56 lib/app/updater/updater.dart、pages/about、Android Manifest/FileProvider
- DownloadManager、AndroidPlatform和最终release task contracts

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
