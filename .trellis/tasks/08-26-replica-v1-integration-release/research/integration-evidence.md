# Replica v1 integration evidence

验证日期：2026-08-29（Asia/Shanghai）。本文件只汇总当前仓库和各叶子
证据，不把模拟器、fixture 或已有登录会话扩大成 API 36、物理设备、三大
运营商或完整生产发布证明。

## Implemented

- 最终集成补齐了根目录 `LICENSE`、`NOTICE`、`README.md` 的 AGPL-3.0-only
  许可、原作者归属、修改说明和“当前尚无可用发行版”边界。
- `research/license-audit.md` 固定了当前 lockfile、依赖 license 文件、
  beta56 资产来源、第三方审查来源和生产签名限制。
- 已接收 24 个原 Replica v1 实现叶子和 5 个 hardening 叶子的归档证据。
  Widgets、Updater 仍保持开放状态；Live 以 feasibility blocker 归档，
  没有留下 fixture/mock 播放器。
- 静态审计没有发现生产代码中的 `HttpOverrides`、cleartext 网络、固定 IP、
  TLS trust-all、SNI/Host 改写或第三方反代路径。源码中的 `unavailable`、
  `unsupported` 和可见 placeholder 均是显式失败/边界状态，不是伪造成功。

## Compiled

通过：

```text
/opt/flutter-3.47.0/bin/flutter analyze
No issues found! (ran in 2.3s)

/opt/flutter-3.47.0/bin/flutter build apk --debug
Built build/app/outputs/flutter-apk/app-debug.apk
SHA-256: 2ec7b14ce70b703fec85a6566398e71dc6cbea37b3cdb41a7c42f78c970f732c
```

Updater flavor 的四种 Gradle 变体也已通过：

```text
./gradlew :app:assembleGithubDebug :app:assembleFdroidDebug --no-daemon --max-workers=1
./gradlew :app:assembleGithubRelease :app:assembleFdroidRelease --no-daemon --max-workers=1
```

本次构建产物的 SHA-256 为：

| 变体 | SHA-256 |
|---|---|
| GitHub debug | `1a6c508a84404e84338a00e5a0d7ba1a3e3f0762e17e3ac711fd79151e3478d8` |
| F-Droid debug | `a8265780662e225a3d0cd4579e282e6885ee616adb00e54659fa2f4e3836a0f2` |
| GitHub release | `6051e50d4b5b21f3fa4ea7cb58100bd4461a1d3b17829a80c3c4a2827fc6b8b3` |
| F-Droid release | `c0e3e8b2e11acc74fe0b9a0a873de5d5f8a0fd730ff99f695cb6addebffc36e9` |

`/opt/flutter-3.47.0/bin/flutter build apk --release` 未生成
`build/app/outputs/flutter-apk/app-release.apk`，因为当前 Android 模块有
显式 `github`/`fdroid` flavor dimension；发布构建必须指定 flavor。两种
flavor 的 release Gradle 构建已通过，但当前都使用本地 debug signing
config，不能当作生产签名发布物。

Merged manifest/APK permission 审计通过：GitHub debug/release 含
`REQUEST_INSTALL_PACKAGES`，F-Droid debug/release 不含；两者均保留
`INTERNET`。F-Droid 不带 updater check button，GitHub/F-Droid flavor
隔离证据见 `08-26-updater-flavors/research/implementation-evidence.md`。

## Unit-tested

本次最终集成聚焦命令通过，共 125 tests：

```text
/opt/flutter-3.47.0/bin/flutter test --concurrency=1 --reporter compact \
  test/feed_generation_commit_test.dart test/mutation_ownership_test.dart \
  test/novel_markup_hardening_test.dart test/download_recovery_test.dart \
  test/android_platform_boundary_test.dart test/restricted_compat_network_test.dart \
  test/ugoira_test.dart test/ugoira_recovery_test.dart test/ugoira_viewer_test.dart \
  test/widget_feed_loader_test.dart test/widget_snapshot_store_test.dart \
  test/widget_snapshot_test.dart test/updater_about_test.dart \
  test/updater_flavor_contract_test.dart test/updater_manifest_test.dart \
  test/updater_download_test.dart
```

另外，`test/oauth_service_test.dart` 单文件 16 tests 通过；真实 loopback
取消用例单独运行通过。全量门禁没有通过：

- 默认 `/opt/flutter-3.47.0/bin/flutter test --reporter compact` 在 `+401`
  后卡在 `download_manager_test.dart` 的真实 socket cancellation case；
  等待约 2 分 50 秒后中止，Flutter runner 报
  `Bad state: Cannot close sink while adding stream` / `Cannot add event while
  adding stream`。
- `--concurrency=1` 的全量尝试在另一次 loopback/OAuth 交互中停在
  `oauth_service_test.dart` 的 token exchange case；OAuth 文件单独运行仍
  通过。该现象与测试文件之间的 WSL/Flutter runner loopback 资源竞争相符，
  但在它被稳定复现并修复前，不能宣称全量测试通过。

`git diff --check` 和当前最终 task 的 Trellis validation 均通过。

## Device-tested

MuMu emulator-tested, not physical-device-tested。

MuMu 管理器确认本次选用 `127.0.0.1:16384`；候选端点
`127.0.0.1:7555` 与 `127.0.0.1:16384` 都先做了 ADB 状态检查，没有盲选。
每次适用验证前执行了：

```text
adb devices -l
adb -s 127.0.0.1:16384 get-state                         -> device
adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk -> 35
adb -s 127.0.0.1:16384 shell settings get global http_proxy -> null
adb -s 127.0.0.1:16384 shell dumpsys connectivity
```

`dumpsys connectivity` 观察到 MuMu guest Wi-Fi、`VALIDATED`、`NOT_VPN`、
`10.0.2.15` 和 host-NAT gateway；没有将其写成运营商真机覆盖。包检查未见
外部代理/VPN App。

已在 API 35 MuMu 上实际安装/启动当前 debug flavor 并观察到：

- 已登录账号的真实 Recommended feed、详情/图片封面和账户 Settings
  页面可用；该真实账号链路不是 blocker。
- Widgets 的 headless refresh、last-good snapshot、deep link、resize、
  app 进程死亡后 worker 重建和无秘密 snapshot 已实际观察；没有执行系统
  重启后的完整 restore 矩阵。
- GitHub flavor 的 About update check 进入真实失败状态；本地没有生产
  public key/signed manifest/APK，因此未伪称更新成功。F-Droid flavor 实际
  不显示 updater check button，且 APK 没有 `REQUEST_INSTALL_PACKAGES`。
- 兼容网络、反向搜图 `ACTION_SEND`/picker、账号迁移、Profile read/editor
  等叶子已有 API 35 证据；每项的未执行写操作和外部 provider 失败均保留在
  owning evidence 中。

## Evidence matrix

| Gate | 当前结论 | 证据与剩余边界 |
|---|---|---|
| Feed generation / commit | Unit pass；API 35 smoke | `feed-generation-commit-hardening` 归档证据；API 36 和完整跨网络矩阵未验证 |
| Mutation ownership | Unit pass；API 35 boundary smoke | `mutation-ownership-hardening` 归档证据；真实 bookmark/follow/comment/profile 写操作未执行，不是缺少账号 |
| Novel markup | Unit pass；API 35 read path | typed token、未知标记、chunk/layout/cancel 证据完整；API 36 与真实长章压力仍未验证 |
| Media recovery / Ugoira | Unit pass；API 35 normal image/Ugoira GIF finalization | API 36 MediaStore、完整大媒体压力和所有重启组合未验证 |
| Android platform boundary | Unit pass；API 35 | WebView capability、intent、FileProvider、MediaStore 证据存在；API 36 仍缺失 |
| Restricted network | Direct-only / strict TLS unit and API 35 evidence | 系统 proxy/VPN off 的 MuMu 样本通过；ECH/WebView reverse-bypass 未获 capability，不能作大陆普遍可用声明 |
| OAuth/API/image/account | Existing real signed-in read chain pass | fresh OAuth、refresh、账号切换和非读写操作未在本次最终集成重复执行；已有账号不是 blocker |
| Reverse image | Explicit unavailable provider state | 没有 approved provider key/privacy contract，未伪造上传/结果卡 |
| Profile edit | Explicit unavailable mutation state | 没有 reviewed official update endpoint；read/editor UI 真实账号可见，Save 保持 disabled |
| Live | Feasibility blocker | 当日真实 App API list 三个 filter 均 `lives=0`，无 valid id/detail/HLS，故未加入 player/mock |
| Widgets | Implemented, focused/device pass | API 35 已验证；API 36、系统重启/OEM后台矩阵缺失，任务保持 `in_progress` |
| Updater/flavors | Implemented, focused/build/device pass | 生产 public key、signed manifest/APK、匹配 production signer 和可分发 release signing 尚未配置 |
| License/provenance | Repository audit pass with release follow-up | AGPL/NOTICE/README 和 lock/cache/asset provenance 已记录；发行包仍需随附完整第三方 license texts |

## Mainland access gate

当前唯一实际网络样本是 2026-08-29 的 MuMu API 35 host-NAT Wi-Fi；系统
proxy 为 `null`、网络为 `VALIDATED`/`NOT_VPN`。已观察到真实登录后的 API
Recommended、`pximg` 图片和部分 Android/WebView/下载相关路径，但这不代表
API 36、物理设备、移动/联通/电信或整个中国大陆可用。

所有已实施出口继续使用 exact-host、direct-first/DirectOnly 或能力门控的
严格 TLS 路径；没有证书关闭、SNI/Host 改写、固定 IP 安全核心、第三方
Pixiv API/图片反代、全局代理或请求体不明时跨 route 重放。当前 WebView
loopback/ECH 能力不足时保持显式 direct-only/unsupported。

## Remaining blockers

1. **API 36 设备门禁**：当前可核验 MuMu 是 API 35；没有 API 36 MuMu 镜像，
   API 36 WebView、MediaStore、headless Worker、intent/lifecycle 和完整
   真实账号矩阵不能宣称通过。
2. **全量测试门禁**：跨文件 loopback/Flutter runner 卡死仍可复现；聚焦测试
   和单文件测试通过，但不能填充成全量 pass。
3. **Live 外部能力**：当日真实账号 list 无 live object，无法取得 detail
   schema/HLS manifest；按 feasibility 设计保持未实现，不是 mock 缺失。
4. **Profile/Reverse Image 外部合约**：资料写入没有审定官方 route，反向搜图
   没有可接受 provider/credential/privacy 方案；应用保留可见 unavailable。
5. **可分发签名与 updater trust root**：release 仍是 debug signing，缺少
   production keystore、公钥、签名 manifest/APK 和匹配 signer；不能创建
   远程 release 或宣称更新安装成功。
6. **设备/变更覆盖**：真实 mutation、OAuth fresh exchange/refresh、系统重启
   widget restore、物理设备和运营商样本均需按 owning task 的范围补证；已登录
   账号本身不是阻塞理由。

## Conclusion

实现代码的主要叶子已经完成并有大量 API 35/单元证据，但 Replica v1 尚未
达到最终集成的真实可验收状态。最终 integration task 必须保持
`in_progress`，原父任务也不能完成或归档；以上 blocker 被解决并重新跑完整
矩阵前，不创建远程 release、不宣称大陆普遍可用。
