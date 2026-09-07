# 评论翻译：百度 + 通用 LLM

## Goal

把评论翻译从「只有 Google 一条路」扩成 D2 定的四条路径（关闭 / 百度 / 通用 LLM /
Google 兼容项），并为需要凭据的两条路径建立安全存储。当前 `translate.googleapis.com`
的 gtx 端点在中国大陆不可直连，等于对目标用户群翻译功能整体不可用。

覆盖审计编号：D2, U4, C19。

审计原文已由用户补回并归档到
`../09-01-func-1-0-hardening/research/audit-source.md`。本 PRD 依据其中 D2/U4/C19 的
决策与现状描述，以及当前 HEAD 的实测状态编写；原文基线是 `9d1cb1b`，实现前仍需重新
定位当前代码。百度额度、实名认证与 Google gtx 可用性属于会变化的外部事实，仍需 research
阶段复核。

## 现状核实（HEAD `409df51`）

`lib/core/comments/comment_translation.dart`（134 行）：

- `CommentTranslationService` 接口已存在（单方法 `translate(text, targetLanguage)`）。
- `GoogleCommentTranslationService`：走 `translate.googleapis.com` 的 `client=gtx`
  非官方端点，无凭据，15 秒超时，解析结果为空时报错。
- `DisabledCommentTranslationService`：抛 `CommentTranslationUnavailable`。
- `_ConfiguredCommentTranslationService`：读 `translationProvider` 分发，
  `disabled` 走禁用实现，**其余一律走 Google**。

`lib/core/settings/app_settings.dart`：

- `TranslationProvider` 枚举**只有** `google(0)` / `disabled(1)`，默认 `google`。
- `SecretSettingRef`（line 44-53）已定义，注释写明「指向 `CredentialStore` 记录的非机密
  指针，故意不参与 `AppSettings` 序列化」——**但当前全仓库无人使用它**。

`lib/core/auth/credential_store.dart`：`CredentialStore` 是 **按 accountId 存 `Credential`**
的专用存储（`read/write/delete(String accountId)`），不是通用 KV。存百度 AppID/密钥与
LLM API key 需要扩展该存储或新增一条同等安全级别的路径——这是 design 阶段的第一个决定点。

UI 侧（`lib/features/comments/comment_item.dart:115-150`）：

- 翻译是**非持久化 overlay**，仅在用户点击翻译图标时触发。
- 目标语言取 `Localizations.localeOf(context).languageCode`。
- 已区分 `CommentTranslationUnavailable`（未配置）与其它失败两种文案。

另注意：`accountTransferServiceProvider.exportCurrentToClipboard()`
（`settings_page.dart:171`）会把**账号凭据**明文放进剪贴板并对 Android <13 明确告警。
这是账号迁移功能，不是设置导出；但它确立了本仓库对凭据外流的处理基准——
翻译凭据不得被加进这条 payload。

## Requirements

### R1. 四条 provider 路径（D2）

`TranslationProvider` 扩为：关闭 / 百度 / 通用 LLM / Google。

- **百度**：D2 定的首选（大陆可用）。需要用户提供 AppID + 密钥。
- **通用 LLM**：OpenAI-compatible，用户提供 base URL + model + API key。
- **Google**：降为兼容项，保留现有实现，不再是默认。
- **关闭**：保留现有 `DisabledCommentTranslationService` 行为。

默认值需要在 design 阶段定：现有默认是 `google`，而 D2 把它降级了；但百度和 LLM 都
需要凭据，无法作为开箱默认。倾向默认「关闭」+ 首次点击翻译时引导去设置页，
但这会改变现有用户的行为，需在 design 阶段确认。

### R2. 百度的实名认证门槛必须前置告知（G3）

复核文档 G3 已经指出：D2 采信的「100 万字符/月」属于**个人高级版，需要实名认证**；
未认证的标准版是 5 万字符/月 + QPS 1，用于评论翻译基本不够。

- 设置页的百度配置引导必须写明这一步，不能等用户配完发现不够用。
- 不做「额度用尽自动降级到别的 provider」——那是静默 fallback，违反全局 Debug-First 约定。
  额度问题必须作为可见错误呈现。

### R3. 外部事实复核是硬前置

以下三条是**外部事实**，会随时间变化，实现前必须复核，不得直接采信审计的时点结论：

1. 百度翻译开放平台当前的额度政策与实名认证要求（决定 R2 的引导文案，也决定
   「首选百度」这个结论本身是否还成立）。
2. 审计称腾讯云 `TextTranslate` 已下线——若该结论有误，腾讯可能重新成为候选。
3. Google gtx 端点是否仍可用（决定兼容项是否还有意义）。

**该复核未能在规划环境完成**（WebSearch 不可用）。留给 child 的 research 阶段，
或由用户直接提供结论。若复核推翻「首选百度」，回到用户重新决策 D2，不自行改选。

### R4. 凭据安全存储（parent R5.4）

- 百度 AppID/密钥、LLM API key **不进 `AppSettings`**，不参与其 JSON 序列化。
- `AppSettings` 中只保留 `SecretSettingRef` 这种非机密指针（该类已为此存在）。
- 凭据不出现在日志、错误消息、崩溃上报中。
- 凭据不得被加入 `accountTransferService` 的剪贴板 payload。
- 用户可以清除已保存的凭据。

### R5. 不建 provider 框架（parent R6 的澄清）

- 沿用已有的 `CommentTranslationService` 接口 + `_ConfiguredCommentTranslationService`
  分发结构，新增实现类。
- **不新建**注册表、插件发现机制、动态 provider 加载。parent R6 的禁令指向的是这些，
  不是接口本身（G2 已澄清）。

### R6. 隐私边界保持

现有实现已经承诺并做到的，不得放松：

- 只在用户明确点击翻译时发送该条评论文本。
- 不持久化原文与译文。
- 不记录评论正文到日志。

新增约束：使用 LLM provider 时，请求体只含待翻译文本与必要的指令，不附带用户账号信息、
作品 ID、其它评论上下文。

### R7. 失败必须可区分

当前只有「未配置」与「失败」两类文案，扩 provider 后至少要能区分：

- 未配置 / 已关闭
- 凭据无效（配错了，用户能改）
- 额度或频率超限（配对了，等一等或换 provider）
- 网络失败（可重试）

不允许把凭据错误显示成网络错误——那会让用户反复重试一个永远不会成功的请求。

## Acceptance Criteria

- [ ] 设置页可选择四条路径；选中需要凭据的路径时提供输入与保存。
- [ ] 百度路径配置成功后，中文以外语言的评论可翻译出结果。
- [ ] 通用 LLM 路径填入 base URL / model / key 后可翻译出结果。
- [ ] Google 路径行为与当前一致（回归不破坏）。
- [ ] 关闭时点击翻译显示「已关闭」，不发出任何网络请求。
- [ ] 百度配置引导中出现实名认证与额度说明。
- [ ] 凭据不出现在 `AppSettings` 的 JSON 中（单测断言序列化结果）。
- [ ] 凭据不出现在账号迁移 payload 中（单测断言）。
- [ ] 凭据无效 / 超限 / 网络失败三类错误在 UI 上可区分。
- [ ] 清除凭据后该 provider 回到未配置状态。
- [ ] `flutter analyze` 与 `flutter test` 通过。
- [ ] 真机验证清单交付（大陆网络下百度路径真实可用是本 child 的核心价值，必须真机确认）。

## Open Questions

- **R3 的三条外部事实复核结论**仍需 research 阶段完成；若事实推翻 D2，暂停并上报，不
  自行换 provider。
- 产品选择已确认：新安装默认关闭、旧 Google 兼容；凭据使用独立安全 namespace；LLM prompt
  固定且不可配置。实现阶段只需记录事实复核结果，不再重新讨论这些选择。

## Notes

- 上级需求与跨 child 约束见 `../09-01-func-1-0-hardening/prd.md`（R5.4 凭据约束、
  R6 禁令与 G2 澄清）。
- 复核文档的 G3 是本 child 的必读项。
- 本 child 与其它 child 的代码面几乎不重叠，可并行安排。
