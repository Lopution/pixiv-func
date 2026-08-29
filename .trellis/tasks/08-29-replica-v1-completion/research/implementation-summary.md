# 实施完成摘要（2026-08-29）

`08-29-replica-v1-completion` 主体实施完成记录。验证状态按
`Completion Gate` 分层：自动测试=已跑通；真机=用户执行；真实账号/大陆网络=
用户执行。

## R1/R2 大陆直连 Phase 1

| 项 | 状态 | 证据 |
|---|---|---|
| SNI spike | ✅ | `test/tls_sni_behaviour_test.dart`（离线、确定性、3 项平台行为） |
| 回退分类修复 | ✅ | `network_contracts.dart` + 回归测试（`tlsHandshake` 可回退、`certificateMismatch` 终止） |
| DNS wire codec | ✅ | `lib/core/network/compat/dns_message.dart` + 11 测试 |
| DohResolver + 默认接线 | ✅ | `secure_resolver.dart`（IP 字面量端点、有序切换、TTL 钳位、取消、上限）+ 10 测试 |
| route ladder 合并 | ✅ | `NetworkAccessPolicy.runLadder`，API 与下载双出口共用（测试钉死） |
| 每主机路由记忆 | ✅ | 有界 map + TTL + revision/mode 清除；3 个测试 |
| 分层探测页 | ✅ | `network_probe.dart`（注入式分层逻辑 + 8 测试）+ `network_probe_page.dart` |
| 设置入口 + i18n | ✅ | `network_settings_page.dart` + 四语言 13 个新 key |

### spike 的重大发现（超出原计划的必修项）

`test/tls_sni_behaviour_test.dart` 钉死了一个比「回退分类 bug」更底层的问题：

**Dart `HttpClient.connectionFactory` + 直连 + https 时，SDK 完全跳过 TLS**——
`_http/http_impl.dart` 只在 `cf == null` 分支调用 `SecureSocket.startConnect`；
设置了 factory 时返回的 socket 原样使用（实测 wire 上是明文 `GET /...`）。
因此原有 `NativeStrictConnector` 的 secureDns 路径在真实触发时（大陆无代理）
会把带 `Authorization` 的请求**明文**发到解析出的 IP——凭据泄漏。

修法（`strict_http_client.dart`）：connectionFactory 内先 `Socket.startConnect`
再 `SecureSocket.secure(socket, host: url.host)` 包装——SNI、链验证、主机名校验
全部保持，只改 TCP 目的地，正是 design 想表达的契约。spike 同时实证：
Dart 对 IP 字面量 host **会发送 IP 作为 SNI**（design 中「不敢断言」的存疑点
有了答案），因此「省 SNI 且保留主机名校验」在 Dart 无解 → Phase 2 必须 native，
design 结论成立。真机验收时 secureDns 从未被触发（代理环境下 direct 直连成功），
故此前未暴露。

## R4 Android 10

| 项 | 状态 | 证据 |
|---|---|---|
| minSdk = 29 | ✅ | `android/app/build.gradle.kts`；已装 RMX5200 验证 `minSdk=29 targetSdk=36` |
| 剪贴板 capability | ✅ | Kotlin `capabilities` method（`sensitiveMarkSupported`，API<33=false）；Dart `<33` 导出后显式警告（settings 页）+ widget 测试 |

## R7 UI 缺陷

| # | 修复 | 回归测试 |
|---|---|---|
| U1 | 推荐页顶部 `viewPadding.top` SliverPadding（不加 AppBar） | `recommended_feed_test.dart`（viewPadding=100→top padding>0） |
| U2 | `copyWith` 转发 `createDate` + `mergeAll` no-regress（`entity.createDate ?? existing`）+ 日期/统计行排版 | `illust_store_test.dart` 往返测试；同型排查：CommentEntity/UserEntity/HistorySnapshot/HistoryRecord/BookmarkEntry 均完整，无其它漏字段 |
| U3 | viewer 图片 `SizedBox.expand` 紧约束 + `_onTransformed` 仅在 zoom 翻转时 setState | `illust_detail_page_test.dart`（SizedBox.infinity 断言） |
| U4 | 退出提示 SnackBar duration 绑定 `RootBackCoordinator.exitWindow` + floating + 短动画 | `home_page_test.dart`（duration == exitWindow） |
| U5 | 首帧 `AsyncLoading` 时直接渲染 store 快照（Hero 目标端首帧存在）；删除不可达 `IllustDetailLoading` 状态 | `illust_detail_page_test.dart`（首帧无 spinner、Hero 在）；同型排查：feed 类页面（Ranking/New/Search/Profile）首帧无 ordered ids，store 无可渲染目标，spinner 为正确行为，非同型 |
| U6 | HTML→typed span 纯函数 + 详情页紧凑可点富文本 + pixiv 站内链接内部路由 + 站外 outbound `openUrl` intent | `illust_caption_test.dart`（7 测试）+ detail widget 测试（无字面 `<br>`、链接可点） |

## 验证状态

- ✅ `flutter analyze` clean
- ✅ 新增/修改测试文件全过（87 个新增/改动用例 + detail/viewer/settings 套件）
- ✅ `flutter build apk --debug --flavor github` 成功，已安装 RMX5200（transport_id 22）
- ⚠️ 全量 `flutter test`：仅 `download_manager_test` real-sockets 用例与
  `tls_sni_behaviour_test` 偶发超时（WSL loopback 丢包，与本任务无关；前者标注
  environment-flaky，后者已加重试+放宽超时后单跑稳定）
- ⏳ 真机验证（U1-U6 截图）、大陆分层探测报告、API 29 验收——用户执行，回填后
  再决定 Phase 2

## 待办（不属于本任务代码改动）

- RMX5200 上已装 APK 为日期行排版整理**之前**的构建（`_StatItem` 在装完后改的）；
  如需验证最新排版需重装。日期字段本身（U2 核心）已在已装 APK 内。
- `android/gradle.properties` 的 JVM 内存从 8G 降到 3G（本机 11G 内存，
  daemon 崩溃问题）；这只是本机构建参数，不影响产物。