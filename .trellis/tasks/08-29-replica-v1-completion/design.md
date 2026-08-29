# 设计

## 现有架构的位置

大陆直连所需的骨架**已经存在**，这决定了 Phase 1 的形状是补齐而不是新建：

```
PixivPolicyHttpClient.send / PolicyDownloadTransport.open
  → NetworkRoute.direct        直连，原 URI
  → 失败且 eligible
  → NetworkAccessPolicy.resolve(destination)      ← resolver 在这里
  → 逐候选地址 NetworkRoute.secureDns
  → NativeStrictConnector.create(route)           ← socket 引到指定 IP
      connectionFactory 只改 TCP 目的地，URI 不变
      所以 SNI / Host / 证书主机名校验全部保持原样
```

缺的只有两处：

1. `resolve()` 背后是 `SystemSecureResolver`，即 `InternetAddress.lookup()`——在大陆就是被
   污染的系统 DNS。整条阶梯因此对墙内用户零收益。
2. 省 SNI 那一档不存在，且 `NativeStrictConnector` 走的是 Dart 的 `HttpClient`，SNI 与主机名
   校验是同一个旋钮（`SecureSocket.secure(host:)`），无法只关其一。

## 关键判断

### 为什么 DoH bootstrap 不需要任何证书妥协

DoH 端点用 IP 字面量 URL（`https://1.1.1.1/dns-query`）。这类端点的证书带 `iPAddress` SAN，
所以主机名校验对着 IP 正常通过，`badCertificateCallback` 根本不会触发。同时它不需要先做一次
DNS 解析来 bootstrap。两个性质都是免费拿到的。

### 为什么省 SNI 大概率必须下 native

Dart 的 `badCertificateCallback` 对**任何**校验失败都触发，且不告知原因，也只给叶子证书、
不给链。因此无法只放行「主机名不匹配」而继续拦住自签名中间人——要么退化成 SPKI pinning
（随 Pixiv 换证书而碎），要么自己重写一遍 WebPKI。

OkHttp 把 `HostnameVerifier` 与 `TrustManager` 分开，且更干净的做法是连 verifier 都不用换：
保持 URL 为真实主机名，只用自定义 `Dns` 喂地址、用 `SSLSocketFactory` 包装清空
`SSLParameters.serverNames`。于是 `Host` 头、证书链验证、主机名校验三者全部保持默认且正确，
唯一的变化就是 ClientHello 里不带 `server_name`。核心约 50 行。

但这个结论建立在「Dart 对 IP 字面量 host 仍会发送 SNI」之上，我不敢断言，所以先做 spike。

### 为什么 native 必须是 Application 作用域

widget 的后台刷新走 headless Flutter，`runWidgetBackground` 自己 boot 一个 `ProviderContainer`
（`lib/core/widget/widget_background.dart`），够不到 `MainActivity.configureFlutterEngine` 注册的
channel。现有 6 个 channel 全是 Activity 作用域。

这同时是 Phase 1 留在 Dart 的一个理由：Dart 侧的 resolver 对前台与 headless 一视同仁，零额外工作。

### 为什么回退分类的 bug 会让整条阶梯失效

`isFallbackEligible` = `{dns, connect, timeout, reset}`。GFW 在 TLS 握手期注入 RST，Dart 抛
`HandshakeException`，文本不含 cert/hostname 关键字，因此分类为 `tlsHandshake` → 不可回退 →
**DoH 那一档一次都不会被尝试**。这是接线之前必须先修的前提，否则 Phase 1 做完也看不到效果。

`certificateMismatch` 必须继续保持终止：它是链路上有人换证书的信号，重试只会换个 route 再被
同一个中间人接住。

## 数据流

```
读取型原生流量
  exact Pixiv destination
    → 路由记忆命中？跳过 direct
    → direct HTTPS
    → eligible failure（含 tlsHandshake）
    → DoH 解析（仅 registry 允许的主机）
    → 逐候选 IP：原 host 的 TLS + Host + 完整链验证
    → response / 分类后的 error

写操作 / OAuth
  先按路由记忆或严格 preflight 选定单一路径，body 只发一次；
  发送状态不确定后不得跨 route 自动重放。
```

## 探测页的设计意图

它不是"诊断功能"，它是**决定 Phase 2 是否存在的测量仪器**。因此分层必须能区分：

| 观察 | 结论 |
|---|---|
| 系统 DNS 与 DoH 结果不一致 | DNS 污染，Phase 1 足够 |
| DoH 有结果但 TCP 连不上 | IP 黑洞，客户端无解，需要中继（不做） |
| TCP 通、带真实 SNI 的 TLS 握手失败 | **SNI 被封，需要 Phase 2** |
| TLS 通但请求失败 | 应用层问题，与封锁无关 |

没有这张表就只能猜，而猜出来的 native 传输层就是下一个"为假想需求建的基础设施"。

## 风险与回滚

- 路由记忆若没有 TTL，用户换到可用网络后会被钉在慢路径上。TTL + revision 双重清除。
- DoH resolver 若被误用于非 Pixiv 主机就成了通用解析器；靠 `resolve()` 只接受
  `PixivDestination` 在类型上堵死，不靠运行时检查。
- ladder 合并会同时改动 API 与下载两个出口，`restricted_compat_network_test.dart` 与
  `download_manager_test.dart` 是回归护栏。
- `minSdk` 从 24 提到 29 会把 Android 7~9 排除在外——那些设备本来下载就是坏的，属于把
  名义支持面收敛到真实支持面。
