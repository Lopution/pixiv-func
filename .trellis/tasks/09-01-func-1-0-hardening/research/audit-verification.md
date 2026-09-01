# 审计断言复核（2026-09-01）

复核基线：HEAD `9d1cb1b`，工作树 clean。
方法：对 2026-09-01 审计文档中的每条代码断言，在当前代码上直接定位并核对。

## 一、断言复核结论

**全部成立。** 未发现错判，也未发现「已修但仍被列为问题」的项。

| 编号 | 断言 | 复核结果 | 证据位置 |
|---|---|---|---|
| U1 | 下拉刷新用 `RefreshIndicator.noSpinner` + `NotificationListener` + 自绘指示器 | 成立 | `lib/app/pull_to_refresh.dart:93,204,224` |
| U2 | trending tag 卡片只显示文字，representative 仅用于长按 | 成立 | `lib/features/search/search_page.dart:157-195`；API 侧已解析 `representative`（`lib/core/search/search_repository.dart:327-340`） |
| U9 | 详情页作者区域漏 tap | 成立。`showUserPage` 只被 caption 内 pixiv 用户链接消费 | `lib/features/illust/detail/illust_detail_page.dart:867` |
| C5 | recovery 依赖 `downloadManagerProvider` 被实际创建 | 成立 | `lib/core/download/download_providers.dart:45` |
| C6 | App 侧 eager start WidgetCoordinator | 成立 | `lib/core/widget/widget_coordinator.dart:120-121` |
| C7 | Home 用 IndexedStack，child build 即 watch 自身 feed | 成立 | `lib/features/home/home_page.dart:217` |
| C8 | `UserRoute` 可解析但 Home consumer 只处理 `IllustRoute` | 成立 | `lib/core/platform/intent_router.dart:294,308,321` vs `lib/features/home/home_page.dart:137` |
| C9 | R18/AI/blockedTags 设置存在但 feed 层无过滤 consumer | 成立。仅在设置页与详情页 tag 展示被读取 | `lib/features/settings/settings_page.dart:674,682`；`lib/features/illust/detail/illust_detail_page.dart:563` |
| C10 | Accept-Language 默认硬编码 `zh-CN`，无 settings 注入 | 成立 | `lib/core/network/pixiv_headers.dart:15,27`；`lib/core/network/pixiv_http_client.dart:55` |
| C11 | LoginPage 设置读取失败直接白屏 | 成立。loading 与 error 都返回空 Scaffold | `lib/features/login/login_page.dart:120-122` |
| C12 | Recommended `fetchPage()` 直接 throw | 成立 | `lib/features/home/recommended/recommended_feed_controller.dart:28` |
| C13 | `ImageSourceMode` 只有两个值却仍暴露为可选设置 | 成立 | `lib/core/settings/app_settings.dart:7-14` |
| C15 | `AppSettings` 无 `networkMode` 字段，重建即回 automatic | 成立。`NetworkMode` 只存在于网络层 | `lib/core/network/compat/network_contracts.dart:89`；settings 层无该字段 |
| C16 | native login intercept 生产 wiring 仍在 | 成立 | `lib/features/login/login_page.dart:55`；`lib/features/settings/network_settings_page.dart:264` |
| C17 | 全局 `insecureNoSniEnabled` 暴露为普通用户开关 | 成立 | `lib/core/settings/app_settings.dart:64,123-125`；`lib/features/settings/network_settings_page.dart:244` |
| C20 | 生产 UI 仍有硬编码中文 | 成立（另发现一处：`'无法打开链接：$error'`） | `lib/features/illust/detail/illust_detail_page.dart:806` |
| D4 | `previewQuality` / `scaleQuality` 为两个语义不清的 bool | 成立 | `lib/features/settings/settings_page.dart:645-655` |
| D6 | `namingRule` 可保存但下载侧不消费 | 成立。设置页有输入框，`namingRule` 无下载侧引用 | `lib/features/settings/settings_page.dart:727-760` |
| R1 | release 仍用 debug signing | 成立 | `android/app/build.gradle.kts:61-62` |
| R2 | updater verifier 用 Ed25519（API 33+） | 成立 | `android/app/src/github/kotlin/.../DistributionUpdaterChannel.kt:61-63` |
| R3 | 下载 host allowlist 不含 `release-assets.githubusercontent.com` | 成立 | `lib/core/download/download_request.dart:143-152` |
| C1 | `credentialRevision` 被当作全局世界版本 | 成立，且范围比审计描述更大 | 19 个文件 79 处引用 |

## 二、审计文档本身的缺口

复核过程中发现 4 处审计未覆盖或表述不足的问题，已分配给对应 child。

### G1. R2 遗漏了签名的生成侧（→ `release-blockers`）

审计写「更新生成签名脚本和 Android verifier」，但仓库里 **`tool/` 目录为空，不存在
manifest 签名脚本**，`.trellis/scripts/` 下也没有。R2 的真实工作量不是「把
`Ed25519` 换成 `SHA256withECDSA`」，而是从零建立签名生成侧：密钥生成、manifest 签名
脚本、CI 接线，再加 verifier 改造。排期必须按前者的数倍计。

另注意 `DistributionUpdaterChannel.kt` 位于 `android/app/src/github/`，是 flavor
特定实现；`fdroid` flavor 的分发策略需要一并确认。

### G2. 禁令清单与 D2 冲突（→ 已在 parent `prd.md` R6 澄清）

审计第十二节禁止 `TranslationProviderFramework`，但 D2 要求同时支持
关闭 / 百度 / LLM / Google 四条路径。而 `lib/core/comments/comment_translation.dart:33-132`
已经存在 `CommentTranslationService` 接口与 `_ConfiguredCommentTranslationService`
分发结构。若不澄清，实现方会在「能否新增实现类」上空转。禁令实际指向的是
注册表与插件发现机制，不是接口本身。

### G3. D2 未写百度翻译的实名认证门槛（→ `comment-translation`）

审计据「100 万字符/月」推荐百度，但该额度属于**个人高级版，需要实名认证**。
未认证的标准版是 5 万字符/月 + QPS 1，用于评论翻译基本不够。

产品后果：D2 的「用户自己填 AppID + 密钥」实际包含一次实名认证。这一步必须写进
child 的 PRD 与设置页引导文案，不能等实现时才发现。同时，「首选百度」这一结论本身
需要在实现前复核当前 API 可用性与额度政策 —— 审计中关于腾讯云 `TextTranslate`
已下线的结论同理，属于需复核的外部事实而非既定前提。

### G4. C9 是未决的产品决策（→ `settings-productization`）**［已决策］**

审计给出两个互斥选项（接线 or 删开关）但没有定。

**2026-09-01 用户决策：做实现，不删开关。** 范围与分页策略见
`../09-01-settings-productization/prd.md` R1。要点：

- 适用发现类列表 + 桌面小组件；收藏、历史、详情页不适用。
- 过滤后自动续拉直到填满一屏，续拉次数有上限。

实现约束（复核 HEAD 得出，审计原文未覆盖）：审计称把 predicate 放在「共享 Feed
presentation boundary」，但该边界**当前不存在** —— `lib/app/widgets/` 下没有共享
illust grid，各 feed 页自行渲染。真正的共享处是 `PagedFeedController`（state 为
`List<int> ids`，实体在 `IllustStore`）。同时 `WidgetFeedLoader` 走独立路径
（`lib/core/widget/widget_feed_loader.dart:136` 直接 `fetchPage(null)`），不经过
feed 状态机。因此过滤规则必须实现为可被两条路径共同消费的纯谓词，位置在 design
阶段定稿。

## 三、需要在实现中重新评估工作量的项

- **C1（`credentialRevision` 收敛）**：审计归类为「cleanup」，方向正确（不应造
  `GlobalRevision2`），但涉及 19 个文件 79 处引用，横跨 feed / mutation / download /
  ugoira / widget / profile。应作为 `behavior-correctness-cleanup` 内部的独立阶段，
  且必须排在 UI 类 child 之后，避免与其改动冲突。
- **U1（下拉刷新）**：`lib/app/pull_to_refresh.dart` 被全部 feed 页共用，属共享滚动
  基建重写，风险等级高于同 child 的其它项，需单独验收。
