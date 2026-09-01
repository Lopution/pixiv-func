# 归档证据失效清单

2026-08-29 的过度防御代码清理（`08-29-defensive-code-removal`）删除了若干符号，使部分已归档
叶子的证据与当前代码不再一致。按 `writeback_policy: archives_read_only`，归档件**不回写**——
它们记录的是当时的事实。本文件是失效索引。

| 归档件 | 描述的内容 | 失效原因 |
|---|---|---|
| `08-27-android-platform-boundary-hardening`（`design.md`、`research/implementation-evidence.md`） | WebKit 能力探测（Dart channel + `WebKitCapabilities` + `WebKitCapabilityChannel.kt` + `androidx.webkit` 依赖）；`WebViewRouteSession` 租约 | 全链路已删除。能力探完之后没有任何消费者，不影响任何决策 |
| `08-26-restricted-compat-network`（`prd.md`、`design.md`、`implement.md`、`research/implementation-evidence.md`） | `DohResolver`（157 行）、`NetworkHealthSnapshot` / `healthEntries` / `recordHealth`、`PixivDestinationRegistry.allowedHosts`、`NetworkRouteKind.ech` / `.webViewLoopback`、`EchCapabilityGate` | 均已删除：DoH 从未接入生产、health 只写不读、`allowedHosts` 的唯一消费者是已删的 `WebViewRouteSession`、两个枚举值从未被构造。**注意：`DohResolver` 将在 `08-29-replica-v1-completion` 中重新实现并真正接线，届时的实现与归档件描述的那份无关** |
| `08-27-feed-generation-commit-hardening`（`prd.md`、`design.md`、`research/implementation-evidence.md`） | `FeedRequestContext.networkRevision`、`FeedDiscardReason.networkChanged`、`FeedCommitGate.commit` 的 `networkRevision` 参数 | `networkRevision` 作为第三重身份闸门恒不可达：`advanceNetworkRevision` 的唯一调用点位于 `account_store.dart`，其 4 个到达路径都先推进了 `credentialRevision`，因此 `networkChanged` 永远被 `credentialChanged` 先命中 |
| `08-26-android-platform-parity`（`design.md`） | `WebViewRouteSession` 相关的平台边界描述 | 同上，类已删除 |
| `08-26-secure-clipboard-account-migration`（`prd.md` R2/R3/R6、`design.md`） | `createdAt` / `expiresAt` / nonce / `ReplayStore`、严格 base64 与精确键集校验、错误文案 `expired` / `replayed-on-this-device` | 信封携带的是明文凭据，TTL 不提供任何保护；重放一次导入是幂等的。`TransferEnvelope` 版本升到 2，收敛为透明信封 + SHA-256 损坏检测。真实边界（`TransferCredentialVerifier` 服务端校验、sensitive clipboard 标记、5 分钟自动清除）保留 |
| `08-27-mutation-ownership-hardening`（`design.md`、`prd.md`、`research/implementation-evidence.md`） | `MutationEnvelope` 的 `policy/network revision` 字段、`MutationDiscardReason.networkChanged`、`sameMutationBoundary(includeNetwork:)` | 同 feed 闸门恒不可达；`includeNetwork` 无调用方传 `false` |
| `08-26-profile-edit`（`research/implementation-evidence.md`、`technical-research.md`） | `ProfileEditOwner` 的 network revision 比较 | 同上 |

## 一个由此暴露的真实缺陷

`widget_feed_loader` 的 `ApiUnauthorized` 分支原先比较凭据版本判断是否被取代。但**处理 401
本身就会触发 `markReauthRequired`，而它会推进凭据版本**——因此生产环境中每次真实鉴权失败都被
判成 `superseded`，快照不清除，主屏留着过期作品。

测试里 `networkRevision` 默认是常量 stub，使该分支恒为真，因此从未暴露。已改为只比对账号 id，
并在代码与 spec 中写明为什么不能比版本。

`08-26-android-home-widgets` 尚未归档，其残余工作已并入本任务，证据将在本任务内一并更新。

## 影响范围

以上失效仅涉及**描述**，不涉及归档叶子当时交付的功能。当前代码的权威契约见
`.trellis/spec/frontend/state-management.md`（已于 2026-08-29 同步更新）。
