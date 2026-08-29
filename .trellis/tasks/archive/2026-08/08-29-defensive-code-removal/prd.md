# 移除过度防御性代码

## 问题

代码库里存在一类模式：为假想的威胁模型建设基础设施，然后让基础设施反过来否决真实需求。
症状是登录链路被 `已拒绝非 Pixiv 登录导航` 完全堵死。

## 删除项

### A 类 —— 正在造成损害

| 位置 | 删除内容 | 为什么它是错的 |
| --- | --- | --- |
| `login_webview_page.dart` | 生命周期杀会话、主机白名单、`WebViewRouteSession` 租约、不可恢复的 `_fail` | 读验证码 / 第三方 IdP / 全屏输入法都会离开前台；`oauth.secure.pixiv.net` 是 Pixiv 自己的主机却不在白名单里 |
| `pkce.dart` | 回调形状检查、参数白名单、强制 `state` | Pixiv 真实回调是 `pixiv://account/login?code=...&via=...`，三处都被拒 |
| `next_page_parser.dart` | 端点路径白名单 + 每端点参数白名单 | `next_url` 来自 Pixiv 自己的 TLS 响应；Pixiv 加一个参数就整条分页挂掉（`viewed[n]` 补丁就是上一次挂掉的疤） |
| `widget_feed_loader.dart` | 401 分支里的凭据版本比较 | 处理 401 本身就会推进凭据版本，比较它会把每次真实鉴权失败判成 superseded |
| `account_transfer.dart` | nonce + 重放存储 + TTL + 时钟偏移 + 严格解析器 | 信封本身就是明文凭据，过期不提供任何保护；重放一次导入是幂等的 |

### A 类 —— 恒假的闸门

`networkRevision` 作为第三重身份闸门（`FeedCommitGate` / `MutationBoundary` / 下载 / ugoira / widget / 资料编辑）。
`account_store.dart` 里 4 个调用点，`_resetNetworkSession()` 永远紧跟 `_nextCredentialRevision()`，
所以 `networkChanged` 永远被 `credentialChanged` 先命中，无法独立触发。

### B 类 —— 无消费者的死基础设施

- `webview_route.dart`（491 行：`WebViewRoutePolicy` / `Session` / `Lease` / `LoopbackAdapter`）
- `EchCapabilityGate` + `EchCapabilityEvidence`
- `DohResolver`（157 行，从未接入生产）+ `DnsSource.doh`
- `NetworkHealthSnapshot` / `healthEntries` / `recordHealth`（只写不读）
- `PixivDestinationRegistry.allowedHosts`（唯一消费者是已删的 WebViewRouteSession）
- `NetworkRouteKind.ech` / `.webViewLoopback`（从未构造）；`NativeStrictConnector` 里对应的不可达分支
- WebKit 能力探测全链路：Dart channel + `WebKitCapabilities` 接口 + `WebKitCapabilityChannel.kt` + `androidx.webkit` 依赖
- `sameMutationBoundary(includeNetwork:)`（无调用方传 false）

## 保留的真实边界

- PKCE 会话与授权码的绑定
- `TransferCredentialVerifier`：剪贴板导入的凭据必须通过 Pixiv 服务端校验
- 剪贴板敏感标记、指纹条件清除、5 分钟自动清除
- 下载逐跳同源校验（`i.pximg.net` / `s.pximg.net`），更新器 APK 的 `strictUrlPolicy`
- `next_url` 必须是 `app-api.pixiv.net` 上的 https，无 userinfo、非默认端口
- 各 repository 对游标端点与身份参数的钉定
- 账号 / 凭据版本闸门；`TokenRefreshGate` 单飞；`NovelReaderCommitGate` 代际闸门
- `FeedCommitGate` 游标重复检测（防无限分页）

## 验证

- `flutter analyze`：clean
- `flutter test`：全量通过，除 2 个已知 WSL loopback 环境不稳定用例
  （`oauth_service_test` token exchange、`download_manager_test` real sockets；单独跑通过）
- MuMu API 35 真机：见 `research/screenshots/`
