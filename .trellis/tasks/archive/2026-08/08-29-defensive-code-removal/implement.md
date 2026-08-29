# 实施记录

## 变更规模

`git diff HEAD --stat -- lib test android`：**55 个文件，+473 / −2467**

删除的文件：

- `lib/core/network/compat/webview_route.dart`
- `lib/core/platform/webkit_capabilities_channel.dart`
- `android/app/src/main/kotlin/io/github/lopution/pixivfunc/WebKitCapabilityChannel.kt`

## 逐项

### 1. 登录链路（用户报告的故障）

四道彼此独立、每一道都足以堵死登录的闸门，按在真机上暴露的顺序：

1. `didChangeAppLifecycleState`：任何非 `resumed` 状态销毁 PKCE 会话 →
   现在只有 `detached` 才终止。
2. `_fail` 不可恢复但 UI 声称可恢复 → 拆成 `_abortLogin`（终止，按钮「重新打开」）
   与 `_reportRecoverable`（保留会话，按钮「知道了」）。
3. 主机白名单拒绝 `oauth.secure.pixiv.net`（Pixiv 自己的 oauth 主机，只是归在另一个
   `purpose` 下）→ 整套导航拒绝删除。能在这个 WebView 里加载出来的，本来就是 Pixiv
   自己选的登录端点。
4. `parsePixivAccountCallback` 以三种方式拒绝真实回调
   `pixiv://account/login?code=...&via=...`（路径非空、未知参数 `via`、强制 `state`）
   → 只保留「code 缺失 / 重复 / 为空」这类真正让授权码有歧义的判断。
   `state` 存在时比对，不存在不强求；PKCE verifier 仍是会话绑定。

### 2. networkRevision 第三重闸门

`NetworkRevision` 在 `NetworkAccessPolicy` 内部仍然有用（socket/client 池的键、
换账号时作废旧池），保留。删除的是它作为身份闸门穿过：

`FeedRequestContext` → `FeedCommitGate`（`beginRequest`/`commit`/`isActive`/`discard`/`_reason`）
→ `PagedFeedController`；`MutationBoundary` / `MutationEnvelope` / `MutationLedger`
→ bookmark / comment / follow store；`DownloadSubmissionContext` / `Snapshot`（含磁盘 JSON）
→ `DownloadManager._identityKey` / `_sameContext`；`UgoiraExport._checkOwner`；
`ProfileEditOwner.matches`；`WidgetFeedLoader`。

`FeedDiscardReason.networkChanged` 与 `MutationDiscardReason.networkChanged` 一并删除，
三个 store 里的三元兜底分支收敛为 `credentialChanged`。

`download_recovery.dart` 的磁盘格式不再写 `networkRevision` / `networkIdentity`；
`fromJson` 不再读它们 —— 旧记录里多余的键被忽略，无需迁移。

### 3. 剪贴板账号转移

`TransferEnvelope` 从 600 行的重放防护协议收敛为透明信封：
`version` / `payloadType` / `payload` / `checksum`，版本号升到 2。

删除：nonce、`TransferReplayStore` / `InMemoryTransferReplayStore` /
`SecureTransferReplayStore`、`createdAt` / `expiresAt` / `maxLifetime` /
`maxClockSkew`、严格 base64 正则与往返比对、精确键集校验、控制字符扫描、
`accountId` 正则、凭据键集相等判断，以及
`AccountTransferErrorCode.expired` / `.replayedOnThisDevice` 和它们的 4 语言文案。

保留：SHA-256 作为损坏检测（半截粘贴 / 手改），以及真正的边界 ——
`TransferCredentialVerifier` 拿凭据去 Pixiv 服务端换权威身份。
剪贴板 5 分钟自动清除移到 `transferClipboardLifetime` 常量。

### 4. next_url 参数白名单

`NextPageParser.parse` 保留 https + `app-api.pixiv.net` + 无 userinfo + 默认端口，
删除端点路径白名单、每端点参数白名单与 `kNextPageParamPatterns`。
`kNextPageEndpoints` 保留给 `firstPage()` —— 那是描述本客户端自己发出的请求。

### 5. widget 401 分支

原代码用「网络版本未变 且 账号 id 相同」判断是否被取代。测试里 `networkRevision`
默认是常量 stub，所以这条永远为真；生产里 `markReauthRequired` 会推进网络版本，
于是每次真实 401 都会被判成 `superseded`，快照不清除、主屏留着过期作品。
改为只比对账号 id，并写清为什么不能比版本。

## 验证

```
flutter analyze                      # No issues found
flutter test                         # 全量通过，除下述 2 个
```

已知环境不稳定（改动前就存在，与本次变更无关）：

- `test/oauth_service_test.dart` token exchange
- `test/download_manager_test.dart` HttpDownloadTransport over real sockets

两者都是 WSL 下 dart:io loopback 在并发压力下丢连接；测试文件自己就标了
`(environment-flaky)` 并带 `tolerant()` 重试。单独运行通过：

```
flutter test test/oauth_service_test.dart \
  --plain-name "verifier is cleared after a failed exchange"   # All tests passed
```

## 真机验证（MuMu，Android API 35）

`flutter build apk --debug --flavor github` → `adb install` → `pm clear` 冷启动。

| 截图 | 验证点 |
| --- | --- |
| 01–04 | 引导流程到登录入口 |
| 05 | Pixiv 登录页完整加载，无「已拒绝非 Pixiv 登录导航」 |
| 06–07 | 退到后台再回来，PKCE 会话仍可用，无「页面已暂停，请重新打开」——**这是改动前必然失败的场景** |
| 08 | 点第三方 IdP 正常跳转到 `accounts.google.com` —— 旧主机白名单会拒绝这一跳 |
| 09 | 返回后 app 健康，无 FATAL |
| 10 | 重开登录页，新的 PKCE 会话正常 |

未验证：需要真实账号的路径（token 交换落地、剪贴板账号导出/导入、下载、widget 刷新）。
