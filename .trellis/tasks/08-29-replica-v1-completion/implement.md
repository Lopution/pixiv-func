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

已知环境不稳定（与本任务无关，改动前即存在）：`oauth_service_test.dart` token exchange 与
`download_manager_test.dart` real sockets，WSL 下 `dart:io` loopback 并发丢连接，测试文件自带
`tolerant()` 重试且标注 `(environment-flaky)`。

## 设备验证

**由用户亲自执行，本任务不代跑。** 交付物是带探测页的 debug APK。需要的证据：

- 真实境内网络（系统 proxy/VPN 关闭、无外部代理 App）下的分层探测报告，按日期、运营商、
  网络类型、IP family 记录
- Android 10（API 29）与 API 36 各一台

探测结果回填后再决定 Phase 2 是否启动。**在此之前不得宣称大陆可用，也不得开始写 native 传输层。**

## Risky Files

- `lib/core/network/compat/network_policy.dart`（ladder 合并同时影响 API 与下载两个出口）
- `lib/core/network/compat/network_contracts.dart`（回退分类改动影响所有网络失败路径）
- `android/app/build.gradle.kts`（minSdk）

## Completion Gate

- 自动测试、模拟器、真机、真实账号四层证据明确区分，没有证据不声称完成。
- 不残留 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- Phase 2 未启动时，如实记录为"待实测决定"，不写成"已完成"或"不需要"。
