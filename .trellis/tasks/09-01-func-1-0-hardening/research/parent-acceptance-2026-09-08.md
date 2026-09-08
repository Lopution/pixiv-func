# Parent 验收证据（2026-09-08）

基线：worktree `/root/Pixiv-func-P`，分支 `task/09-01-func-1-0-hardening`，HEAD `1186ff2`（`Merge pull request #9`）。
方法：读 parent PRD / 七个已归档 child / spec；`git log`；对 R6 与 N-FIXED 做 `rg`；只跑网络相关套件 + `flutter analyze`。未改代码，未勾 `prd.md`。

---

## AC1 — 7 个 child 已实现、check、归档

`python3 ./.trellis/scripts/task.py list`：

```
09-01-func-1-0-hardening/ (planning) [7/7 done] [Lopution]
```

`task.py list-archive 2026-09` 七条均在：`ux-correctness`、`settings-productization`、`reverse-image-saucenao`、`comment-translation`、`behavior-correctness-cleanup`、`network-perf-ab`、`release-blockers`。

| Child | `task.json` | PR / 归档记录 | implement 检查门 |
|---|---|---|---|
| `ux-correctness` | `status=completed`，`completedAt=2026-09-02`，`branch=null`，`pr_url=null`，`commit=null` | 分支工作流之前直接进 `main`。提交：`e6a8fd7`（U9/C11/C20）、`f6c9da9`（U2）、`af44831`（D7/U8）、`62e5a12` / `409df51` / `10ea29d`（U1）。无 PR 号 | 代码门全勾。未勾：人眼过渡/手感（`prd.md` 全阶段、`implement.md` 阶段 4 交付清单） |
| `settings-productization` | `completedAt=2026-09-08`，`branch=task/09-01-settings-productization`，`pr_url=null` | [PR #3](https://github.com/Lopution/pixiv-func/pull/3) merge `206644e` 2026-09-08 00:40 +0800 | 代码门全勾。未勾：阶段 5 真机耗时；最终真机（过滤/质量/三种下载目标/命名/多页/语言/网络模式重启） |
| `comment-translation` | 同上形态，`pr_url=null` | [PR #5](https://github.com/Lopution/pixiv-func/pull/5) merge `28895bc` | 代码门全勾。未勾：大陆百度 / LLM / Google / 关闭零请求 |
| `release-blockers` | 同上 | [PR #6](https://github.com/Lopution/pixiv-func/pull/6) merge `8c91c37` | 代码门全勾。未勾：API 29 + 高版本真实自更新（需 keystore + Secrets）；F-Droid/store-managed 真机 |
| `network-perf-ab` | 同上 | [PR #7](https://github.com/Lopution/pixiv-func/pull/7) merge `358fbbb` | 代码门与「用户真机观察 Automatic 首屏」均已勾（`research/real-device-probe-2026-09-03.md`）。parent 级有/无代理矩阵仍留给用户 |
| `reverse-image-saucenao` | 同上 | [PR #8](https://github.com/Lopution/pixiv-func/pull/8) merge `54d2ea8` | 代码门全勾。未勾：截图→详情、非 Pixiv 外跳、限流/挑战/隐私提示 |
| `behavior-correctness-cleanup` | 同上 | [PR #9](https://github.com/Lopution/pixiv-func/pull/9) merge `1186ff2` | 代码门全勾。未勾：API 29/高版本、有/无 widget、kill/restart recovery、mutation refresh、深链、多 tab 滚动 |

七份 `task.json` 都没有写入 `pr_url`；PR 号来自 `git log --merges` / `gh pr list`，不是 archive 记录本身。

**判定**：满足（未勾项全部是真机/密钥，见文末清单）。

---

## AC2 — R5 跨 child 顺序（D5 先于 C4/C5/C22）

R5 要求：`settings-productization` 的 D5（`DownloadDestination` + 迁移）必须先于 cleanup 的 C4/C5/C22（owner = `accountId + DownloadDestination.identity`，orphan 只按精确 owner 清）。

### 真实提交顺序（不能编）

| 时间 | 提交 | 内容 |
|---|---|---|
| 2026-08-28 | `fdef098` `fix: harden media job recovery and owned outputs` | recovery owner **已存在**，字段是 `accountId` + **`credentialRevision`** + 字符串 `destination`（当时常量 `kDownloadDestination = 'Pictures/PixivFunc'`）。这是分支工作流之前的 main 提交，也是 C4 要修的形态 |
| 2026-09-07 18:05 +0800 | `6d720f2` `wip(09-01): in-progress code for the Func 1.0 hardening children` | **同一提交**新增 `lib/core/download/download_destination.dart`（`class DownloadDestination`，`identity`），并把 `_identityKey` / `_sameContext` 改成 `accountId\|destination.identity`，去掉 download 侧 `credentialRevision`。commit message 写明 spans several 09-01 children，settings 与 cleanup 都未单独 check |
| 2026-09-07 18:05 之后 | `git log 6d720f2..HEAD -- download_destination.dart download_recovery.dart download_manager.dart` | **空**。PR #3 / #9 没有再改这三份文件 |

因此：不存在「先 merge settings PR、再 merge cleanup C4」的代码顺序。D5 与最终 owner 算法是 `6d720f2` 一次落地。更早的 owner 代码（`fdef098`）在 value object 之前，且用的是 `credentialRevision`。

PR 归档顺序（文档/测试，不是 D5/C4 代码）：#3 settings（2026-09-08 00:40）→ #9 cleanup（2026-09-08 13:24）。cleanup `implement.md` 把「D5 已交付、PR #3」写成开工 gate，指的是归档状态，不是 git 上 D5 首次出现的时间。

### 最终状态（HEAD `1186ff2`）是否消费 `DownloadDestination.identity`

是。

| 位置 | 证据 |
|---|---|
| `lib/core/download/download_destination.dart:38-43` | `identity`：`album:pixivfunc` / `album:<name>` / `saf:<uri>` |
| `lib/core/download/download_recovery.dart:49-52` | 注释：owner = 稳定 accountId + destination，持久化只存 `identity` |
| `lib/core/download/download_recovery.dart:195` | `'destination': snapshot.destination.identity` |
| `lib/core/download/download_manager.dart:487-493` | `_identityKey` → `'${accountId}|${destination.identity}'` |
| `lib/core/download/download_manager.dart:745-752` | `_sameContext` 只比 `accountId` + `destination.identity` |
| `lib/core/download/download_manager.dart:322,431-440` | C22：pending 按 `record.owner.ownerId` 精确匹配；无匹配行不删 |
| `lib/core/download/download_providers.dart:65-70` | 产品提交：`accountId` + `downloadDestinationProvider` |
| `lib/app/app.dart:36` | C5：`initState` `ref.read(downloadManagerProvider)` |
| `.trellis/spec/frontend/state-management.md:898-900` | 合同：`accountId + DownloadDestination.identity` |

**判定**：最终状态满足 R5 的产品约束。字面「实际提交顺序 settings 先于 cleanup」无法从 git log 证明为两个先后提交——主会话自行决定是否勾 parent 第二框。

---

## AC3 — R6 禁止清单

范围：`lib/`、`android/`、`plugins/rhttp/rhttp/lib`。澄清：禁止的是注册表/插件发现，不是普通 map / host allowlist / `switch` 分发。

| 条目 | 模式 | 命中 |
|---|---|---|
| `GlobalWorldRevision` | `rg -e GlobalWorldRevision` | 0 |
| `IdentityEpochManager` | `rg -e IdentityEpochManager` | 0 |
| `OwnershipPolicyEngine` | `rg -e OwnershipPolicyEngine` | 0 |
| `EntityAuthorityGraph` / `AuthorityGraph` | `rg -e EntityAuthorityGraph` / `AuthorityGraph` | 0 |
| `MutationTransactionManager` | `rg -e MutationTransactionManager` | 0 |
| `RefreshGestureEpoch` | `rg -e RefreshGestureEpoch` | 0 |
| `BallisticLease` | `rg -e BallisticLease` | 0 |
| `NetworkFallbackStateMachine` | `rg -e NetworkFallbackStateMachine` | 0 |
| `AccountTransactionJournal` | `rg -e AccountTransactionJournal` | 0 |
| 反向搜图 provider 注册表 | `ProviderRegistry` / `PluginRegistry` / `pluginDiscovery` / `plugin.?discover` | 0 |
| 翻译 provider 插件发现 | `TranslationProviderFramework` + 上列 registry | 0。生产是 `ConfiguredCommentTranslationService` 的 `switch`（`lib/core/comments/comment_translation.dart:581-591`），R6 澄清允许 |
| 自制 Android 文件浏览器 | `FileBrowser` | 0。下载目录走系统 SAF（`SafTreeChannel.kt`） |
| JS 文件命名脚本 | `javascript` / `eval(` / `Function(` @ `naming_rule.dart` | 0。模板只有 `{artist,title,id,page,ext,date}` |
| Cronet | `rg -e Cronet` | 0 |
| QUIC | `rg -e 'QUIC\|\bquic\b'` | 0 |
| VPN / `VpnService` | `rg -e VpnService`；`\bVPN\b\|\bvpn\b` | 0 |
| `_runLegacyLadder`（C14，对照） | `rg -e _runLegacyLadder` 于上述三目录 | 0 |
| 全局 version/epoch 层 | `WorldRevision` / `EpochManager` / `IdentityEpoch` @ `lib/` | 0 |

未当作违规：

- `PixivDestinationRegistry`：精确 Pixiv host allowlist，不是插件发现。
- `plugins/rhttp/rhttp/lib` 里的 `HttpVersionPref.http3` 枚举：上游类型；生产 `RhttpClientFactory.settingsFor` 固定 `HttpVersionPref.all`，注释写明 HTTP/3 未开（`rhttp_client_factory.dart:101-106`）。
- `e6ec619` 删除无消费者的 `imageSourceProvider`；隐藏字段 `imageSource` 按 settings R7 保留，不是新死设置。

**判定**：满足。未见回归或新禁令项。

---

## AC4 — N-FIXED-1 ~ N-FIXED-9 在 HEAD 仍成立

原文定义：`research/audit-original-2026-09-01.md:672-706`。child 侧对照：归档 `09-01-network-perf-ab/research/current-head-recheck.md`（2026-09-03，仍写 preflight；已被 09-03 用户决策 + 09-08 child check 修订）与该 child `prd.md:136`。

`secure_resolver_test.dart` 不存在。实际网络套件：`ls test | rg -i 'network|resolver|http_client|route|rhttp|tls_sni'`。

| 编号 | 原文「现」 | HEAD `1186ff2` 代码 | 钉死测试 |
|---|---|---|---|
| N-FIXED-1 | 先 side-effect-free probe，再只发一次业务请求 | **形态已改**（用户 2026-09-03）：`runLadder` 无 `probe`，业务请求即 attempt（`network_policy.dart:296-327`）。POST/换 token 只在未送达失败（DNS/connect/reset/TLS 握手）换档；已送达（含 timeout-after-send）不重发。旧「卡在 direct、POST 不可 replay 到不了 ECH」没有回来 | `restricted_compat_network_test.dart:426` timed-out POST never replayed；`:1041` OAuth POST 严格档失败后才到 fast；`:1082` failed fast POST 不重复 token exchange |
| N-FIXED-2 | route selection 与业务 mutation 分开 | 生产 mutation 走同一 attempt-first ladder，不是用收藏/关注 POST 当探测。探测页仍是 `runExplicit`（`network_policy.dart:126`） | `:397` POST 先严格档；`:460` mutation 优先 ECH |
| N-FIXED-3 | rhttp 结构化异常分类 | `network_contracts.dart:446-470`：`RhttpInvalidCertificate` / `Timeout` / `Cancel` / `Connection` / status | `:294` rhttp structured exceptions map without textual security guesses |
| N-FIXED-4 | 普通 handshake ≠ certificate mismatch | `network_contracts.dart:375-380,401-404`：`HandshakeException` → `tlsHandshake`；只有 `RhttpInvalidCertificateException` → `certificateMismatch`（terminal） | `:294`；`:673` HTTP/auth/certificate/cancel 不 fallback |
| N-FIXED-5 | 成功 ECH memory 刷新/保留，不被自己删掉 | `network_policy.dart:133` `_routeMemory`；`:604-619` 成功写入并在同 ECH key 时保留 `createdAt` | `:498` remembered ECH reused within TTL；`:868` per-host memory skips doomed direct |
| N-FIXED-6 | 同一轮不重复同一 kind | `network_policy.dart:343-344,439-454,494` `attemptedKinds` / `attemptedKeys` | `:632` failed remembered route invalidated without repeating its tier |
| N-FIXED-7 | download client cache key 含 canonical host | `policy_download_transport.dart:63-64` `canonicalHost\|route.key\|purpose` | `:1001` download transport cache identity includes the canonical host |
| N-FIXED-8 | OAuth 走 network policy | `account_store.dart:323-328` `oauthServiceProvider` ← `pixivNetworkFactoryProvider.client(oauth)` | `:854` production OAuth provider is backed by the policy factory |
| N-FIXED-9 | `HttpVersionPref.all`，ALPN，不强制 h2 prior knowledge | `rhttp_client_factory.dart:101-106` | `rhttp_client_factory_test.dart:22-26` |

本机（2026-09-08）`export PATH=/opt/flutter-3.47.2/bin:$PATH`：

```
flutter test \
  test/restricted_compat_network_test.dart \
  test/network_probe_test.dart \
  test/network_fast_route_test.dart \
  test/pixiv_http_client_test.dart \
  test/doh_resolver_test.dart \
  test/rhttp_client_factory_test.dart \
  test/tls_sni_behaviour_test.dart
```

结果：`+112: All tests passed!`

| 文件 | 项数 |
|---|---|
| `restricted_compat_network_test.dart` | 27 |
| `network_probe_test.dart` | 18 |
| `network_fast_route_test.dart` | 4 |
| `pixiv_http_client_test.dart` | 23 |
| `doh_resolver_test.dart` | 23 |
| `rhttp_client_factory_test.dart` | 14 |
| `tls_sni_behaviour_test.dart` | 3 |
| 合计 | 112 |

`flutter analyze`：`No issues found!`（5.3s）。未跑全量 `flutter test`（children 刚报 674 passed）。

**判定**：满足。N-FIXED-1 的「先探测再单发」被 attempt-first 取代是产品决策，不是回退到「POST 不可 replay 就到不了 ECH」。

---

## AC5 — `.trellis/spec/` 是否沉淀本轮约定

| 约定 | 位置 | 状态 |
|---|---|---|
| **设置项必须有真实消费者** | `.trellis/spec/` 内 **没有** 这条中文/英文禁令。最接近：`frontend/state-management.md:473-475`（罗列已有 typed providers，不是「没有消费者就不留开关」）。settings child 自己的 PRD 有并已勾（归档 `09-01-settings-productization/prd.md:147`），但 parent AC 要的是 spec。本文件不代写 | **缺** |
| **删除 guard 时同步删除固化错误行为的测试** | `frontend/quality-guidelines.md:109-114`（标题即该中文句）；`backend/quality-guidelines.md:65-68`（指向前者，PR #9） | 有 |
| Secrets 不进任何序列化面 | `backend/quality-guidelines.md:35-59`（`09-01-comment-translation`） | 有 |
| Release artifacts / 签名合同 | `backend/release-artifacts.md:10-25`（产物）；`:27-49`（`09-01-release-blockers` Signing） | 有 |
| Merge sources（C3） | `frontend/state-management.md:43-55` `EntityMergeSource.feed\|detail` | 有 |
| Route ladder（P-NET / C14） | `frontend/state-management.md:484-529` 档序；`:548-555` attempt-first / 未送达才换档；`:976-978` `_runLegacyLadder` 已删 | 有 |
| 框架旁禁止第二套手势状态机（U1） | `frontend/quality-guidelines.md:48-57` | 有（额外，ux） |
| Download owner = `accountId + DownloadDestination.identity` | `frontend/state-management.md:898-900` | 有（额外，cleanup） |

**判定**：未完全满足。guard 删除规则与其它 child 约定已在 spec；「设置项必须有真实消费者」未进 `.trellis/spec/`。主会话决定是否补写后再勾第五框。

---

## 结论

| Parent 级条目（`prd.md:148-153`） | HEAD `1186ff2` | 原因 |
|---|---|---|
| 7 个 child 完成实现、check、归档 | **满足** | `list` 为 `[7/7 done]`；七份 archive `task.json` `completedAt` 已填；六份有 merge PR #3/#5/#6/#7/#8/#9；ux 无 PR，见 AC1 |
| R5 提交顺序 D5 先于 C4/C5/C22 | **最终状态满足；字面顺序无法证明** | D5 与 identity owner 同提交 `6d720f2`；更早 owner 用 `credentialRevision`（`fdef098`）。最终代码消费 `DownloadDestination.identity` |
| R6 禁止清单未被违反 | **满足** | 点名类型与 Cronet/QUIC/VPN/注册表/JS 命名/自制文件浏览器均为 0 |
| 9 项网络历史修复未被回退 | **满足** | 九项均可定位；网络套件 112/112；N-FIXED-1 为 attempt-first 新形态 |
| spec 沉淀（至少：设置消费者 + 删 guard 测） | **未完全满足** | 删 guard 测已在两边 quality-guidelines；设置消费者规则不在 spec |

未发现代码回归或 R6 违规。`prd.md` 的勾选（含「最终真机验收」）留给主会话；本文件不改 `prd.md`。

未决产品项（非真机、非本 check 范围）：settings R3 预览/详情/查看三组 vs 两组；comment-translation 是否加腾讯 TMT。

---

## 留给用户的真机项

只汇总七个 09-01 child 仍未勾的人眼/真机项，以及 parent `最终真机验收`（必须保持未勾）。

### Parent（`prd.md` 最终真机验收，用户亲自跑）

- API 29 与当前主流高版本各一轮。
- 大陆移动 / Wi-Fi，无代理与有代理（Automatic 不得破坏用户已有网络）。
- 首次普通 OAuth 登录可完成。
- 已登录后：Recommended / Ranking / Search / User / Detail / token refresh / bookmark / follow / comment / image / download / Ugoira / history / widget（有实例与无实例）。
- kill process → restart → download recovery 可见可恢复。
- GitHub updater 在 API 29 与高版本各一次真实安装（需正式签名 draft release + Secrets）。

### ux-correctness

- Profile header：展开 / 中间 / 折叠三态，含无头像用户。
- U1 五手势：慢拉；反向回滑且回滑期间内容不上滚；快速甩动；松手惯性不重唤指示器；连续第二次刷新不残留。
- 观感：下拉时列表内容跟手下移（iOS 式），不再是「内容不动、指示器悬浮」。

### settings-productization

- 截图到 `research/screenshots/`。
- `networkMode` 重启保持。
- 内置相册 / 自定义相册 / SAF 三种目标真实落盘。
- 多 P 命名不覆盖；预览文件名 = 落盘名。
- UI 日语后抓包 `Accept-Language`。
- 高屏蔽率下列表仍能滚、续拉有上限。
- 阶段 5 耗时口径：API 首屏、OAuth 换 token、图片首屏、下载、widget 刷新。

### reverse-image-saucenao

- Pixiv 作品截图 → 命中并进站内详情。
- 非 Pixiv 图 → 系统浏览器打开。
- 限流等待文案、挑战页失败、隐私提示。

### comment-translation

- 大陆网络百度真实翻译（高级版需实名）。
- LLM 自定义 HTTPS endpoint。
- 旧 Google 配置回归。
- 关闭时无翻译网络请求。

### behavior-correctness-cleanup

- API 29 + 高版本。
- 桌面 widget：有实例 / 无实例。
- kill / restart 后 recovery 可见，不自动重发。
- 收藏 / 关注 / 评论在 token 刚过期时 refresh 一次后成功。
- 用户深链进用户页；多 tab 滚动位置保留。

### network-perf-ab

- child 已有无 VPN 大陆 5G/4G 两轮探测并勾了 Automatic 首屏观察。
- 仍缺：有代理环境是否破坏用户网络（并入 parent 矩阵）。

### release-blockers

- 配置 APK keystore + manifest 签名 Secrets，dispatch `release.yml`，核对正式证书指纹。
- API 29 与高版本各一次：下载 → 验签 → 安装；负向篡改由用户在同一轮看。
- F-Droid / store-managed 构建行为确认。
- API 29 设备 provider 上跑一遍 `SHA256withECDSA`（离线 fixture 已有）。

### 相关但非本 parent 七 child（用户点名）

- `09-07-release-size-per-abi` B6：`opt-level = "s"` 已测未采用；墙钟对照在 `build/b6-compare/`（登录 + 列表 + 大图下载）。不是 09-01 child 的未勾项。
