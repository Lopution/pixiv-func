# 设计

## 现有架构的接缝

Phase 1 留下的结构不需要推翻，它恰好把传输实现收敛到了一个函数类型上：

```
PixivPolicyHttpClient.send          PolicyDownloadTransport.open
        └────────────┬────────────────────────┘
        NetworkAccessPolicy.runLadder(destination, canReplay, attempt)
                     │
                     ├─ 选档：direct / 路由记忆命中的档
                     └─ attempt(route, url)
                            ├─ API/图片：policy.clientFor(purpose, route)
                            │              └─ _clientFactory(route)      ← 唯一替换点
                            └─ 下载：NativeStrictConnector.create(route)  ← 唯一例外
```

- `NetworkClientFactory = http.Client Function(NetworkRoute route)`（[network_policy.dart:12](../../../lib/core/network/compat/network_policy.dart#L12)）
- 图片走 `PixivNetworkFactory.imageCacheManager` → `HttpFileService(httpClient: client(image))`，
  吃的是 `http.Client`，随替换点自动跟随，**不需要改 `PixivImage`**
- 下载走 `PolicyDownloadTransport._transportFor` → `dart:io HttpClient`，是唯一不吃
  `http.Client` 的出口，必须单独迁移

`rhttp` 的 `RhttpCompatibleClient` 是 `package:http` 的 `BaseClient` 实现，且有
`createSync`，与 `NetworkClientFactory` 的同步签名天然对齐。

## 关键判断

### 为什么 DNS 与 ECH config 都留在 Dart

PixEz 的 fork 把「用阿里 DNS 查 `cloudflare-ech.com` 的 HTTPS RR」写死在 Rust 里。我们不这么做：

- Dart 侧已经有 `DohResolver`：端点有序切换、失败退避、TTL 钳位、取消信号、响应大小上限、
  只对 registry 允许的主机生效——全部离线可测。在 Rust 里重写一遍等于放弃这些测试。
- fork 与上游的差异面越小越好。DNS 下沉会把端点策略、缓存、取消都拖进 Rust。
- 因此 Rust 侧只做纯执行者：接收 `DnsSettings.static(overrides)`（Dart 解析好的 IP）
  和一段 ECH config bytes，其余不知情。

**fork 的功能性改动因此收敛为：`TlsSettings` 多一个可选的 `echConfigList`，
Rust 侧据此构建 rustls `EchMode::Enable` 并经 reqwest 预配置 TLS 注入。**

代价是 Dart 侧要能解析 HTTPS RR（type 65）。现有 `dns_message.dart` 的 `_readAnswer`
对非 A/AAAA 记录只保留 type/ttl，**丢弃了 rdata**，所以需要：保留 rdata → 解析
SvcParams → 取 key 5（`ech`）。这是纯函数改动，可离线测。

### 为什么策略档必须进 `NetworkRoute`

`route.key` 已经同时是三样东西的键：客户端池（`_clients`）、每主机路由记忆（`_routeMemory`）、
诊断事件。同一个 IP 用真实 SNI 和用 ECH 是两条物理上不同的路，如果策略不在 route 里，
池化会串档、记忆会记错、诊断会把两档的失败混在一起。

因此 `NetworkRouteKind` 从 `{direct, secureDns}` 扩展为策略档，`key` 与
`_HostRouteMemory` 一并按档区分。

### 为什么 ECH 档不绑定固定 IP

ECH 走 Cloudflare anycast，`DohResolver` 已经能解析到 104.18.x / 172.64.x。硬编码表只在
DoH 全败时兜底，且必须可被设置覆盖——三家客户端都踩过「硬编码 IP 腐化」这一坑
（`research/third-party-clients-survey.md` §5.5）。

### 为什么 direct 仍然是第一档

墙外与代理环境下 direct 最快，且不产生任何非常规 ClientHello。墙内的代价（每次先付一次
超时）已经由既有的每主机路由记忆消掉：一旦某档成功，后续请求直接从该档起步。

### 兜底档为什么不是「自动降级的最后一档」

`insecureNoSni` 关闭的是主机名校验与链验证，它一旦进入自动降级序列，一个能让前面所有档
失败的中间人就能把连接推到这一档并静默成功。所以它的语义是**「用户显式打开才存在」**，
不是「其它都失败后自动尝试」。探测页与诊断中必须显著标注该档，让「我现在正跑在不校验
证书的路径上」始终可见。

### WebView 的边界：一个真实的架构裂缝

`webview_flutter_android` 4.14 的 `WebViewClient`（`pigeons/android_webkit.dart:463`）只暴露
`onPageStarted` / `onPageFinished` / `onReceivedError` / `onReceivedHttpError` /
`shouldOverrideUrlLoading` 一类通知回调，**没有 `shouldInterceptRequest`**。原因是结构性的：
Android 在非 UI 线程**同步**调用它并要求同步返回 `WebResourceResponse`，而 pigeon 通道是异步的。

三条路：

| 方案 | 代价 |
|---|---|
| A. 为登录页自写 Android PlatformView（原生 WebView + 拦截） | 登录相关能力要自己实现；但登录 WebView 的需求很窄（load、URL 变化、cookie、返回键） |
| B. vendored fork `webview_flutter_android` 加 native 钩子 | 再多一个 fork 要维护，且它随 Flutter 版本变动 |
| C. 反射替换 plugin 内部的 WebViewClient | 脆弱，随 plugin 小版本碎 |

**选 A**，且只用于登录页；`webview_flutter` 的现有实现保留为可切换的回退，登录是关键路径，
不接受「改了就只剩新路」。

拦截内部还有一个选择：拦到的请求由谁重发。

- **Kotlin/OkHttp 自己发**：与 Rust 侧策略不同源，且 Android 的 `SSLSocket` 做不了 ECH，
  这条路只能用空 SNI / 自定义 DNS。
- **回到 Dart 用同一套策略发**：`shouldInterceptRequest` 在后台线程，可以阻塞等待；
  Kotlin post 到主线程发起 MethodChannel、Dart 用与 API 出口相同的 client 拉取字节后回传，
  后台线程带超时地等结果。策略完全同源，代价是一次跨线程往返和死锁风险（必须有超时）。

**选后者**，因为「同源」正是这个任务的价值：登录页与 API 走同一条被探测证明可用的档。
Kotlin/OkHttp 直发作为超时后的降级（放行原生栈本身就是最终降级）。

## 数据流

```
读取型请求（API / 图片 / 下载）
  registry.require(uri, purpose)                     ← 目的地白名单不变
  → runLadder
      ├ 路由记忆命中该 host？→ 从记住的档起步
      ├ 档 1 direct        : 系统 DNS + 真实 SNI + 完整校验
      ├ 档 2 ech           : DoH 地址 + ECH(outer=cloudflare-ech.com) + 完整校验
      ├ 档 3 dohRealSni    : DoH 地址 + 真实 SNI + 完整校验
      ├ 档 4 noSni         : DoH/兜底源站 IP + 空 SNI + 完整校验
      └ 档 5 insecureNoSni : 同上但关校验（仅用户显式开启）
  → 成功：记住 (host → 档)，TTL + revision 双重失效
  → 失败：按 isFallbackEligible 决定继续还是终止（certificateMismatch 永远终止）

ECH config
  DohResolver.lookupEchConfig(frontHost)
    → HTTPS RR (type 65) → SvcParams → key 5 (ech) → bytes + TTL
    → 缺失/过期 → ech 档标记不可用，跳过（不静默退化为普通 TLS）

WebView 登录（Android）
  原生 WebView.shouldInterceptRequest (后台线程)
    → 命中 Pixiv 域名的 GET？→ 主线程 MethodChannel → Dart 用同档 client 取字节
    → 回传 body + headers → Set-Cookie 注入 CookieManager → WebResourceResponse
    → 未命中 / 非 GET / 超时 / 失败 → 返回 null，放行原生栈
```

## 与 Phase 1 已有代码的关系

| 现有件 | 处置 |
|---|---|
| `PixivDestinationRegistry` | 不变，继续是唯一的目的地白名单 |
| `DohResolver` / `SystemSecureResolver` | 保留，新增 ECH config 查询 |
| `dns_message.dart` | 扩展：保留 rdata + SvcParams 解析 |
| `runLadder` / `_HostRouteMemory` | 保留，档位化 |
| `TransportFailureClassifier` | 保留；需要能识别 rhttp 抛出的错误类型 |
| `NetworkDiagnostics` | 保留，事件新增「档」字段 |
| `NativeStrictConnector` | 下载迁移完成后删除（不留死代码） |
| `NetworkProbe` | 扩展新层；probe 自身也要经 Rust 传输才能测 ECH/空 SNI |
| `network_probe_page` / `network_settings_page` | 扩展展示与开关 |

## 兼容性与回滚

- Rust 传输层是构建期依赖，一旦引入无法在运行时回退到 `dart:io`。因此
  `NetworkClientFactory` 的抽象必须保留，且 `StrictHttpClientFactory` 在迁移期内不删除，
  直到下载出口也迁完、全链路测试通过。
- 每一档都可以在设置里单独关闭，最差情况用户可以只留 `direct`，行为等价于今天。
- 账号导入（refresh token）路径必须始终可用，它是 WebView 改动的安全网。

## 风险点

- **ECH 可用性未经实测**。设计上 ECH 只是阶梯中的一档，失败即降级，不会让方案整体失效。
- **Rust 工具链**：本机无 rustup/cargo，NDK 在 `/opt/android-sdk/ndk`。首次构建要装
  toolchain + Android target，构建时长与 APK 体积都会涨。
- **下载出口**：断点续传、手动重定向校验、取消三层语义叠在一起，是迁移风险最高的一处。
- **WebView 同步桥接**：后台线程阻塞等待主线程结果，必须有超时与失败放行，否则登录页会卡死。
