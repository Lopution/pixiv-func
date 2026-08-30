# 实施计划

## 起点

本任务以 `08-29-replica-v1-completion` 的**未提交工作树**为起点，不是从 HEAD 开始：
DoH resolver、`runLadder`、路由记忆、`dns_message.dart`、`network_probe.dart`、探测页与
网络设置页都在工作区里且已通过测试。**第 0 步先把那批改动按它自己的验收标准提交**，
让本任务的 diff 只包含传输层重构，否则两轮工作会混在一起无法回滚。

## 顺序

每一步都留一个可验证的落点，不合并推进。

0. **切出 08-29 的成果**
   `git status --porcelain` 逐项确认后提交 08-29 的 Phase 1 + R7 改动（含本轮已修的
   `new_page.dart` TabBar 对齐）。本任务从干净工作树开始。

1. **Rust 工具链**
   装 rustup + cargo，加 Android target（`aarch64-linux-android`、`armv7-linux-androideabi`、
   `x86_64-linux-android`）。NDK 在 `/opt/android-sdk/ndk`。
   验证落点：能 `cargo build --target aarch64-linux-android` 通过一个空 crate。
   **这一步失败则整条线阻塞**，先解决再继续。

2. **vendor rhttp fork**
   从上游 codeberg `Tienisto/rhttp` 0.18.0 取源码放到 `plugins/rhttp/`，`pubspec.yaml` 改 path 依赖。
   记录上游 commit 到 `plugins/rhttp/UPSTREAM.md`（后续同步的基准），保留其 MIT LICENSE。
   验证落点：`flutter build apk --debug --flavor github` 成功，一个 demo 请求能通。
   **此时不改任何业务代码**——先证明构建链路可用。

3. **DNS: HTTPS RR 解析**
   `lib/core/network/compat/dns_message.dart`：`_readAnswer` 保留 rdata；新增 SvcParams 解析，
   取 key 5（`ech`）。纯函数，无 I/O。
   验证落点：`test/dns_message_test.dart` 新增截断、未知 key、多 SvcParam、无 ech 四类用例。

4. **DohResolver: ECH config 查询**
   `secure_resolver.dart` 新增 `lookupEchConfig(frontHost)`，复用现有端点切换、退避、TTL 钳位、
   取消、大小上限。拿不到即返回空，由调用方决定跳档。
   验证落点：`test/doh_resolver_test.dart` 新增 mock HTTP 用例（TTL、超时、取消、端点切换）。

5. **Rust fork: ECH 注入**
   `TlsSettings` 增加可选 `echConfigList`（bytes）；Rust 侧据此构建 rustls `EchMode::Enable`
   并经 reqwest 预配置 TLS 通道注入。**fork 的唯一功能性改动，控制在这一处。**
   验证落点：Dart 侧能构造带 ECH 的 client 且不 panic；无 config 时行为与上游一致。

6. **策略档模型**
   `network_contracts.dart`：`NetworkRouteKind` 扩展为 `direct / ech / dohRealSni / noSni /
   insecureNoSni`；`NetworkRoute.key` 与诊断事件带上档位；`_HostRouteMemory` 按档记忆。
   验证落点：`test/restricted_compat_network_test.dart` 断言同 IP 不同档不共用池、记忆按档。

7. **rhttp client factory**
   新增 `lib/core/network/compat/rhttp_client_factory.dart`：每档映射到一组
   `ClientSettings`（DNS overrides / sni / verifyCertificates / echConfigList），
   实现 `NetworkClientFactory`。`StrictHttpClientFactory` 暂时保留。
   验证落点：单测断言每档产出的 settings 与预期一致（不发真实请求）。

8. **ladder 档位化**
   `runLadder` 按 purpose 选择默认档序列（Cloudflare 组 / 源站组），兜底档仅在设置开启时入列。
   验证落点：离线测试钉住顺序、降级条件、`certificateMismatch` 终止、兜底档 gating。

9. **下载出口迁移**
   `policy_download_transport.dart` 改用 rhttp 流式响应，保留手动重定向校验、取消、断点续传。
   验证落点：`test/download_manager_test.dart` / `download_recovery_test.dart` 全绿
   （WSL loopback 既有 flaky 用例除外）。

10. **删除旧传输**
    确认 API / 图片 / 下载三出口全部走 rhttp 后，删除 `NativeStrictConnector` 与
    `StrictHttpClientFactory`，不留死代码。

11. **探测页扩展**
    `network_probe.dart` 新增 ECH config 获取、带 ECH 的握手、空 SNI 握手、421 判定层；
    结论枚举扩展为「应选哪一档」。探测自身经 rhttp 走，否则测不到 ECH/空 SNI。
    验证落点：`test/network_probe_test.dart` 钉住每种层组合到结论的确定映射。

12. **设置与 i18n**
    网络设置页加：ECH 开关与前置主机、各档启用状态、兜底档开关（显式警告）、诊断导出。
    四语言补齐。
    验证落点：`test/settings_test.dart` 断言 key 四语言齐全 + 兜底档默认关闭。

13. **WebView 登录拦截（Android）**
    为登录页自写 Android PlatformView（原生 WebView），实现 `shouldInterceptRequest`：
    后台线程 → 主线程 MethodChannel → Dart 用同档 client 取字节 → 回传 →
    `Set-Cookie` 逐条注入 `CookieManager`。超时/非 GET/未命中一律返回 null 放行原生栈。
    `webview_flutter` 的现有实现保留为可切换回退。
    验证落点：登录流程在真机可完成；账号导入路径不受影响。

14. **全量验证与交付**
    见下方验证段；产出带探测页的 debug APK 交用户实测。

## 验证

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
/opt/flutter-3.47.0/bin/flutter build apk --debug --flavor github
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-29-cn-direct-transport
git diff --check
```

新增/改动测试：

| 文件 | 覆盖 |
|---|---|
| `test/dns_message_test.dart` | HTTPS RR rdata、SvcParams、ech key、截断与未知 key |
| `test/doh_resolver_test.dart` | ECH config 查询的 TTL / 超时 / 取消 / 端点切换 |
| `test/restricted_compat_network_test.dart` | 档位化 route key、按档路由记忆、ladder 顺序与降级、兜底档 gating |
| `test/rhttp_client_factory_test.dart`（新） | 每档 → ClientSettings 映射 |
| `test/network_probe_test.dart` | 新层与「应选哪一档」结论映射、421 与握手失败的区分 |
| `test/download_manager_test.dart` / `download_recovery_test.dart` | 流式迁移后的续传、取消、重定向校验 |
| `test/settings_test.dart` | 新 i18n key 四语言齐全、兜底档默认关闭 |

## 回滚点

- 第 2 步后若构建链路不可用 → 撤销 `plugins/rhttp` 与 pubspec 改动即可回到现状。
- 第 7–8 步之间 `StrictHttpClientFactory` 仍在，可以一行切回旧工厂。
- 第 10 步（删除旧传输）是不可逆点，**必须在三出口全绿之后**。
- 第 13 步 WebView 保留旧实现可切换，登录不会只剩新路。

## Risky Files

- `lib/core/network/compat/network_policy.dart`（ladder 与池化，同时影响三个出口）
- `lib/core/network/compat/network_contracts.dart`（route/档位模型，影响所有失败路径与诊断）
- `lib/core/network/compat/policy_download_transport.dart`（续传 + 重定向 + 取消三层语义）
- `plugins/rhttp/rust/src/api/client.rs`（fork 的唯一功能性改动，越小越好同步）
- `lib/features/login/login_webview_page.dart` + 新增 Android PlatformView（登录是关键路径）
- `pubspec.yaml` / `android/app/build.gradle.kts`（构建链路引入 Rust）

## 设备验证

沿用 memory 约定：UI 可见的改动真机安装 + 截图存 `research/screenshots/`。

网络部分**由用户在真实境内网络执行**，本任务不代跑，也不得以本机结果替代
（本机开着 TUN，任何可达性结论都无效）。需要的证据：

- 关闭系统代理/VPN 后的分层探测报告：每个主机每一档的结果，按日期、运营商、网络类型、
  IP family 记录
- API 与图片两组分别记录（它们在不同基础设施上，结论可能不同）

**在这份报告回填之前，不得宣称大陆可用。**

## Completion Gate

- 自动测试、模拟器、真机、真实境内网络四层证据分开记录，没有证据不声称完成。
- 不残留 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- ECH 若实测不可用，如实记录为「该档不可用」，不改成「已完成」或「不需要」。
