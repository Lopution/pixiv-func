# 实现受限兼容网络模式 — Implementation Plan

## Start Gate

- 上级父任务与本次开源审查规划已完成，Q1 已由用户确认允许 App 内部仅 Pixiv 域的兼容传输；实现仍须满足 capability 和严格 TLS Gate。
- 直接依赖任务需先完成、提交并归档；父任务执行顺序还要求当前 `08-26-ugoira-player-export` 在途边界完成后再启动本叶子，避免与其共享媒体文件并行修改。
- 当前任务经最终 planning review 和用户明确批准 `task.py start` 后才能开始。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 记录当日大陆网络事实与平台能力：按真实失败区分 DNS/SNI/connect/TLS；核验 Pixiv API/OAuth/accounts/pximg/Web/Live host；核验 AndroidX WebKit reverse bypass。分别对 Dart `connectionFactory + SecureSocket`、候选 ECH transport 做最小 API 36 spike，未通过不引依赖。
2. 建立 threat/decision matrix：direct、secure DNS、ECH、WebView loopback 各自能解决的问题；列出证书关闭、SNI/Host 替换、domain fronting、固定 IP、第三方反代、全局 override 和 BODY 日志拒绝证据。
3. 定义 `PixivDestinationRegistry`、`NetworkAccessPolicy`、`NetworkRevision`、`RouteAttempt`、failure taxonomy、resolver/health contracts 与脱敏 diagnostics。
4. 实现 system/direct baseline、approved DoH、A/AAAA/TTL/answer validation、network-bound health/circuit 和 original-host `NativeStrictConnector`；先用 bad-cert/host-mismatch/injected IP 测试证明严格 TLS。
5. 通过共享 factory 依次接入 OAuth exchange/refresh、`PixivHttpClient`、download/Ugoira 和 `CachedNetworkImage` file service；加入“未接入默认 client”搜索/测试，并证明 Updater、翻译、反向搜图 provider 不受影响。
6. 只有 ECH spike 满足 endpoint advertisement、`require ECH`、system trust、pool/cancel/stream 与 API 36 证据时才实现 `EchTransportAdapter`；否则记录 blocker并保持依赖图不变。
7. 只有在已批准的 App 内部范围且 WebKit exact-host reverse bypass 可证明时，才实现 loopback strict CONNECT parser、limits 与 `WebViewRouteSession` set/clear/refcount；否则明确删除该 implementation branch。
8. 接入 direct-first route selection、mutation preflight/no-replay、用户 `DirectOnly`/诊断状态与明确 failure UI；不更改 beta56 之外的页面结构。
9. 增加 canonicalization、DNS poisoning/rebinding、failure classification、route retry、network change/cancel、TLS、WebView lifecycle、secret/log 与非 Pixiv isolation 测试。
10. 运行全量质量门禁；在系统 proxy/VPN 关闭且无外部代理 App 的 API 36 设备执行 OAuth/API/image/download/WebView 矩阵，并按可取得的移动/联通/电信样本记录覆盖与 blocker。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-restricted-compat-network
git diff --check
```

Android 与构建：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- Exact-host/Unicode/suffix/trailing-dot/IP/userinfo/redirect/rebinding destination corpus。
- System vs DoH A/AAAA/TTL/bootstrap/size/cancel/network-revision；candidate IP + original-host SNI/cert 和 bad-cert/host-mismatch failure。
- `DNS/connect/timeout/reset/TLS/cancel/HTTP/auth/429/parse` 分类与 automatic eligibility；POST/mutation body 不自动重放。
- API/OAuth/refresh、CachedNetworkImage、Download/Ugoira 共用 factory；Updater/翻译/反向搜图永不进入 compat。
- ECH advertisement/required/fail-closed/system trust/pool/cancel/stream（仅采用时）。
- WebView capability/reverse-bypass/CONNECT parser/443/rate/size/loopback/set/clear/refcount/SNI 边界（在已批准范围内，仅能力满足时）。
- API 36 干净设备：system proxy=`none`、VPN=`off`、无外部代理 App；真实 OAuth、feed/detail/bookmark、pximg、download/Ugoira、accounts WebView。
- 大陆样本记录：日期、运营商、Wi-Fi/蜂窝、IPv4/IPv6、Android/WebView、route/failure；不足三运营商时禁止广泛可用声明。
- 安全搜索：无 `badCertificateCallback => true`、trust-all、hostname verifier true、SNI/Host replacement、fixed Pixiv IP、全局 `HttpOverrides`、请求/响应 BODY/Authorization 日志。

## Risky Files and Rollback Points

- `lib/core/network/compat/`、`pixiv_http_client.dart`、`oauth_service.dart`、download/Ugoira transport、project image file service、Android WebKit bridge、network settings/diagnostics、可选 transport dependency 与 `pubspec.lock`
- 先提交 contracts/strict direct tests，再接单一出口；每接一个出口均可独立回滚。ECH 与 WebView loopback 分别为可整体移除的 adapter。

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、单设备真机、真实账号和大陆运营商矩阵明确区分。
- 不残留未接入的 Pixiv 默认 client、placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 `trellis-check`，按需更新 spec，提交仅包含本任务文件，再 finish/archive。
