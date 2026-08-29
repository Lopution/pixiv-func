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
- **C 类（外部依赖，客户端无解）**：Profile 写入官方 route、反向搜图 provider。保持可见
  unavailable，不写 mock、不引依赖。Live 原属此类，2026-08-29 直接移出 Replica v1 范围——
  受众极小而成本不可控，继续挂在清单里只会伪装成「早晚要做」。

本轮**不解决任何一项**，只做归属与阻塞原因的如实记录。

### R6. 归档证据失效声明

2026-08-29 的清理使 6 个已归档叶子的证据与代码不再一致。归档按 `writeback_policy:
archives_read_only` 不回写，改为在本任务的 `research/archived-evidence-drift.md` 记录具体
文件与失效原因。

### R7. 真机试用暴露的 UI 缺陷

2026-08-29 用户在 RMX5200（API 36）上试用 `eb3a87f` 构建，报了 5 个问题。逐个查证后，其中
两个的根因比表面症状严重得多：**日期字段在每次实体合并时被静默丢弃**，以及**详情页宣称的
snapshot-first 契约在首次进入时根本不可达**。5 个都不是外观偏好，是缺陷。

| # | 症状 | 根因 |
|---|---|---|
| U1 | 主页顶部与系统状态栏重叠，其它 tab 正常 | `RecommendedIllustPage` 是唯一 `Scaffold` 既无 `appBar` 也无 `SafeArea` 的 tab；Ranking/New/Search/Settings 都有 AppBar，会自动吃掉 `MediaQuery.padding.top`。设备 targetSdk 36，Android 15+ **强制** edge-to-edge，而全仓库没有任何 `SystemChrome` / edge-to-edge 处理 |
| U2 | 所有作品都显示「投稿日期未知」 | `IllustEntity.copyWith` 不转发 `createDate`——它是构造器最后一个参数，也是 copyWith 唯一漏掉的字段。`IllustStore.mergeAll` 对每个已知实体都走 `copyWith`，`updateBookmark` 同样，所以详情响应一合并进来日期就没了 |
| U3 | 多图缩放不正常 | `InteractiveViewer(child: Center(child: PixivImage(fit: contain)))`：`Center` 给的是宽松约束，`RenderImage` 因此按**自身固有尺寸**（原始像素 ÷ DPR）布局而不是铺满视口，`BoxFit.contain` 没有可放大的目标。另：`_onTransformed` 每帧 `setState` 重建整个 `PageView` |
| U4 | 「再按一次退出」提示生硬 | 默认 `SnackBar` 显示 4 秒，而 `RootBackCoordinator.exitWindow` 是 1 秒——提示还在的时候早就不能连按退出了，提示本身在说谎 |
| U5 | 首次进入作品无动画，加载过一次才有 | 比动画严重：`IllustDetailController._load` 从不返回 `IllustDetailLoading`，且 `AsyncNotifier.build()` 返回 Future，首帧必然是 `AsyncLoading` → 页面渲染 spinner。**feed 早已把实体放进 `IllustStore`，却从不在首帧渲染**，所以既没有内容也没有 Hero 目标端，`IllustDetailLoading` 分支是死代码 |
| U6 | （用户截图中可见，未单独提出）简介渲染出字面量 `<br />` | Pixiv 简介是 HTML，当前直接塞进 `Text` |

已确定的处置（2026-08-29 用户决策）：

- **U1**：只加安全区顶部内边距，不加 AppBar——保留信息流的沉浸观感，只消除重叠。
- **U2**：修 `copyWith`；日期行与统计行的排版一并整理。
- **U3**：给图片视口紧约束，让 `contain` 有放大目标；顺带收敛每帧重建。
- **U4**：SnackBar 时长对齐 1 秒判定窗口，改 floating + 淡入淡出。
- **U5**：首帧直接渲染 `IllustStore` 快照，Hero 目标端随之在首帧存在，动画自然恢复。
- **U6**：简介渲染为**链接可点的富文本**（`<br>` 换行、实体解码、`<a href>` 可点）。站内
  pixiv 链接走已有内部路由（`showUserPage`、`IllustDetailPage`），站外链接需要一条出站
  intent 能力——当前 `AndroidIntentChannel` 只有入站，没有 outbound open-url。
- 简介入口做成**紧凑可点区块**（不占满宽度），位置不变。

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
- [ ] AC9：残余阻塞清单完成，6 项逐条有归属与解开条件；A/B/C 三类的处置差异写明。
      本轮不解决其中任何一项，也不得把任何一项标成已完成或不需要。
- [ ] AC10：U1–U6 全部修复，每项有**钉住根因而非症状**的回归测试：`copyWith` 保留
      `createDate` 且 `mergeAll` 往返不丢失；详情页首帧渲染 store 快照且 Hero 目标端存在；
      viewer 图片按视口而非固有尺寸布局；退出提示时长等于 `RootBackCoordinator.exitWindow`；
      简介 HTML 解码为 typed span。UI 改动按 memory 约定真机安装 + 截图留证。

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
- U2 与 U5 各自暴露一类**系统性**问题，不能只当单点缺陷修：前者是「构造器加了字段但
  `copyWith` 没跟上」，同类漏字段在其它实体上可能已经存在；后者是「注释宣称的契约与
  `AsyncNotifier` 的实际首帧行为不符」，同样的 `async.when(loading:)` 写法在其它 snapshot-first
  页面（Ranking/New/Search/Profile）可能重复。修复时需顺带排查同型，不做则如实记录。
