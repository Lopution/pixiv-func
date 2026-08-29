# Implementation evidence

验证日期：2026-08-28（Asia/Shanghai）。本记录只描述本叶子新增实现和实际运行范围，不把 API 35 结果扩大成 API 36 或物理设备覆盖。

## Implemented

- Flutter 生成 schema v1、account-revision keyed、无凭据的 `WidgetSnapshot`；`WidgetSnapshotStore` 使用 `.write.lock`、临时文件和最后切换的 `active.json`，写入失败保留同账号 last-good，切换/退出/reauth 清除旧 render state。
- Widget feed 复用共享 `PixivHttpClient`、image destination policy 和 `NetworkAccessPolicy`，以 account、credential revision、network revision 三重边界阻止旧生成发布。
- Native 实现 Recommend/Refresh providers、RemoteViews 圆角封面、唯一 PendingIntent、typed `pixivfunc://illusts/<id>` deep link、WorkManager 30 分钟 periodic 与受限 one-shot refresh；不在 native 读取账号、credential、token、cookie 或建立独立网络栈。
- Snapshot/native reader 对 schema、整数、路径、文本、文件、年龄和尺寸做边界校验；RemoteViews 解码同时受单图 pixel、总 bitmap bytes 和尺寸预算限制。

## Compiled

通过：

```text
/opt/flutter-3.47.0/bin/flutter analyze
No issues found

./gradlew :app:compileDebugKotlin :app:testDebugUnitTest --no-daemon
BUILD SUCCESSFUL

/opt/flutter-3.47.0/bin/flutter build apk --debug
Built build/app/outputs/flutter-apk/app-debug.apk
SHA-256: 2ec7b14ce70b703fec85a6566398e71dc6cbea37b3cdb41a7c42f78c970f732c
```

## Unit-tested

- Flutter widget focused suite：28 tests passed：
  `test/widget_feed_loader_test.dart`、`test/widget_snapshot_test.dart`、`test/widget_snapshot_store_test.dart`。
- Android JVM widget suite：13 tests passed，覆盖 schema/年龄、atomic staging、budget overflow、unique work names/retry bounds。
- 全量 `flutter test --reporter compact`：到 `+375`，但既有 `test/download_manager_test.dart` 的 `HttpDownloadTransport over real sockets ... cancel terminates a real in-flight transfer` 在 30 秒测试超时，退出码 1；未通过该全量门禁，不能宣称全量测试通过。

## Device-tested

验证设备先通过 MuMu 管理器确认，选用 `127.0.0.1:16384`，并在每次设备验证前执行：

```bash
adb devices -l
adb -s 127.0.0.1:16384 get-state
adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk
adb -s 127.0.0.1:16384 shell settings get global http_proxy
adb -s 127.0.0.1:16384 shell dumpsys connectivity
```

观察结果：两个 ADB 端点均显示 `device`；选定端点 SDK `35`；系统 proxy 为 `null`；默认网络为 Wi-Fi、`NOT_VPN&VALIDATED`，地址为 MuMu NAT 网段 `10.0.2.15`。包查询未发现 Clash/V2Ray/Shadowsocks/Surfboard/Sing-box/WireGuard/VPN/proxy 类外部代理 App。

结论标记：`MuMu emulator-tested, not physical-device-tested`。

在最终 debug APK 安装后，API 35 MuMu 实际观察到：

- Launcher 中已存在 Recommend 和 Refresh 两个 widget（AppWidget IDs 15、14）；推荐封面可见，刷新图标可点击。
- `am kill io.github.lopution.pixivfunc` 后确认 PID 消失，再点击 Refresh widget；系统重新创建 headless Flutter 进程，日志出现 `WidgetBackground: entered`、`loader built`、`outcome written`，随后 `Worker result SUCCESS`，AppWidget IDs 15/14 被通知更新，桌面封面实际变化。
- 快速连续点击 Refresh 的历史验证显示只有一个 WorkManager work 启动，其余为 `is already enqueued for processing`，证明 `KEEP` 合并行为。
- Recommend 封面点击实际进入应用的图片/Detail 路径；带额外 query 的恶意 deep link 未推进到作品页面并显示拒绝提示。
- `run-as` 检查的 widget snapshot 只含 schema、截断 account key、revision、作品/作者展示字段和受控图片文件名；没有 token、cookie、credential 或明文账号字段。最终 snapshot 为 8 个图片文件，periodic name 为 revision keyed 的 `widget_recommend_periodic_r0`。

可复查截图：

- [冷启动](./screenshots/01-app-cold-start.png)
- [真实推荐封面](./screenshots/02-recommend-widget-real-cover.png)
- [widget 点击 deep link](./screenshots/03-widget-click-deeplink.png)
- [前台刷新](./screenshots/04-foreground-refresh-widget.png)
- [刷新 widget 已放置](./screenshots/05-refresh-widget-placed.png)
- [点击刷新后重新生成](./screenshots/06-refresh-click-regenerated.png)
- [resize](./screenshots/07-widget-resized.png)
- [移除 Recommend 后保留 Refresh](./screenshots/08-recommend-removed-refresh-kept.png)

## Real API/account boundary

使用 MuMu 中现有的真实登录会话，推荐 API 和 `i.pximg.net` 图片封面均实际加载成功；这不是 mock 或假数据。没有重新执行 OAuth、token refresh、收藏/关注/评论/资料写入等变更操作，因此这些动作单独保持未验证，不把“已有真实登录会话”错误写成登录链路 blocker。

## Unverified / blockers

- 当前 MuMu 镜像是 API 35；没有 API 36 MuMu 镜像，PRD 要求的 API 36 真机验收仍是 blocker，不能宣称 API 36 已通过。
- MuMu 使用宿主机 NAT，以上结果不代表三大运营商、物理手机、不同 Android/WebView/OEM 的覆盖；没有宣称 carrier 或 physical-device 覆盖。
- 本次未做系统重启后的完整 widget restore 矩阵；已做 app 进程死亡后的 headless worker、resize 和 Recommend 移除/Refresh 保留验证。
- 全量 Flutter 测试仍受既有 loopback cancellation test timeout 影响；widget focused tests、Android JVM tests、analyze、debug build 均有独立通过证据。
