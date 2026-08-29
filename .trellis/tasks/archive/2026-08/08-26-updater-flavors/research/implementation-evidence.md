# 08-26-updater-flavors implementation evidence

验证日期：2026-08-29（Asia/Shanghai）

## Implemented

- 增加 `github` 与 `fdroid` Android product flavor，并把
  `REQUEST_INSTALL_PACKAGES` 只放在 GitHub overlay；F-Droid 的 native
  channel 只返回 store-managed capability，不构造 updater HTTP client、验签
  或安装实现。
- 增加 bounded signed-manifest check：固定仓库、schema、tag/channel、严格
  semver、exact asset size、SHA-256、package name、signing certificate、严格
  HTTPS release host 与手动 bounded redirect policy。没有内置/注入公钥时保持
  fail closed。
- About 更新入口保留检查、无更新、prerelease、失败、确认、下载进度和取消
  状态；F-Droid 显示商店管理更新说明，不显示空操作按钮；安装前要求用户
  confirmation，并通过受控 `files/updates/` + `FileProvider` content URI。
- APK 通过 `DownloadManager` 的 `updaterApk` target 流式写入，校验 exact size、
  SHA-256、package 和已安装 signer；下载状态绑定 download id 与完整签名
  asset identity，进程恢复不跨 URL/hash/owner 重挂任务。
- 取消操作从 About 传递到 updater coordinator 的精确 download task；取消或
  失败会清理受控临时 APK，错误保持可观察。

## Compiled

- `/opt/flutter-3.47.0/bin/flutter analyze`：`No issues found!`。
- `/opt/flutter-3.47.0/bin/flutter build apk --debug --flavor github`：通过。
- `/opt/flutter-3.47.0/bin/flutter build apk --debug --flavor fdroid`：通过。
- Gradle 四种变体分别通过：
  `:app:assembleGithubDebug :app:assembleFdroidDebug
  :app:assembleGithubRelease :app:assembleFdroidRelease`；release 构建使用
  `--max-workers=1`，规避本机并发 R8/JDK daemon 崩溃。
- 当前 APK SHA-256：
  - GitHub debug：`3b194e60f3df4c416afb0c00d6b54c55b86f7d533ed16b1654b0958b80e60a3d`
  - F-Droid debug：`ca44a123384c6d29264556cafae6118951027d2bb7d9235b844b8aff63cb0107`
  - GitHub release：`c7fe90e7f0060dec64e602f3cadc103108fdcd52899667865273c2890cdd1211`
  - F-Droid release：`816b6233dd09316575ca31dfe98589264c48221a9f20555eef6aa7a05c2c6770`
- merged manifest / APK permission audit：GitHub debug/release 含
  `android.permission.REQUEST_INSTALL_PACKAGES`；F-Droid debug/release 均不含；
  两者都保留正常 `INTERNET` 权限。

## Unit-tested

- `/opt/flutter-3.47.0/bin/flutter test test/updater_about_test.dart
  test/updater_flavor_contract_test.dart test/updater_manifest_test.dart
  test/updater_download_test.dart test/settings_test.dart`：全部通过。
- updater 专项断言覆盖：valid/invalid/malformed/missing signature、unknown
  schema/channel、manifest oversize、429、offline、stable prerelease、asset
  signer mismatch、strict redirect、exact size/hash cleanup、package/signer
  verification delegation、explicit confirmation、下载取消和恢复身份。
- `git diff --check`：通过。

## Device-tested

MuMu emulator-tested, not physical-device-tested。

- ADB 候选端点为 `127.0.0.1:7555` 与 `127.0.0.1:16384`；经状态核验，本次统一
  使用 `127.0.0.1:16384`，state=`device`。
- 设备为 MuMu emulator API 35（`ro.build.version.sdk=35`）；global
  `http_proxy=null`。`dumpsys connectivity` 显示 Wi-Fi、NAT 地址
  `10.0.2.15`、`VALIDATED`、`NOT_VPN`。这只是宿主机 NAT 的模拟器样本，不能
  推广为运营商真机覆盖。
- GitHub debug APK 已安装并启动；登录状态、推荐 feed/图片和 About 页面可见，
  无 `FATAL EXCEPTION` 或应用崩溃。About 的 update check 实际执行后显示
  `更新检查或安装失败，请稍后重试`；本地构建未注入 release public key，仓库
  也没有可供验签安装的 production manifest/APK，因此不把它记为真实发布成功。
- F-Droid debug APK 已安装并启动；`dumpsys package` 未列出
  `REQUEST_INSTALL_PACKAGES`，About 显示 `此构建由 F-Droid 管理更新。`，无
  updater check button。

## Real API / release boundary

- 已在无系统代理、无外部代理 App 的 MuMu 环境执行 GitHub flavor 的真实检查
  路径；应用只使用代码中配置的 exact HTTPS release endpoint，未改 Host、SNI、
  证书校验或请求 route。当前结果受本地缺少 public key / production signed
  manifest 与 APK 限制，不能宣称真实 release/install 链路通过。
- 未创建 GitHub Release、未上传 manifest/APK、未持久化私钥或账号凭据，也未把
  当前已登录账号身份写入证据。

## Blockers and limits

- 当前只有 MuMu API 35，没有 API 36 镜像；API 36 acceptance 仍是 blocker，
  API 35 证据不能冒充 API 36。
- 生产 public key、与其匹配的 release signer、signed manifest 和发布资产尚未
  配置；因此 GitHub updater 的成功验签、未知来源授权后安装、安装拒绝/取消
  的完整系统分支尚未声称通过。
- 全量 `flutter test` 此前执行到 `+375` 后在既有
  `test/download_manager_test.dart` 的真实 loopback cancellation case 超过
  30 秒失败；本叶子未削弱该断言或修改无关下载测试。
- 本次只做 MuMu 模拟器验证；没有物理设备、API 36、三大运营商或远程发布验证。
