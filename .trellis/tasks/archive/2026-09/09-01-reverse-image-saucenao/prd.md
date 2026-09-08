# 反向搜图：零配置 SauceNAO

## Goal

把已经建好的反向搜图流程接上真实 provider。整条管线（输入、所有权、状态机、结果映射、
外链安全、i18n 文案）在 HEAD 上都已存在且有测试覆盖，唯一缺的是 transport——
`reverse_image_search_page.dart:41-48` 硬接了 `UnavailableReverseImageProvider`，
用户点进反向搜图只能看到「暂不可用」。

覆盖审计编号：D1, U3, C18。

审计原文已由用户补回并归档到
`../09-01-func-1-0-hardening/research/audit-source.md`。本 PRD 依据其中 D1/U3/C18 的
决策与现状描述，以及当前 HEAD 的实测状态编写；原文基线是 `9d1cb1b`，实现前仍需重新
定位当前代码。SauceNAO 匿名调用策略、额度与限流属于会变化的外部事实，仍需 research
阶段复核。

## 现状核实（HEAD `409df51`）

已经存在、不需要重建的部分：

| 组件 | 位置 | 状态 |
|---|---|---|
| 输入引用与校验 | `lib/core/reverse_image/image_input.dart` (467 行) | 限制已定：10MB / 8192px / 16M 解码像素；仅 PNG·JPEG·GIF·WebP；MIME 与实际格式必须一致；header 解析有界，不在 UI isolate 解全图 |
| 临时文件所有权 | 同上 `OwnedReverseImageInput` | 一次流程持有一个临时文件，注入 delete 回调，终态释放 |
| 平台桥 | `reverse_image_platform.dart` (191 行) | picker 与 ACTION_SEND 两个入口 |
| 流程状态机 | `reverse_image_controller.dart` (351 行) | 8 状态 + CancelToken + generation 防串场 |
| **SauceNAO 结果映射** | `reverse_image_provider.dart` 的 `ReverseImageResultMapper.fromSauceNaoJson` | **已完成**：similarity 校验（0–100）、`pixiv_id` 提取、`ext_urls` 安全解析、缩略图、按 key 去重取最高相似度、稳定排序 |
| 外链安全 | `reverse_image_external.dart` | 仅 https、无 userInfo / port / fragment，否则拒绝 |
| 失败分类 | `ReverseImageProviderFailureCode` | `providerUnavailable / cancelled / network / rateLimited / malformedResponse / unsafeResultUrl` 六类已定义 |
| i18n | `replica_strings.dart:213,230-239` | 9 条文案已就位（含隐私提示） |

缺的部分：**只有 transport**。没有任何代码向 SauceNAO 发出过请求。

## Requirements

### R1. 零配置（D1）— 含一项硬前置复核

- 用户不需要注册账号、不需要填 api_key 即可使用反向搜图。
- **硬前置**：实现前必须复核 SauceNAO 当前的匿名调用策略——是否仍允许不带 `api_key`
  调用、免费额度、限流窗口（每 30 秒 / 每日）。用户已确认采用零配置 WebView 路径；复核
  只决定当前服务是否可用，不把失败静默改成要求 key。
- 若复核结论是匿名调用已不可用，**D1 的前提就不成立**，必须回到用户重新决策，
  不允许悄悄降级成「让用户自己填 key」——那是另一个产品，不是 D1。
- 该复核**未能在规划环境完成**（WebSearch 不可用；`saucenao.com/user.php?page=search-api`
  返回 403）。留给 child 的 research 阶段，或由用户直接提供结论。

### R2. Transport 与 WebView 实现

- 采用 PixEz 式 WebView/HTML 路径：用户确认搜索后，将本地压缩图片以 multipart POST 提交
  `search.php`，在受控 WebView 中显示结果页面。
- Pixiv 作品链接拦截到 Func 原生详情，作者链接走现有用户页；其它来源保持页面导航或交给
  系统浏览器。
- 不新增结构化 provider registry、mirror 或用户 API key 设置；若后续确需原生结果，再另
  开 task 基于真实 HTML/API 设计。
- WebView 加载失败、HTML challenge、429/限流、超时和取消必须是可见失败并清理临时文件，
  不渲染“成功但空结果”。

### R3. 结果不限制 Pixiv 来源（D1）

- `ReverseImageHit` 已经同时支持 `pixivId` 与 `externalUrl`，两者至少有一个。
- 有 `pixivId` 的结果点击进入站内作品详情页。
- 只有 `externalUrl` 的结果经 `ReverseImageExternalLauncher` 交给系统浏览器。
- **不因为「不是 Pixiv 来源」就过滤掉结果**——D1 明确要求不限制来源。

### R4. 隐私边界不得放松

当前代码与文案已经向用户承诺了三条，实现必须全部保持：

- 图片只在用户明确点击「开始反向搜图」后才发送（`searchReversePrivacyDetail`）。
- 取消、失败、成功三条路径都清理临时文件。
- 除图片本身外不发送任何身份信息：不带 Pixiv token、不带账号 ID、不带设备标识。

追加约束：不持久化反向搜图历史。

### R5. 失败必须是可见终态

- 沿用 `ReverseImageSearchFailure`，不允许出现「空成功」——即返回 `ReverseImageSearchSuccess`
  且 `hits` 为空却不告诉用户发生了什么。
- 「无匹配结果」与「请求失败」在 UI 上必须可区分。

## Acceptance Criteria

- [ ] 用一张 Pixiv 作品的截图搜索，能命中该作品，点击进入站内详情页。（**用户真机项**；代码侧：SauceNAO 的 `member_illust.php?mode=medium&illust_id=` 与 `/en/artworks/` 已映射到 `IllustRoute` → `IllustDetailPage`，`intent_router_test` / `sauce_nao_navigation_policy_test` 覆盖）
- [ ] 用一张非 Pixiv 来源的图搜索，结果可点击并由系统浏览器打开。（**用户真机项**；代码侧 `openExternal` → 安全 launcher，单测覆盖）
- [x] 触发限流时显示「可重试 + 等待时间」，而不是泛化的失败文案。（429 `Retry-After` 秒数 / 30 s 窗口 → `searchReverseRateLimitedWait`；日限额 → `searchReverseDailyLimit`；page test 分别断言两种文案）
- [x] 服务端返回 HTML / 挑战页时显示明确失败，不显示空结果列表。（`challenge` 码 + `searchReverseChallenge` 文案；Cloudflare interstitial / Attention Required / turnstile 标记测试）
- [x] 「无匹配结果」与「请求失败」在界面上是两种不同的呈现。（无匹配页 → 空成功 → `searchReverseNoResults`；失败走 `_failure` 组件；page test 覆盖两者）
- [x] 取消 / 失败 / 成功三条路径均已清理临时文件（扩展现有的 exactly-once 清理测试）。（`reverse_image_search_test`：cancel / rate limit / WebView success 各 exactly-once）
- [x] 全流程不要求用户填写任何 key 或注册任何账号。（reverse-image 目录无 `api_key`/`apiKey` 引用；匿名 HTML 路径实测可用，见 `research/anonymous-policy.md`）
- [x] `flutter analyze` 与 `flutter test` 通过。（2026-09-08）
- [x] 真机验证清单交付（结果跳转、限流提示、隐私提示可见）。（见 `implement.md` 最终验证「真机」一项；执行由用户完成）

## Open Questions

- R1 的匿名调用事实仍需 research 复核；若服务不可用，保持明确 unavailable，不改成 key/mirror。
- 限流数字不写死在普通 UI，只在实际响应中展示等待信息。以上为实现 gate，不是待用户选择。

## Notes

- 上级需求与跨 child 约束见 `../09-01-func-1-0-hardening/prd.md`。
- 排序与依赖见同文件 R5「执行顺序与 gate」。
- 本 child 与其它 child 的代码面几乎不重叠，可在 settings / cleanup 之外并行安排。
