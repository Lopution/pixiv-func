# Dart 架构收敛：分层、单一 owner 与现代化（09-02 child C）

## Goal

让 `lib/` 的分层与依赖方向有明文规则并被测试校验，每个跨层契约只有一个 owner，删除零引用与重复实现，
并完成移植后不彻底的现代化（i18n、状态范式、分页、文件规模）。行为、协议、持久化格式不变。
对应 parent R3（最小实现部分）、R4、R5、**R8（UI 组件体系）**、**R9（性能约束）**；事实来源
`research/dart-architecture-audit.md`、`research/modernization-gaps.md`、`research/runtime-baseline.md`；规则与目标形态见
parent `design.md` §2–§3、§13–§14。已确认决策 D-6（gen-l10n）；D-9/D-10 待确认（推荐默认已写入 parent）。

## 开工 gate

- 触及 `lib/` 的 09-01 child 全部归档：`settings-productization`、`behavior-correctness-cleanup`、`network-perf-ab`、
  `reverse-image-saucenao`、`comment-translation`（`ux-correctness` 已完成）。
- P0 运行时基线（C 前）已由用户在真机记录到 `research/runtime-baseline.md`。
- 开工第一步：对当时 HEAD 重跑两份研究的计数脚本，更新"已知违反清单"。
- `login_intercept_controller.dart` 及其原生端的去留以 `09-01-settings-productization` 的结论为准，本 child 只清理残留。

## Requirements

### R1. 规则先行

- `test/architecture/layering_test.dart`：`lib/core/**` 不 import `lib/features/**`；features 之间只 import
  `lib/app/navigation/routes.dart`；repository/controller/entity 只在 `lib/core/<domain>/`。首次以已知违反白名单通过，
  每消除一项就删白名单；本 child 结束时白名单为空。
- 规则写入 `.trellis/spec/frontend/directory-structure.md`（与 child E 协作，C 负责规则文本）。

### R2. 分层修正（design §2.1）

- `IllustCard`（`recommended_illust_page.dart:236`）与瀑布流 sliver（8 处 `SliverMasonryGrid.count`）移到 `lib/app/widgets/`。
- 32 个私有 feed 尾部/空态/错误 widget → `lib/app/widgets/feed_states.dart` 的 `FeedTail/FeedEmpty/FeedError`（先列差异矩阵）。
- 5 个 `features/` 内 repository、6 个 controller 迁入 `lib/core/<domain>/`，controller 与 repository 分文件；
  消除 `core/widget/widget_feed_loader.dart → features` 与 `app ↔ onboarding` 互引（`startup_gate.dart` 上移 `lib/app/`）。
- 导航合并到 `lib/app/navigation/`：`ReplicaPageRoute` 唯一（删 `replicaRoute()`）、`routes.dart` 门面、`HomeShellMetrics` 静态量改 provider。

### R3. 单一 owner（design §2.2）

- 主机/头：`PixivDestinationRegistry`、预热列表、fast-route 引导、探测页、登录拦截、`illust_detail_repository.dart:18`
  全部从 `PixivClientIdentity` 派生；新增 `PixivHeaders.image()` 供 4 个下载/图片调用点。语义不变。
- `sharedPreferencesProvider` + `PreferenceKeys`（键值不变，不迁移数据）替换 7 处直接构造。
- `lib/core/entity/json_read.dart` 替换 `_map`×5、`_firstString`×3、`_positiveInt`×5、`_optionalString`×4、`_requiredString`×2、
  `_nonNegativeInt`×2、`_nextUrl`×5；容器类型统一 `Map<String, Object?>`。
- `showAppSnackBar` 替换 44 处手写 SnackBar；`log()` 替换 23 处 `debugPrint`。
- 生产代码禁止内联 `http.Client()`/`HttpClient()`：第三方与解析器客户端经 provider 注入（三套栈保留，理由写 spec）。

### R4. 现代化（design §2.3、§3）

- i18n 迁到 `gen-l10n` + ARB（D-6）：迁移脚本把 `ReplicaStrings` 四语言生成 ARB；补齐 ru `detailQuality`、
  修复未定义 `networkDohEndpointsInvalid`；删除 27 个 `_xxxText` 包装与 `ReplicaStrings`；`ReplicaLanguage` 保留为设置枚举。
- 3 个 `ChangeNotifier` 控制器（ProfileEdit、ReverseImageSearch、NovelReader）改 Riverpod `Notifier`；页面不再 `new` 平台适配器。
- `history_page` 改用 `PagedFeedController`（`HistoryFeedController`），保持 account 切换与 outbox 语义。
- 颜色收敛到 `FuncTokens`/主题；保留 replica 视觉决定的 `Colors.*`。

### R5. 删除与拆分（design §2.4）

- 删除零引用：`compat_network.dart`、`android_platform.dart`、`tag_search_repository.dart`（视测试）、13 个零引用声明、
  8 个未用 `AppIcons` 常量（放宽 `icon_font_test`）；142 个仅本文件使用的公共声明按需私有化。
- 拆分 `settings_page.dart`（14 页面类，`MePage` 去重）、`illust_detail_page.dart`（`_GlobalRectClip` 公开化，
  `hero_transition_test` 改 `find.byType`）、`user_page.dart`、`network_policy.dart`（策略 / `PixivNetworkFactory` / 图片缓存所有权分离）；
  5 个 >150 行方法拆分。

### R6. 不做

- Material 3、`material_ui`、声明式路由、json 代码生成、Result 类型、新 Exception 基类框架；不改 `catch` 行为与错误分类语义；
  不改 `NetworkAccessPolicy` 路由阶梯/ECH/SNI/证书/fallback 顺序；不改任何持久化 key 或格式。

### R7. UI 组件体系（parent R8，design §13）

- `lib/app/` 按 design §13.1 组织：`theme/`、`motion/`（`replica_page_route.dart`、`HeroRectClip`、`motion_tokens.dart`）、
  `navigation/`（门面）、`widgets/feed/`（`IllustCard`、`IllustFeedGrid`、`FeedTail/Empty/Error`）、`widgets/pixiv_image.dart`、
  `widgets/feedback.dart`、settings 原语。
- `IllustFeedGrid` 统一 8 处 `SliverMasonryGrid.count`（当前全部写死 `crossAxisCount: 2`），列数由 `illustColumnsFor(crossAxisExtent)`
  计算，手机宽度恒为 2 列。
- `PixivImage` 增加 `PixivImageSize.feed/detail/viewer/avatar` decode 策略与命名构造器（含 `.hero(tag:)`）；13 个调用点切换；
  `PersonAvatar` 复用 avatar 变体；Hero 转场契约（`component-guidelines.md`）不变。
- `motion_tokens.dart` 是全部时长/曲线的单一来源。
- 无障碍基线只针对共享组件与收藏/关注/下载/导航四类动作：语义标签或 tooltip、≥48dp 触达、可聚焦可键盘激活；14 处
  `GestureDetector` 逐个评估改 `InkWell`/`IconButton` 或补 `Semantics`。
- 新组件需 ≥3 处现存重复作为证据（`NovelCard`/`UserCard` 先盘点）。D-10 的交互/视觉项不实现。

### R8. 性能边界（parent R9，design §14）

- feed/avatar 图片按布局尺寸 decode，detail 按屏宽，viewer 不限；以 P0 基线的 PSS（50/200 张）对照。
- 重建边界：先用 DevTools rebuild 统计推荐页滚动与一次收藏操作；只有整卡重建被证实才引入 `select`/拆分，结论写入基线文件。
- 首帧前等待项（settings + 账号）不得增加；新初始化 `unawaited` 或后置。
- `DownloadRecoveryStore` 500 条记录写入耗时、history 三条查询 `EXPLAIN QUERY PLAN` 有记录；超出可接受范围才改增量写入。
- 重复请求只统计不预设；有证据再决定 repository 层 in-flight 共享，不建全局缓存层。
- C 完成后用户复测基线；首帧 +10%、PSS +10%、jank +2 pp 以上视为回归，回滚对应提交。

## Acceptance Criteria

- [ ] `layering_test` 白名单为空并在 CI 通过；`rg "features/" lib/core` 为 0。
- [ ] Pixiv 主机字面量只在 `pixiv_client_identity.dart` 一处声明（注释除外）。
- [ ] `rg "SharedPreferencesAsync()" lib` 为 0；`rg "http.Client()|HttpClient()" lib` 在生产路径为 0。
- [ ] `gen-l10n` 生成成功，四语言 ARB key 集合一致，`ReplicaStrings` 与 27 个包装已删除。
- [ ] `rg "ChangeNotifier" lib` 为 0；`history_page` 无手写 `ScrollController` 分页。
- [ ] 零引用文件/声明为 0；`unreachable_from_main` 开启且 0 问题。
- [ ] `settings_page.dart`、`illust_detail_page.dart`、`user_page.dart`、`network_policy.dart` 拆分后单文件 ≤ 600 行。
- [ ] `rg "crossAxisCount: 2" lib/features` 为 0；`PixivImage` 13 个调用点全部使用变体构造器；`motion_tokens.dart` 外无时长/曲线字面量；
      共享组件 semantics 测试通过（与 E 协作）。
- [ ] P0 复测：浏览 200 张后 PSS 不高于基线；首帧、jank 在 R9 阈值内；重建与重复请求结论已记录。
- [ ] `flutter analyze`、`flutter test` 全绿，测试数不减少；Hero 转场、Tab 动画、下拉刷新、`PagedFeedController` 三相语义测试通过。
- [ ] 真机：登录、浏览、详情、用户页、历史、设置各页面行为与迁移前一致。

## Notes

- 复杂 child：`task.py start` 前需从 parent design §2–§3 与 implement "Child C" 派生本目录的 `design.md`（差异矩阵、
  i18n 迁移脚本设计、门面 API）与 `implement.md`（C0–C9 清单，每项一个提交）。
- 停止条件：任何拆分改变 `component-guidelines.md` 的三个契约或 `PagedFeedController` 三相语义 → 回滚该提交。
