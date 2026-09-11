# 技术设计：全项目 UI 视觉与交互重构

本设计只描述实现边界和可验证契约，不改变 Pixiv API、账户、下载、缓存或阅读状态的业务语义。PRD 中的全局视觉原则和样板链路已获用户确认。

## 0. 现状与已合并前置（2026-09-11 复核）

已随首个 PR 合入 main 的**前置修正**（不计入任何阶段完成度，实现时直接在其之上继续）：

- `fix(nav)`：`tag/:keyword` 与各推送路由收敛为 `_commonBranchRoutes`，`/reverse-image` 成为 root-level stack root；`openIllust/openUser/openMe/...` 门面改为 `_currentStackRoot`（留在当前栈）。
- `fix(ui)`：去掉 M3 `NavigationBar` 下方多余 margin（golden `home_bar.png` 已更新）。
- `fix(illust)`：内联 `UgoiraViewer` 移除 `DragToDismiss`（全屏 `image_viewer_page` 仍保留，属预期）。
- `refactor(search)`：搜索输入页从 `SearchAnchor` 视图改为内联 `SearchBar` + 页面内建议面板、进入自动聚焦。

复核确认的现状事实（实现须以此为基线，不是凭 PRD 假设）：

- `FuncTokens` 已存在为 `abstract final class` 静态常量（非 `ThemeExtension`）；`replica_theme.dart` 将 `surfaceContainerLow~Highest` 全部压平为同一 `surface`。
- 已有共享组件：`feed/`（`IllustFeedGrid`、`IllustCard`、`FeedTail`、`FeedEmpty`、`FeedError`——**没有 `FeedLoading`**）、`replica_*` 系列、`settings/` tiles、`novel_card`、`bookmark/follow_switch_button`、`app_snack_bar`、`person_avatar`、`pull_to_refresh`、`pixiv_image`。
- `/settings` 当前就是第 5 个 shell branch（带 11 个子页路由、`includeHistory: false`）；`/me` 只是每个 stack root 下的子路由，**唯一可见 UI 入口**是 设置→账户管理（`account_settings_page.dart:36`），但路由层能力（子路由 + `openMe` + 深链形态）已具备。
- shell 无统一 AppBar：`home_page` 仅 `body + bottomNavigationBar`，五个 tab 页各自持有 AppBar；宽屏 `NavigationRail`/`Drawer` 尚不存在；`home_shell_metrics` 为 Hero 测量底栏。
- `_illustColumnsFor` 已存在但为 private 且吃 `MediaQuery` 整窗宽；`test/goldens/` 仅 `home_bar.png` 一张，golden 基建近乎为零。

```text
AppShell
├─ Primary destinations: 推荐 / 排行 / 最新 / 搜索 / 我的
├─ Shell action: 应用设置（/settings，与“我的”平级）
└─ Route content: feed → illust detail → user/my → secondary pages
```

窄屏使用 `NavigationBar`（或项目当前等价壳层），宽屏使用 `NavigationRail`/Drawer；五个目的地始终保持相同语义和顺序，导航形态随可用空间变化而不是随设备型号硬编码。`/settings` 不占用个人内容导航槽位，在窄屏由 shell 级齿轮/应用菜单打开，在宽屏作为 rail/drawer 的同级 action 显示。

## 2. 主题与 tokens

在 `lib/app/theme/` 分两层：**`FuncTokens` 保留为基础常量 owner（具体色值/数值的唯一存放处）**，其上新增语义化 `ThemeExtension`（引用常量、不重复维护色值），组件只消费语义 token，不直接拼装颜色和间距：

| 类别 | 语义层 | 约束 |
| --- | --- | --- |
| Color | canvas、surface、surfaceRaised、surfaceOverlay、divider、contentPrimary/Secondary/Tertiary、brand、onBrand、danger/success/warning | 亮/暗主题成对定义；正文与交互文字通过对比度检查 |
| Type | display、title、body、label、caption、numeric | 建立字号/行高/字重阶梯；长标题允许自然换行 |
| Space | 2/4/8/12/16/24/32/48 | 仅保留有限尺度，页面不新增任意数字 |
| Shape | card、control、pill、sheet、dialog | 卡片与控件分别定义，避免所有元素同一圆角 |
| Motion | short、standard、emphasized、reducedMotion | reduced-motion 下不隐藏状态、不依赖动画完成 |

`ThemeData` 只负责将 tokens 投影到 Material 组件主题；Cupertino/原生承载页通过同一组语义值适配。现有品牌色若与可读性冲突，优先保证内容和状态对比度，品牌色只用于强调而非大面积背景。

## 3. 共享组件边界

组件放在 `lib/app/widgets/`，页面只组合，不复制近似实现。**对 §0 清单中已存在的组件只做扩展/改名/补缺，不新建同义组件**（如不得再造一套与 `ReplicaScaffold`/`feed/*` 平行的 scaffold 或卡片家族）；设计中出现的目标名字（`AppScaffold`、`PageHeader` 等）落到实现时映射到现有家族或显式重命名。

- `IllustCard`/`IllustFeedGrid`：媒体、标题、作者、统计和快捷操作为可选槽位；卡片本身不 watch 整个 feed state。
- `MediaPreview`：统一缩略图比例、裁剪、占位、失败重试和受限标识；详情/Ugoira 通过命名变体表达差异。
- `AuthorSummary`、`ActionBar`、`TagChips`：明确 `Semantics`、tooltip、触控目标和异步状态。
- `FeedStates`：`FeedLoading`、`FeedEmpty`、`FeedError`、`FeedTail`，文案、图标和 action 通过参数注入；禁止页面私有 `_...Loading/_...Empty/_...Error` 新增。
- `AppScaffold`/`PageHeader`/`FilterBar`：统一安全区、滚动行为、标题层级和窄/宽屏排列。

作品卡片保留原始宽高比，元数据（作者、标题、统计、角标）使用固定层级和有限行数；不以统一固定高度换取“整齐”。详情页组合顺序固定为主视觉、作者/标题、主操作、标签/说明、评论/相关作品。作者头部只承载沉浸式视觉和核心身份，完整 tab/操作在其下方展开；搜索首屏的 `SearchBar`、筛选、历史和趋势是同一信息区，趋势点击发出显式 search intent。

共享组件的状态输入来自现有 Riverpod provider 或显式参数。组件不得创建 repository、HTTP client、偏好存储或平台 channel；异步操作由页面级 controller 持有，组件只报告 intent 并渲染 `idle/loading/success/failure`。

## 4. 路由与状态边界

- `go_router` 的 shell branch 对应五个一级目的地；每个 branch 保留独立导航栈和滚动位置。
- `/me` 升为第 5 个 branch：从 `_commonBranchRoutes` 移除 `me` 子路由，`openMe` 门面改为 branch 切换；`_stackRoots` 增加 `/me`；底栏第 5 项换“我的”图标与 l10n 文案（四个 arb 都要加 key）。旧 `/{stack}/me` 路径无外部消费者，直接移除不保留重定向。
- `/settings` 改为 root-level route：整棵 `/settings/...` 子路由树原样挂到 root（URL 空间不变，深链/返回/状态恢复兼容），沿用 `settingsNavigatorKey`/`settingsRouteObserver` 与 `includeHistory: false` 的语义；`_stackRoots` 中 `/settings` 保留（其页面仍可承载推送）。
- 应用设置入口：窄屏在**五个 tab 页现有 AppBar 的 actions** 中注入同一个共享 settings action（不给 shell 加第二套 chrome，也不五份拷贝）；宽屏作为 `NavigationRail`/Drawer 的同级入口（rail 本身就是壳层 chrome）；账户管理只出现在“我的”上下文。
- `home_shell_metrics` 在宽屏无底栏时必须如实上报（空/0），Hero 落点不得按幽灵底栏裁剪。
- 路由参数只传稳定 id/query/page 等可序列化值；页面间跳转继续通过 `lib/app/navigation/routes.dart` 门面，避免 features 互相 import 页面。
- tab、搜索词、详情页码和必要滚动恢复使用 `restorationScopeId`/`PageStorageKey` 或路由参数；不得把 transient controller 状态写进 URL。

## 5. 响应式规则

以 `LayoutBuilder` 的可用宽度作为唯一输入，先算内容最大宽度与列数，再决定导航形态：

```text
< 600dp       compact：底栏，图库 2 列，操作栏可横向滚动
600–839dp     medium：rail 或紧凑 drawer，图库 3 列，详情双栏按约束启用
≥ 840dp       expanded：rail/drawer + 最大内容宽度，图库 4+ 列，详情媒体/信息分栏
```

具体列数由纯函数决定：把现有 private `_illustColumnsFor` 导出为按 `crossAxisExtent`（LayoutBuilder 约束，不是 `MediaQuery` 整窗宽——≥840 双栏时两者不同）计算的共享函数，并为最小卡片宽度、间距和最大内容宽度写单测。瀑布流只根据真实测量的图片尺寸布局，不允许通过改变内容高度来消除重叠。禁止按 `Platform.isAndroid`、机型名或固定屏幕尺寸分支布局。

## 6. 视觉验收与测试

每个迁移阶段维护同一套代表场景：

1. 亮/暗主题；中文、英文、日文、俄文。
2. 320、390、600、840、1200dp，另加手机横屏。
3. 内容、骨架加载、空结果、网络错误可重试、受限作品、长标题/长用户名、Ugoira。
4. 推荐→详情→作者→我的/收藏的连续导航及返回/恢复。

自动化层：组件 semantics/widget 测试、路由/状态恢复测试、响应式纯函数测试、golden/screenshot 测试。golden 不做维度全排列：**样板链路（推荐→详情→作者→我的/收藏）走完整矩阵，其余页面族只做代表状态矩阵**；需要先补 golden harness（泵装、字体/环境稳定化、CI 一致性——当前 `test/goldens/` 只有 `home_bar.png`）。人工层：真实设备触控/键盘/字体缩放/减少动效和页面视觉复核。报告分别记录，golden 失败不能被“更新快照”掩盖，设备未执行也不能填成通过。

## 7. 迁移顺序与兼容策略

先建立 tokens 和壳层，再完成样板链路，之后按页面族迁移：feed（推荐/排行/最新/搜索）→ 详情/Ugoira → profile/我的/收藏/关注/历史 → 小说/评论/下载/设置/登录引导。每个阶段保留旧路由可达性，使用小步提交；阶段末运行聚焦测试和 `flutter analyze --no-pub`，避免大批量重排与行为改动混在一起。

已有设置行为、网络策略、下载恢复、Ugoira 时间轴和阅读位置是高风险消费者。视觉重构只能改布局/文案呈现/入口归属；若测试显示业务状态变化，应停止该阶段并回退当前提交。

## 8. 关键风险与停止条件

- 主题 token 与 legacy/Material 组件主题隔离：若桥接后出现系统性颜色或字体错乱，停在主题阶段，先确定适配策略。
- shell branch 与现有恢复/分页 controller 生命周期冲突：若返回、滚动或分页三相语义变化，停止导航迁移，先补状态契约。
- 共享卡片抽象吞掉页面特例：若受限、动图或小说卡片需要大量布尔参数，保留命名变体或拆分组件，不制造万能组件。
- 截图矩阵膨胀：只保留能代表状态/断点的样例，新增样例必须说明覆盖的回归风险。
- 双套同义组件：`replica_*`/`feed/*` 与新建命名族并存导致永远收不齐——迁移遵循“扩展或重命名，不新增平行家族”，收尾时旧名不得残留。
- `me` 子路由移除的回归：任何还在 push `*/me` 的调用点会先撞到编译/路由测试；`openMe` 改 branch 切换后返回语义（tab 间返回 vs 栈内 pop）需在 `navigation_restoration_test` 覆盖。
- golden 环境敏感：跨平台/字体差异导致 golden 在本机与 CI 不一致时，先把 harness 固定（固定字体资产或一致环境），不允许用放宽阈值蒙混。
