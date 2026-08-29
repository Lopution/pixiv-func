# 完成 Replica v1：大陆直连、Android 10 与收尾

## Background

Replica v1 的 24 个实现叶子与 5 个 hardening 叶子已完成归档。剩余工作此前散在 3 个卡在
`in_progress` 的叶子、5 个停在 `planning` 的 parent 和一个 trellis init 空壳里，没有单一
owner。本任务把它们全部收编。

同期发生了两件改变边界的事：

1. **2026-08-29 的过度防御代码清理**（`08-29-defensive-code-removal`）删除了登录导航拒绝、
   `networkRevision` 第三重闸门、剪贴板重放协议、`next_url` 参数白名单，以及 `webview_route`、
   `EchCapabilityGate`、`DohResolver`、WebKit 能力探测等一批无消费者的死基础设施。
2. **大陆访问的手段边界被推翻**。原规划把「关证书校验 / 省 SNI / 固定 IP」打包拒绝，导致
   兼容路径实际等价于直连、对墙内用户能力为零。三者风险差着数量级，现拆开处理。

## Requirements

### R1. 大陆直连范围与手段边界

- 范围是**登录后使用**：`app-api.pixiv.net`、`oauth.secure.pixiv.net`、`i.pximg.net`、
  `s.pximg.net` 四个 Pixiv 自有主机。不含第三方源站，不需要 WebView 参与。
- 首次注册与首次登录**不在范围内**，由「在有连通性的环境登录一次 + 已有的剪贴板账号迁移」
  承担。注册页会加载第三方 captcha 等本项目管不到的源站，无法用 Pixiv 主机的手段解决。
- 本项目**不提供任何网络服务**，不搭建中继或反代。
- **允许**：省略 SNI，前提是仍完成完整证书链验证并自行核对 SAN 命中目标主机名——那只是把
  主机名检查换个地方做。固定 IP 作为解析失败后的兜底。
- **拒绝**：关闭证书或主机名校验、改写 `Host`、domain fronting、全局 `HttpOverrides`、
  第三方 API/图片反代、通用代理。被审查的链路上按定义存在中间人，关掉链验证等于交出
  refresh token。不做 SAN 核对的省 SNI 与"接受一切证书"是同一个缺陷。

### R2. Phase 1（纯 Dart）

- `DohResolver` 重新实现并**接入 `NetworkAccessPolicy` 成为默认 resolver**（此前从未接线）。
  端点使用 IP 字面量 URL，bootstrap 不依赖任何 DNS，且这些端点证书带 `iPAddress` SAN，
  主机名校验正常通过——整个 bootstrap 零证书妥协。作用域由 `PixivDestinationRegistry` 天然
  收敛，不可能被用于非 Pixiv 主机。
- 修复 `TransportFailureClassifier.isFallbackEligible`：GFW 在握手期注入的 RST 在 Dart 里是
  `HandshakeException` → 分类 `tlsHandshake` → 当前不可回退 → **DoH 那一档永远不会被尝试**。
  加入 `tlsHandshake`；`certificateMismatch` 继续保持终止，那是中间人信号。
- 每主机路由记忆：境内直连必然失败，direct-first 会让每个请求都先付一次超时。
- 合并 `PixivPolicyHttpClient.send` 与 `PolicyDownloadTransport.open` 两条重复阶梯。
- **分出口分层探测页**：对 4 个主机逐层跑 系统 DNS → DoH → TCP → TLS（带真实 SNI）→ 最小
  真实请求，结果可复制。DoH 与系统 DNS 结果不一致 = DNS 污染；TCP 通但 TLS 握手失败 =
  SNI 被封，这就是决定是否需要 Phase 2 的信号。

### R3. Phase 2（条件触发，不预建）

只有当探测页在真实境内网络上显示「TCP 通、带真实 SNI 的 TLS 握手失败」时才启动。届时在
Kotlin 侧以 OkHttp 实现省 SNI 传输：自定义 `Dns` + 清空 `SSLParameters.serverNames` 的
`SSLSocketFactory` 包装，**默认 trust manager 与默认 hostname verifier 都保留**。
在实测证据出现前不写这部分代码——那正是本轮要根治的病。

### R4. Android 10 优雅降级

- `minSdk` 显式钉成 29。当前 `minSdk = flutter.minSdkVersion = 24`，但 `MediaStoreChannel`
  在 <29 直接抛异常，即 Android 7~9 装得上、能登录、**下载必然失败**。
- 剪贴板 `EXTRA_IS_SENSITIVE` 是 API 33 的能力，Android 10 上整个分支跳过，凭据明文进系统
  剪贴板且用户毫无提示。native 返回该 capability，导出页在 <33 时显式警告。这是安全降级，
  不能静默少做一件事。
- API 29 进验收矩阵。现有证据全部是 API 35，API 29 从未跑过。

### R5. 收编的残余阻塞

3 个归档时仍是 `in_progress` 的叶子（Widgets、Updater、集成验收）的未完成项全部转入本任务。
逐条展开见 `research/residual-blockers.md`，按「什么才能解开」分三类：

- **A 类（用户可解，需设备或真实账号）**：API 36 门禁、API 29 验收、重启后 widget restore、
  真实写操作与 fresh OAuth、大陆分层探测报告。本任务交付 debug APK，验证由用户执行。
- **B 类（需用户提供密钥材料）**：生产 keystore / 公钥 / 签名 manifest / 匹配 signer 均不存在，
  因此 updater 的验签成功→安装、安装拒绝、用户取消整条系统分支**无从验证**；release 仍是
  debug signing。密钥不进仓库，在到位前不用 mock 冒充验签结果。
- **C 类（外部依赖，客户端无解）**：Live（核验当日三个 filter 均 `lives=0`）、Profile 写入官方
  route、反向搜图 provider。保持可见 unavailable，不写 mock、不引依赖。

本轮**不解决任何一项**，只做归属与阻塞原因的如实记录。

### R6. 归档证据失效声明

2026-08-29 的清理使 6 个已归档叶子的证据与代码不再一致。归档按 `writeback_policy:
archives_read_only` 不回写，改为在本任务的 `research/archived-evidence-drift.md` 记录具体
文件与失效原因。

## Acceptance Criteria

- [ ] AC1：`DohResolver` 是 `NetworkAccessPolicy` 的默认 resolver，且有测试证明它只对 registry
      允许的 Pixiv 主机生效、TTL/超时/取消/端点切换均有界。
- [ ] AC2：回归测试钉死 `tlsHandshake` 可回退、`certificateMismatch` 不可回退。
- [ ] AC3：两个出口共用同一条 route ladder，新增一档只需改一处。
- [ ] AC4：探测页能在离线测试中给出确定的分层结果，并在真机上导出可复制报告。
- [ ] AC5：`minSdk = 29`；剪贴板在 <33 显式警告；四语言 i18n 齐全。
- [ ] AC6：`flutter analyze` clean，`flutter test` 全量通过（WSL loopback 环境不稳定用例除外）。
- [ ] AC7：归档证据失效清单完成。
- [ ] AC8：设备验证由用户亲自执行，结果回填后再决定 Phase 2。**未经实测不得宣称大陆可用。**
- [ ] AC9：残余阻塞清单完成，7 项逐条有归属与解开条件；A/B/C 三类的处置差异写明。
      本轮不解决其中任何一项，也不得把任何一项标成已完成或不需要。

## Non-Goals

- 首次注册、首次登录的境内直连
- 提供或依赖任何自建网络服务、中继、反代
- 关闭证书校验换取连通性
- 在实测证据出现前实现 Phase 2 的 native 传输
- 创建远程 release、上传发布物

## Risks

- 省 SNI 在 2026 年对 Pixiv 是否仍然有效未知。历史上有效，但 GFW 对「向可疑 IP 发无 SNI 的
  TLS」上过启发式。**只能实测，不能以历史结论代替。**
- 境外 DoH 端点在境内的可达性时通时不通；境内合规 DoH 对 pixiv 不会返回真实结果。
- `i.pximg.net` 的 CDN 边缘地址比 API 主机漂得快，固定兜底表会腐化。
- 本仓库无 API 36 与 API 29 镜像，两个版本的验收都依赖用户的真实设备。
