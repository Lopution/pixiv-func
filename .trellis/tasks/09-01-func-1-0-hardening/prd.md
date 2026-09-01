# Func 1.0 产品化收口：真实体验、产品化设置与发布阻塞

## Background

Replica v1 的实现树（`08-26-pixiv-func-replica-v1`，19/19）已于 2026-09-01 全部归档。
本 parent 的源需求来自 2026-09-01 的一次全量审计，证据合并自三类来源：

1. 用户在真实 Android 设备上的直接使用反馈；
2. 对 HEAD `9d1cb1b` 的代码审计；
3. PixEz / Pixiv-Shaft / SauceNAO / 国内翻译服务 / Android / GitHub 官方资料的对标调研。

审计原文由用户持有。本任务对其中每条代码断言的复核结论见
`research/audit-verification.md` —— 复核在 HEAD `9d1cb1b` 上完成，**断言全部成立**，
另发现 4 处审计本身的缺口，已在同文件记录并分配给对应 child。

本轮的定位是**给已经存在的核心能力真正接线**，不是建设更完整的框架。

## Requirements

### R1. 工程原则（约束所有 child）

- Modern by default, compatible by design, graceful by degradation.
- Minimum Necessary Defense。
- 只修有真实产品后果、真实协议要求，或已复现证据的问题。
- 不因「理论上可能」增加状态机、lease、epoch、journal、allowlist 层等基础设施。
- 错误 guard 阻塞正常行为时删除或放松 guard，而不是叠加下一层补丁。
- 测试必须覆盖正常用户 happy path。「测试全绿」不能证明一个虚构的限制就是正确产品行为。

### R2. 已确定的产品边界（不在本轮重新讨论）

- **Android 基线**：API 29 / Android 10，核心功能在 API 29 必须完整可用。
- **Live**：已移出 Replica v1，不作为 1.0 blocker。
- **中国大陆网络**：首次注册与首次 OAuth 登录的网络环境由用户负责；登录后（API /
  refresh / 图片 / 下载 / 收藏 / 关注 / 评论）是 Func 的责任。不做 VpnService、系统代理、
  Private DNS 修改、hosts、root 依赖、自建远端 relay/proxy。Cronet / QUIC / HTTP/3
  在没有实测证据前不扩 scope。
- **TLS**：API / OAuth / 账号接口必须保持真实服务器身份验证，不做 MITM、不装自定义 CA。
  放松传输策略若被 A/B 证明有价值，只允许限制在公开图片 CDN（`i.pximg.net` /
  `s.pximg.net`）内部使用，且不得暴露为全局普通用户开关。
- **非 parity blocker**：DM、发布作品、pixivision 等原项目本来就未完成的能力，不为
  「完整度」临时加入 1.0。

### R3. 已定的产品决策（D1–D7）

| 编号 | 决策 | 归属 child |
|---|---|---|
| D1 | 反向搜图走零配置 SauceNAO，结果不限制 Pixiv 来源 | reverse-image-saucenao |
| D2 | 翻译首选百度（大陆可用）+ 通用 OpenAI-compatible LLM，Google 降为兼容项 | comment-translation |
| D3 | 网络设置简化为「网络模式 / 网络诊断 / 高级设置」三项 | settings-productization |
| D4 | 预览质量默认中图、查看质量默认原图，两组三档枚举 | settings-productization |
| D5 | 下载默认存入 PixivFunc 相册，可切自定义相册或 SAF 目录 | settings-productization |
| D6 | 文件命名走预设优先 + 受限模板 + 实时预览，不引入脚本/正则/条件语法 | settings-productization |
| D7 | 个人主页 header：头像连续插值缩小并入 toolbar，展开态直径 96–112dp | ux-correctness |

### R4. 任务地图

每个 child 独立可规划、可实现、可验收、可归档。依赖顺序写在 child 的 `prd.md` /
`implement.md` 里，不靠树的位置表达。

| Child | 覆盖的审计编号 | 交付物 |
|---|---|---|
| `ux-correctness` | U1, U2, U8, U9, D7, C11, C20（当前触及页面） | 高频路径上的视觉与交互正确性 |
| `settings-productization` | D3, D4, D5, D6, C9, C10, C13, C15, C16, C17, U5, U6, U7 | 每个普通设置都真正改变产品行为 |
| `reverse-image-saucenao` | D1, U3, C18 | 零配置 SauceNAO，多来源结果 |
| `comment-translation` | D2, U4, C19 | 百度 + 通用 LLM，凭据安全存储 |
| `behavior-correctness-cleanup` | C1–C8, C12, C14, C21, C22 | 删除错误边界，不是 architecture v2 |
| `network-perf-ab` | P-NET-1 ~ P-NET-5 | 先测量后改路线，无数据不加新路线 |
| `release-blockers` | R1, R2, R3 | 正式签名、API29 可用的验签、GitHub CDN 重定向 |

### R5. 跨 child 约束

1. **`settings-productization` 必须先于 `behavior-correctness-cleanup` 的 C4/C5/C22 落地。**
   D5 引入 SAF tree URI 目标会改变 destination 语义，而 download recovery 的 owner 定义
   正是 `stable accountId + job/output + destination`。cleanup 侧必须消费 D5 定稿后的
   destination 模型，否则两条线会在同一批代码上互相覆盖。
2. **`network-perf-ab` 不得因为「更像 PixEz」而引入 Cronet / QUIC / VPN。**
   第五节列出的 9 项网络历史问题（N-FIXED-1 ~ N-FIXED-9）在 HEAD 已实质修复，
   任何 child 都不得把它们当作待修项重新打开。
3. **`ux-correctness` 的 U1 不是小改动。** `lib/app/pull_to_refresh.dart` 是全部 feed
   页共用的滚动基建，其改动风险等级高于同 child 的其它项，必须单独验收。
4. **凭据不进普通 AppSettings。** D2 的百度 AppID/密钥与 LLM API Key 走安全存储，
   不出现在设置导出与日志中。

### R6. 明确禁止新增的东西

除非出现新的、具体的协议或平台需求，否则不新增：

`GlobalWorldRevision` / `IdentityEpochManager` / `OwnershipPolicyEngine` /
`EntityAuthorityGraph` / `MutationTransactionManager` / `RefreshGestureEpoch` /
`BallisticLease` / `NetworkFallbackStateMachine v2` / `AccountTransactionJournal` /
反向搜图 provider 注册表 / 翻译 provider 插件发现机制 / 自制 Android 文件浏览器 /
JavaScript 文件命名脚本 / Cronet / QUIC / VPN / 为「防止未来可能发生」而新增的
allowlist 或 deny guard。

> 澄清（针对审计原文的措辞歧义）：禁止的是**注册表与插件发现机制**。
> `lib/core/comments/comment_translation.dart` 已有 `CommentTranslationService` 接口与
> `_ConfiguredCommentTranslationService` 分发，D2 沿用它并新增实现类不属于本条禁令。

每个新 guard 写进代码前必须先回答：「它对应的真实协议、信任边界、平台要求或已复现
bug 是什么？」如果答案只是「以后可能」，就不要写。

## Acceptance Criteria

### Parent 级

- [ ] 7 个 child 全部完成实现、通过各自 check 并归档。
- [ ] R5 的跨 child 顺序约束在实际提交顺序中成立（settings 的 D5 先于 cleanup 的 C4/C5/C22）。
- [ ] R6 禁止清单在最终代码中没有被违反。
- [ ] 审计第五节的 9 项网络历史修复在最终 HEAD 上仍然成立，没有被任何 child 回退。
- [ ] `.trellis/spec/` 中沉淀了本轮产生的新约定（至少覆盖：设置项必须有真实消费者、
      guard 删除时同步删除固化错误行为的测试）。

### 最终真机验收（由用户亲自长期使用执行，不由 agent 代替）

parent 只负责提供可安装的 APK 与验收清单；下列结论以用户的真机使用为准：

- [ ] API 29 (Android 10) 与当前主流高版本 Android 各跑一轮。
- [ ] 大陆移动网络 / Wi-Fi、无代理与有代理环境（Automatic 不得破坏用户已有网络）。
- [ ] 首次登录走普通 OAuth 可完成。
- [ ] 已登录后：Recommended / Ranking / Search / User / Detail / token refresh /
      bookmark / follow / comment / image / download / Ugoira / history / widget
      （有实例与无实例）全部正常。
- [ ] kill process → restart → download recovery 状态可见可恢复。
- [ ] GitHub updater 在 API 29 与高版本 Android 上各完成一次真实安装。

> child 级的功能与界面改动仍按既有规则各自安装 APK 并截图留证到该 child 的
> `research/screenshots/`。本节是 parent 层的最终集成验收，不替代 child 级留证。

## Notes

- 审计断言复核结果与新发现的缺口：`research/audit-verification.md`。
- 本 parent 不承担直接实现工作，只持有源需求集、任务地图、跨 child 约束与最终集成验收。
