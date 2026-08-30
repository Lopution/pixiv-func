# 第三方 Pixiv 客户端的大陆直连实现调研

> 调研日期：2026-08-29。所有源码结论来自当日 `--depth 1` clone 的仓库快照。
> 本机开着 TUN 代理，**任何可达性结论都不来自本机实测**；本文只记录源码事实、
> 公开 DNS 事实，以及由此推出的设计含义。

## 1. 前提事实：API 域���已迁移到 Cloudflare

用 Cloudflare DoH（`https://1.1.1.1/dns-query`，`application/dns-json`）查权威答案。
这一项与本机网络路径无关，查的是全球解析结果：

| 域名 | A 记录 | 基础设施 |
|---|---|---|
| `app-api.pixiv.net` | 172.64.145.17, 104.18.42.239 | Cloudflare anycast |
| `oauth.secure.pixiv.net` | 172.64.145.17, 104.18.42.239 | Cloudflare anycast |
| `accounts.pixiv.net` | 104.18.42.239, 172.64.145.17 | Cloudflare anycast |
| `www.pixiv.net` | 172.64.145.17, 104.18.42.239 | Cloudflare anycast |
| `i.pximg.net` | 210.140.139.129–138（10 个） | Pixiv 自有源站（IDC Frontier JP） |

Shaft 源码注释独立佐证了时间点：`HttpDns.java` 写着
「Pixiv API/OAuth 已迁移至 Cloudflare CDN (2026-04)」，`FALLBACK_IMAGE_IPS` 仍是
210.140.139.x 并注明「图片服务器还在旧 Pixiv 基础设施」。

**设计含义**：`08-29-replica-v1-completion` 的 design.md 规划的 Phase 2「省 SNI 直连源站」
对 API 侧已经不成立——共享 anycast 在无 SNI 时选不出证书，握手必然失败。省 SNI 只对
仍在源站的 `*.pximg.net` 有效。API 与图片必须分成两套策略。

---

## 2. Pix-EzViewer（Kotlin / OkHttp）

仓库：`ultranity/Pix-EzViewer`（PixEz 的 Kotlin 分支，README 主打「免代理直连」）。

### 二维正交模型

`app/src/main/java/com/perol/asdpl/pixivez/networks/NetworkMode.kt` 把问题拆成两个维度：

- **DNS 维度** `DnsMode`：`DIRECT`（硬编码源站 IP）/ `DOH`（Cloudflare anycast）/ `SYSTEM`
- **SNI 维度** `SniMode`：`REPLACE`（替换为前置域名）/ `EMPTY`（空 SNI）/ `PLAIN`（真实 SNI）

源码注释明确写出两者的物理耦合：

> Pixiv 自有源站（IDC Frontier/JP，如 210.140.139.x）：nginx 直接服务，接受无 SNI 并按
> Host 路由；Cloudflare 共享 anycast（104.x/172.64.x）：无 SNI 时选不出证书 → TLS 握手失败。
> 故空/替换 SNI 须配源站，明文 SNI 配 DoH/anycast。

墙内默认组合 = `DIRECT` + `REPLACE(pixiv.me)`。

硬编码直连 IP（`PixivDirectDns`，可经设置 `apiDirectIPs` 覆盖）：
`210.140.139.155/156/157/158`，仅对 `app-api.pixiv.net`、`oauth.secure.pixiv.net`、
`accounts.pixiv.net` 生效，其余交还系统 DNS。

### 替换 SNI 不需要牺牲证书校验

`ReplaceSniSocketFactory.kt` 只有 79 行，核心是分层重载：

```kotlin
override fun createSocket(socket: Socket?, host: String?, port: Int, autoClose: Boolean): Socket {
    val ip = socket!!.inetAddress.hostAddress          // 用 IP 作 peerHost：IP 不会被自动当 SNI
    return (delegate.createSocket(socket, ip, port, autoClose) as SSLSocket).apply {
        sslParameters = sslParameters.apply { serverNames = listOf(SNIHostName(sniHost)) }
    }
}
```

`VerifyConfig` 默认 **true**，即保留系统信任链 + 默认主机名校验。注释解释了为什么能这样：

> REPLACE 下源站返回的多-SAN 证书本就覆盖目标 Host 且链到可信 CA，默认校验可通过。

也就是说 hostname verifier 是按 **URL host** 比对证书 SAN 的，与 ClientHello 里放什么无关。
这直接解掉了 `08-29` design.md 里「要么放弃校验、要么 SPKI pinning」的两难。

### 自适应选 SNI

`SniReplaceConfig.autoSelect()`：
1. 用空 SNI 握一次手，从对端证书读 SAN 列表作候选（`filterNot { startsWith("*.") }`，
   非 `pixiv.net` 结尾的排前面——更可能逃过 GFW 对 pixiv.net 的 SNI 过滤）
2. 兜底候选：`pixiv.me`, `www.pixivision.net`, `fanbox.cc`,
   `public-api.secure.pixiv.net`, `oauth.secure.pixiv.net`, `pixiv.net`
3. 逐个实测：连源站 + 该 SNI，**握手未被 RST + HTTP ≠ 421 + 对端证书确为 Pixiv**
   （防 captive portal 用非-Pixiv 证书冒充成功）
4. 第一个通过的写入 pref

421 是关键判据：源站按 Host 路由，如果选中的证书不覆盖目标 Host，nginx 回 421 Misdirected Request。

### WebView 绕过

`networks/bypass/WebViewBypassInterceptor.kt`：`shouldInterceptRequest` 里拦 GET，
用自定义 DNS + SNI 的 OkHttp 重发，把 `Set-Cookie` 逐条注入 `CookieManager`
（`Map<String,String>` 保不住多值头），失败返回 null 放行原生栈。
`BypassRuleStore` / `CealingHostParser` 支持 Cealing 格式的「域名集合 → SNI 策略 + 落点 IP」规则表。

---

## 3. Shaft（Java/Kotlin，OkHttp + Cronet）

仓库：`CeuiLiSA/Pixiv-Shaft`（7.7k star）。走的是与 SNI 对抗完全不同的路。

### API 侧：Cronet QUIC/HTTP3

`app/src/main/java/ceui/lisa/http/CronetInterceptor.java` 类注释：

> OkHttp Interceptor 将请求通过 Cronet (QUIC/HTTP3) 发送。GFW 对 pixiv SNI 做了 TCP RST，
> QUIC 走 UDP 可绕过。

- `HostResolverRules` 把 `app-api` / `oauth.secure` / `www.pixiv.net` MAP 到
  `CF_IP_PRIMARY=104.18.42.239` / `CF_IP_SECONDARY=172.64.145.17`
- `addQuicHint` 三个域名，`enableQuic(true) + enableHttp2(true)`
- 精确 host allowlist（`DIRECT_CONNECT_HOSTS`），注释说明这是防止把用户代理/第三方源站
  也误路由进 Cronet 而丢掉 OkHttp 侧策略

### 图片侧：空 SNI + TrustAll + 强制 HTTP/1.1

`Shaft.java` 的 Glide OkHttp 装配：`RubySSLSocketFactory`（`createSocket(socket, null, port, autoClose)`
→ 传 null hostname，Java 不发 SNI）+ `TrustAllCertManager` + `hostnameVerifier { true }`
+ `HttpDns`（硬编码 210.140.139.x）。

强制 HTTP/1.1 的原因写在注释里，与封锁无关但值得记：Glide 高并发缩略图会写坏 OkHttp
的 `Http2Writer` 共享帧缓冲，抛 `ArrayIndexOutOfBoundsException` 崩溃。

### 其它可借鉴点

- `HttpDns.java`：DoH 双端点（Cloudflare + dns.sb）+ 硬编码兜底 + `invalidate()`
  让设置切换立即生效（Glide 持有的 OkHttpClient 引用同一个 Dns 实例）
- `IPv4OnlyDns.kt`：IPv6-only / NAT64 网络上所有 Pixiv 请求会变成 UnknownHost，
  但如果系统 DNS 只返回 IPv6 则保留原结果（安全回退）
- `NetworkTestViewModel.kt`（2023 行）：内置分层自检，含 NAT64 合成地址识别
  （`nat64EmbeddedIpv4Candidates` 按 6 种前缀长度取出嵌入的 IPv4）、fake-ip 识别、
  官方 IPv4 段 CIDR 比对来判定污染
- `ImageHostManager.kt`：图片反代镜像切换（pixiv.cat / pixiv.re / pixiv.nl / 自定义），
  注释注明 pixiv.cat 主域名在大陆被墙、pixiv.re 是大陆推荐镜像；
  非 PIXIV 模式必须退回系统 DNS + 标准 TLS（`requiresStandardClient()`），
  否则硬编码 IP + 无 SNI 会打死第三方反代

---

## 4. PixEz Flutter（Dart + Rust）——与我们同技术栈，参考价值最高

仓库：`Notsfsssf/pixez-flutter`。**已经完全放弃 Dart 的 HttpClient**，
所有出口走 `rhttp`（Rust reqwest + rustls，flutter_rust_bridge 封装），
`pubspec.yaml` 里是 `rhttp: path: ./plugins/rhttp/rhttp`——vendored fork，版本 0.18.0。

### 三个网络模式

`lib/network/pixez_network_settings.dart`：

```dart
static r.ClientSettings? forHost(String host, NetworkMode mode) {
  if (mode == NetworkMode.standard) return null;
  if (mode == NetworkMode.ech) {
    return r.ClientSettings(
      enableEch: true, requireEch: true,
      tlsSettings: r.TlsSettings(
        verifyCertificates: true, rootCertSource: r.RootCertSource.webpki, sni: true),
      dnsSettings: r.DnsSettings.static(overrides: {
        appApiHost: ['104.18.10.118', '104.18.11.118'], /* oauth/accounts 同 */ }),
    );
  }
  return compatible();
}

static r.ClientSettings compatible() => r.ClientSettings(
  tlsSettings: r.TlsSettings(verifyCertificates: false, sni: false),
  dnsSettings: r.DnsSettings.dynamic(resolver: ...),  // Hoster 硬编码 IP
);
```

- **ech**：完整证书校验 + 真实 SNI（被 ECH 加密）+ 固定 CF IP
- **compatible**（老方案）：空 SNI + **关闭证书校验** + 硬编码 210.140.139.155/133

`lib/er/hoster.dart` 的硬编码表：`app-api`/`oauth` → `210.140.139.155`，
`i.pximg`/`s.pximg` → `210.140.139.133`，DoH 用 `1dot1dot1dot1.cloudflare-dns.com`
+ 静态 bootstrap `104.16.248.249` / `104.16.249.249`（与我们 `network_policy.dart`
里已有的 `_defaultDohHostOverrides` 完全一致）。

### ECH 实现细节（rhttp fork 里加的）

`plugins/rhttp/rhttp/rust/src/api/client.rs`：

- `use rustls::client::{EchConfig, EchMode}`，`rustls = "0.23"`
- `const ECH_BOOTSTRAP_HOST: &str = "cloudflare-ech.com"`
- `lookup_ech_config()` 忽略请求的 host，**统一从 `cloudflare-ech.com` 的 HTTPS RR（type 65）
  取 ECH config**，注释：「pixiv's ECH front is served via cloudflare-ech.com」
- 查询走 `lookup_alidns_https_ech()`——用**阿里 DNS**（境内可达）查 HTTPS RR
- 按 host 缓存 `EchClientCacheEntry { client, expires_at }`，TTL 来自 RR
- `require_ech` 为真时拿不到 config 直接失败，不静默回退
- 注入方式：`client.tls_backend_preconfigured(build_ech_tls_config(...))`
- reqwest features 里开了 `http3`

### 上游 rhttp 的能力边界（决定我们 fork 要加什么）

对比 `Tienisto/rhttp` 上游的 `settings.dart`：

| 能力 | 上游 rhttp | PixEz fork |
|---|:---:|:---:|
| `sni: bool`（真实/空 SNI） | ✅ | ✅ |
| `verifyCertificates` / `rootCertSource` | ✅ | ✅ |
| 自定义 DNS（static / dynamic） | ✅ | ✅ |
| min/max TLS version、客户端证书 | ✅ | ✅ |
| **ECH（`enableEch` / `requireEch`）** | ❌ | ✅ |
| 替换 SNI（任意 ServerName） | ❌ | ❌ |

**结论**：我们 fork 上游只需要补 ECH 一项。替换 SNI 两边都没有——reqwest 的 SNI 与 URL host
绑定，要做只能靠「URL 用前置域 + Host 头覆盖 + 强制 HTTP/1.1」这种域前置写法，
或下沉到 hyper + tokio-rustls 自建连接。

### License

`rhttp` 上游 MIT；PixEz 仓库 GPL-3.0；本项目 AGPL-3.0。
GPL-3 并入 AGPL-3 是允许的，但从 MIT 上游 fork 自己实现 ECH 更干净，
不会把 GPL-3 的传染范围引进来，也能持续跟上游同步。

---

## 5. 对本任务的可复用结论

1. **ECH 是首选路径且不牺牲任何安全性**：真实 SNI 被加密，outer SNI 是
   `cloudflare-ech.com`，证书链与主机名校验全部保留。目标域名全在 Cloudflare 上，
   条件天然满足。
2. **ECH 不能是唯一路径**：GFW 对 ECH 握手有封锁报告（未在本任务实测），
   且 `i.pximg.net` 在旧源站上根本没有 ECH。必须保留策略阶梯。
3. **空 SNI 未必要关证书校验**：Shaft 与 PixEz 都关了，但 Pix-EzViewer 证明了
   「换 ClientHello + 保留按 URL host 的校验」是可行的。是否能保留应由探测实测决定，
   而不是先验地放弃。
4. **421 是判定「SNI/证书选错但链路通」的关键信号**，探测层必须能区分它与握手失败。
5. **硬编码 IP 会腐化**：三家都把它当兜底而非主路径，主路径是 DoH。我们
   `DohResolver` 已经就位，只需要补 HTTPS RR（type 65）解析用于取 ECH config。
6. **图片与 API 必须分策略**，且第三方反代镜像只能是用户显式选择的兜底，不能是默认。
7. **WebView 是传输层管不到的洞**：Rust client 帮不了系统 WebView，
   Android 侧要靠 `shouldInterceptRequest` 重发 + `CookieManager` 注入。
