# 大陆无代理直连：Rust 传输层与策略阶梯

## 背景

三件事同时成立，才有了这个任务：

1. **前提变了**。`app-api` / `oauth.secure` / `accounts` / `www.pixiv.net` 已于 2026-04 迁到
   Cloudflare anycast（172.64.145.17 / 104.18.42.239），只有 `*.pximg.net` 还在
   210.140.139.x 源站。`08-29-replica-v1-completion` 的 design.md 规划的 Phase 2
   「省 SNI 直连源站」对 API 侧已经**不成立**——共享 anycast 无 SNI 选不出证书，
   握手必然失败。证据见 `research/third-party-clients-survey.md`。

2. **Phase 1 对墙内 API 侧零收益**。`08-29` 已完成 DoH resolver、route ladder、路由记忆、
   分层探测，但 `tls_sni_behaviour_test.dart` 的 spike 钉死了：Dart 对 IP 字面量 host 也会
   发送 SNI，`badCertificateCallback` 无法只放行主机名不匹配。**SNI 与 ECH 在 Dart 里没有旋钮**，
   DoH 只能解决 DNS 污染这一层。

3. **同技术栈的先例已验证**。PixEz Flutter 把全部出口换成 rhttp（Rust reqwest + rustls），
   并自行 fork 加了 ECH。ECH 是唯一**不牺牲任何安全性**的绕过路径：真实 SNI 被加密，
   outer SNI 是 `cloudflare-ech.com`，证书链与主机名校验全部保留，而我们的目标域名恰好
   全在 Cloudflare 上。

因此本任务把传输层下沉到 Rust，把「一档路由」从 `direct | secureDns` 扩展为携带 TLS 呈现
方式的策略档，并让 Dart 侧现有的 registry / ladder / 路由记忆 / 探测 / 诊断继续做唯一决策者。

## Requirements

### R1 传输层替换

- vendored fork 上游 `rhttp`（codeberg `Tienisto/rhttp`，MIT，0.18.0），放在 `plugins/rhttp/`，
  以 path 依赖引入。不使用 PixEz 的 GPL-3 fork。
- fork 的**唯一功能性改动**是 ECH 支持：接受一段 ECH config bytes，构建 rustls
  `EchMode` 并经 reqwest 的预配置 TLS 通道注入。DNS 解析、端点选择、TTL、取消一律不下沉。
- Dart 侧通过 `RhttpCompatibleClient`（`package:http` 的 `BaseClient` 实现）接入现有的
  `NetworkClientFactory`，其余调用方无感。

### R2 策略阶梯

- 路由档扩展为：`direct`（系统 DNS + 真实 SNI）、`dohRealSni`、`ech`、`noSni`、
  `insecureNoSni`（兜底，见 R6）。每一档 = DNS 来源 × TLS 呈现 × 证书校验开关。
- 按目的地分组默认顺序：
  - API / OAuth / accounts / www（Cloudflare）：`direct → ech → dohRealSni`
  - 图片 / 下载（`*.pximg.net` 源站）：`direct → dohRealSni → noSni`
- 复用现有 `runLadder` 与每主机路由记忆；记忆的键必须包含策略档，不能只记 IP。
- 一档失败到下一档的条件继续由 `TransportFailureClassifier.isFallbackEligible` 决定；
  `certificateMismatch` 保持终止语义。

### R3 ECH config 获取（留在 Dart）

- 扩展 `dns_message.dart`：支持 HTTPS RR（type 65）的 rdata 解析，取出 SvcParam `ech`（key 5）。
- 扩展 `DohResolver`：新增查询 ECH config 的能力，走与地址解析同一套端点、TTL、取消、
  大小上限逻辑。查询目标是 ECH 前置主机（Cloudflare 为 `cloudflare-ech.com`），可配置。
- 拿不到 config 时 `ech` 档直接判定不可用并降级到下一档，不静默变成普通 TLS。

### R4 三个出口统一

- API / OAuth：`PixivPolicyHttpClient` 经 `NetworkClientFactory` 自动切换。
- 图片：`PixivNetworkFactory.imageCacheManager` 的 `HttpFileService` 已经吃 `http.Client`，
  随 R1 自动跟随。
- 下载：`PolicyDownloadTransport` 目前直接用 `dart:io HttpClient`，需要迁到 rhttp 的流式
  响应，且必须保留现有的手动重定向校验、取消、断点续传语义。

### R5 探测页扩展

探测层要能把「哪一档能用」这件事测出来，而不只是给结论。新增层：

- ECH 可用性（能否取到 config；带 ECH 的握手是否成功）
- 空 SNI 握手
- HTTP 421 判定（链路通但证书/Host 选错，与握手失败必须区分）
- 每一档的最小真实请求结果（不带凭据）

报告仍需可复制导出，且要能直接读出「当前网络下应选哪一档」。

### R6 证书校验兜底档

- 默认全档保留完整链验证 + 按真实 host 的主机名校验。
- 允许一个 `insecureNoSni` 兜底档，**仅在用户于设置中显式开启后**才进入阶梯，
  开启入口必须有明确警告，且探测页与诊断中该档要显著标注。
- 默认关闭；不得因为探测失败而自动开启。

### R7 WebView 登录拦截（Android）

- Android 侧实现 `shouldInterceptRequest`：命中 Pixiv 域名的 GET 用与 R2 同一套策略重发，
  `Set-Cookie` 逐条注入 `CookieManager`，失败放行原生栈。
- 非 GET、未命中域名、拦截失败一律回落原生栈，不得吞掉错误。

### R8 设置与 i18n

- 网络设置页：模式选择、ECH 开关与前置主机、兜底档开关（带警告）、探测页入口、诊断导出。
- 新增文案四语言（zh / en / ja / ru）齐全。

### R9 平台范围

- 只保证 Android 构建与验收。Rust 插件本身跨平台，不得写入任何 Android-only 的假设
  （WebView 拦截除外，它天然是平台能力）。

## Acceptance Criteria

- [ ] AC1：`plugins/rhttp` 为上游 MIT fork，改动仅限 ECH 相关，且 `flutter build apk --debug --flavor github` 成功。
- [ ] AC2：`dns_message.dart` 能解析 HTTPS RR 的 `ech` SvcParam，覆盖截断、未知 key、
      多 SvcParam、无 ech 参数四类输入的离线测试。
- [ ] AC3：`DohResolver` 的 ECH 查询与地址查询共用端点/TTL/取消/上限逻辑，有 mock HTTP 测试。
- [ ] AC4：策略阶梯在离线测试中可断言顺序、降级条件、每主机记忆按「档」而非按 IP 生效。
- [ ] AC5：API、图片、下载三个出口共用同一策略决策，新增一档只需改一处；下载的重定向
      校验、取消、续传语义有回归测试。
- [ ] AC6：探测页能区分 ECH 可用 / 空 SNI 可用 / 421 / TCP 不通 / DNS 污染，离线测试钉住
      每种组合到结论的映射；真机可导出可复制报告。
- [ ] AC7：兜底档默认关闭，开启需用户显式操作且有警告；测试断言未开启时它不出现在阶梯里。
- [ ] AC8：WebView 拦截在 Android 上生效，Set-Cookie 多值不丢；非 GET 与失败路径回落原生栈。
- [ ] AC9：`flutter analyze` clean；`flutter test` 全量通过（WSL loopback 既有不稳定用例除外）。
- [ ] AC10：四语言 i18n 齐全，有测试钉住 key 完整性。
- [ ] AC11：真机验证由用户执行。**在用户回填真实境内网络的探测报告之前，不得宣称大陆可用。**

## Non-Goals

- 自建或依赖任何中继、代理、反代服务
- 第三方图片镜像（pixiv.re / pixiv.cat 等）作为默认路径
- iOS / desktop 的构建与验收
- 默认关闭证书校验
- QUIC / HTTP3 档（reqwest 有 feature，但本轮不启用，留待探测数据决定）
- 替换 SNI 档（reqwest 的 SNI 与 URL host 绑定，做不了；若探测证明 ECH 与空 SNI 都不可用，
  再另开任务评估域前置写法）

## Risks

- **ECH 未必可用**。GFW 对 ECH 握手有封锁报告，本任务未实测。阶梯必须在 ECH 失败时优雅降级，
  不能把整个方案押在它上面。
- **Rust 工具链是新的构建依赖**。本机当前没有 rustup / cargo，NDK 在 `/opt/android-sdk/ndk`。
  首次构建需要装工具链与 Android target，构建时间与产物体积都会增加。
- **fork 维护成本**。ECH 改动要能跟上游同步；改动面越小越好，这是 R1 限定「唯一功能性改动」
  的原因。
- **下载出口迁移风险最高**。它有断点续传、手动重定向校验、取消三层语义，且是唯一还在用
  `dart:io HttpClient` 的出口。
- **硬编码 IP 会腐化**。Cloudflare anycast 与源站 IP 都可能轮换，兜底表必须可被 DoH 结果覆盖，
  并在设置中可改。
- **WebView 拦截触及登录**。改坏了会让首次登录整体不可用；已有的账号导入（refresh token）
  是回退路径，必须保持可用。
