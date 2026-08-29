# 实施计划

## 顺序

1. **SNI spike**（先做，结论决定 Phase 2 形态）
   `test/tls_sni_behaviour_test.dart`：起 `ServerSocket`，`SecureSocket.connect` 到 `127.0.0.1`，
   服务端读首包、手写解析 ClientHello 的 `server_name`(ext 0x0000)。全离线、确定性，作为平台
   行为的长期文档留在仓库里。

2. **修回退分类**（前提，不修则后面看不到效果）
   `lib/core/network/compat/network_contracts.dart`：`isFallbackEligible` 加入 `tlsHandshake`；
   `certificateMismatch` 保持终止。回归测试钉死两者的区分。

3. **DNS wire codec**
   新增 `lib/core/network/compat/dns_message.dart`：RFC 8484 编解码，纯函数，无新依赖。
   先有编解码测试再接 resolver。

4. **DohResolver + 接线**
   `secure_resolver.dart` 新增 `DohResolver implements SecureResolver`；IP 字面量端点、有序
   失败切换、响应大小上限、超时、TTL 钳位、取消信号，复用 `isPublicNetworkAddress`。
   恢复 `DnsSource.doh`。改 `NetworkAccessPolicy` 默认 resolver，`SystemSecureResolver` 保留
   为可选与测试用。

5. **合并 route ladder**
   在 `NetworkAccessPolicy` 上加 `runLadder<T>({destination, purpose, cancelSignal, attempt})`；
   `PixivPolicyHttpClient.send` 与 `PolicyDownloadTransport.open` 各自收缩为一个 `attempt` 闭包。

6. **每主机路由记忆**
   有界 map + TTL，随 `setMode` / `advanceNetworkRevision` 与现有 `_closeClients()` 一起清空。

7. **分层探测**
   `lib/core/network/compat/network_probe.dart`（可测的纯逻辑 + 注入的 socket/resolver）
   \+ `lib/features/settings/network_probe_page.dart`（展示与复制）。

8. **设置入口 + i18n**
   `settings_page.dart` 加「网络」分组：`NetworkMode`（已有，目前只挂在 `login_page.dart`）、
   DoH 开关与端点、探测页入口。四语言补齐。

9. **Android 10**
   `android/app/build.gradle.kts` 的 `minSdk` 改为字面量 29；
   `AccountTransferClipboardChannel` 增加 sensitive-clipboard capability 上报，Dart 侧在 <33
   于导出页显式警告。

10. **归档证据失效清单**
    `research/archived-evidence-drift.md`。

11. **收编残余阻塞**
    `research/residual-blockers.md`：Widgets / Updater / 集成验收三条线的 7 个未完成项，
    按 A（用户可解）/ B（需密钥材料）/ C（外部依赖无解）分类，逐条写归属与解开条件。
    不解决、不冒充完成。

## R7：真机试用暴露的 UI 缺陷

与网络线无依赖，可独立并行推进。**先做 U2/U5，它们是数据与状态层的真 bug；U1/U3/U4 是
布局层，改完立刻可截图验证。**

12. **U2 · `createDate` 在合并时丢失**
    `lib/core/entity/illust_entity.dart`：`copyWith` 转发 `createDate`。
    然后**排查同型**——检查其它实体（`IllustUser`、`NovelEntity`、`CommentEntity` 等）是否
    存在「构造器加了字段但 copyWith 漏转发」。测试要钉住往返不变式而非单个字段：
    `mergeAll([e]) → get()` 应保留所有非合并语义字段。

13. **U5 · 详情页首帧渲染 store 快照**
    `lib/features/illust/detail/illust_detail_page.dart`：`AsyncLoading` 且
    `illustStoreProvider.get(id) != null` 时直接 `_buildContent`，不再无条件 spinner。
    删掉不可达的 `IllustDetailLoading` 分支或让 `_load` 真正产出它——二选一，不留死代码。
    Hero 目标端随之在首帧存在，动画自然恢复，无需改 `ReplicaPageRoute`。
    **排查同型**：Ranking/New/Search/Profile 是否有相同的 `async.when(loading:)` 吃掉快照。

14. **U1 · 主页安全区**
    `lib/features/home/recommended/recommended_illust_page.dart`：顶部加
    `MediaQuery.viewPadding.top` 的 `SliverPadding`（或 `SafeArea(bottom: false)`）。
    不加 AppBar。注意别把 `RefreshIndicator` 的触发区一起推下去。
    顺带确认 `home_page.dart` 的 `BottomAppBar` 在手势导航下的底部 inset。

15. **U3 · viewer 缩放**
    `lib/features/illust/viewer/image_viewer_page.dart`：给 `PixivImage` 视口紧约束
    （`SizedBox.expand` 或 `width/height: double.infinity`），让 `BoxFit.contain` 有放大目标。
    收敛 `_onTransformed` 的每帧 `setState`——只有 `_activeZoomed` 翻转时才需要重建
    （它唯一的消费者是 `physics`）。保持 `minScale`/`maxScale` 0.9–6.0 不变。

16. **U4 · 退出提示**
    `lib/features/home/home_page.dart`：SnackBar `duration` 绑定到
    `RootBackCoordinator.exitWindow`（不要写字面量 1 秒，两处必须同源），
    `behavior: floating` + 淡入淡出。提示的存在时间即「还能连按退出」的时间。

17. **U6 · 简介富文本**
    新增纯函数 HTML → typed span 解析（`<br>` 换行、实体解码、`<a href>`），与 Novel 的
    typed markup 同构，可离线单测；未知标签保持可观察，不静默吞掉。
    站内 pixiv 链接复用 `showUserPage`（`comment_item.dart`）与 `IllustDetailPage` 路由；
    站外链接需要给 `AndroidIntentChannel` 加一条**出站** open-url（当前只有入站），
    或引入 `url_launcher`——优先前者，与仓库既有的 channel 边界一致。
    简介入口改成紧凑可点区块（不占满宽度），位置不变。


## 验证

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
/opt/flutter-3.47.0/bin/flutter build apk --debug --flavor github
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-29-replica-v1-completion
git diff --check
```

新增测试：

| 文件 | 覆盖 |
|---|---|
| `test/tls_sni_behaviour_test.dart` | ClientHello 是否含 SNI（离线） |
| `test/dns_message_test.dart` | 编解码、截断、超长、TTL、非公网地址拒绝 |
| `test/doh_resolver_test.dart` | mock HTTP（外部边界）、TTL、超时、取消、端点切换、只对 Pixiv 主机生效 |
| `test/network_probe_test.dart` | 分层结果的确定性映射 |
| `test/restricted_compat_network_test.dart` | `tlsHandshake` 可回退 / `certificateMismatch` 不可回退；路由 memo；两出口共用 ladder |
| `test/settings_test.dart` | 新 i18n key 四语言齐全 |
| `test/illust_store_test.dart` | `mergeAll` → `get` 往返保留 `createDate`（钉住 copyWith 完整性，非单字段断言） |
| `test/illust_detail_page_test.dart` | store 有快照时首帧渲染内容而非 spinner；Hero 目标端在首帧存在 |
| `test/image_viewer_test.dart` | 图片按视口尺寸布局，不按固有尺寸；`_activeZoomed` 翻转才重建 |
| `test/home_page_test.dart` | 退出提示时长 == `RootBackCoordinator.exitWindow`；顶部内边距 == `viewPadding.top` |
| `test/illust_caption_test.dart` | HTML → typed span：`<br>`、实体、`<a href>`、未知标签保持可观察 |

已知环境不稳定（与本任务无关，改动前即存在）：`oauth_service_test.dart` token exchange 与
`download_manager_test.dart` real sockets，WSL 下 `dart:io` loopback 并发丢连接，测试文件自带
`tolerant()` 重试且标注 `(environment-flaky)`。

## 设备验证

分两类，不要混为一谈：

**R7 的 UI 改动——本任务自己验，必须留证。** 按 memory 约定真机安装 APK + 截图存到
`research/screenshots/`。RMX5200（API 36，arm64）已可用 adb 连接，`flutter devices` 因序列号
含空格解析失败，用 `adb -t <transport_id> install -r` 绕过。U1/U3/U4/U6 是肉眼可判的布局与
交互，改完必须有前后对比图；U2/U5 除截图外还要有回归测试。

**R1–R4 的网络与 Android 版本矩阵——由用户亲自执行，本任务不代跑。**
交付物是带探测页的 debug APK。需要的证据：

- 真实境内网络（系统 proxy/VPN 关闭、无外部代理 App）下的分层探测报告，按日期、运营商、
  网络类型、IP family 记录
- Android 10（API 29）与 API 36 各一台

探测结果回填后再决定 Phase 2 是否启动。**在此之前不得宣称大陆可用，也不得开始写 native 传输层。**

## Risky Files

- `lib/core/network/compat/network_policy.dart`（ladder 合并同时影响 API 与下载两个出口）
- `lib/core/network/compat/network_contracts.dart`（回退分类改动影响所有网络失败路径）
- `android/app/build.gradle.kts`（minSdk）
- `lib/core/entity/illust_entity.dart`（`copyWith` 是所有 store 合并的必经之路）
- `lib/features/illust/detail/illust_detail_page.dart`（首帧行为同时影响内容、Hero 与快照契约）

## Completion Gate

- 自动测试、模拟器、真机、真实账号四层证据明确区分，没有证据不声称完成。
- 不残留 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- Phase 2 未启动时，如实记录为"待实测决定"，不写成"已完成"或"不需要"。
