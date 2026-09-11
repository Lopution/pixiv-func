# 技术设计：全项目 UI 视觉与交互重构

本设计只描述实现边界和可验证契约，不改变 Pixiv API、账户、下载、缓存或阅读状态的业务语义。PRD 中的全局视觉原则和样板链路已获用户确认。

## 1. 目标形态

```text
AppShell
├─ Primary destinations: 推荐 / 排行 / 最新 / 搜索 / 我的
├─ Shell action: 应用设置（/settings，与“我的”平级）
└─ Route content: feed → illust detail → user/my → secondary pages
```

窄屏使用 `NavigationBar`（或项目当前等价壳层），宽屏使用 `NavigationRail`/Drawer；五个目的地始终保持相同语义和顺序，导航形态随可用空间变化而不是随设备型号硬编码。`/settings` 不占用个人内容导航槽位，在窄屏由 shell 级齿轮/应用菜单打开，在宽屏作为 rail/drawer 的同级 action 显示。

## 2. 主题与 tokens

在 `lib/app/theme/` 建立 `FuncTokens` 及其 `ThemeExtension` 映射，组件只消费语义 token，不直接拼装颜色和间距：

| 类别 | 语义层 | 约束 |
| --- | --- | --- |
| Color | canvas、surface、surfaceRaised、surfaceOverlay、divider、contentPrimary/Secondary/Tertiary、brand、onBrand、danger/success/warning | 亮/暗主题成对定义；正文与交互文字通过对比度检查 |
| Type | display、title、body、label、caption、numeric | 建立字号/行高/字重阶梯；长标题允许自然换行 |
| Space | 2/4/8/12/16/24/32/48 | 仅保留有限尺度，页面不新增任意数字 |
| Shape | card、control、pill、sheet、dialog | 卡片与控件分别定义，避免所有元素同一圆角 |
| Motion | short、standard、emphasized、reducedMotion | reduced-motion 下不隐藏状态、不依赖动画完成 |

`ThemeData` 只负责将 tokens 投影到 Material 组件主题；Cupertino/原生承载页通过同一组语义值适配。现有品牌色若与可读性冲突，优先保证内容和状态对比度，品牌色只用于强调而非大面积背景。

## 3. 共享组件边界

组件放在 `lib/app/widgets/`，页面只组合，不复制近似实现：

- `IllustCard`/`IllustFeedGrid`：媒体、标题、作者、统计和快捷操作为可选槽位；卡片本身不 watch 整个 feed state。
- `MediaPreview`：统一缩略图比例、裁剪、占位、失败重试和受限标识；详情/Ugoira 通过命名变体表达差异。
- `AuthorSummary`、`ActionBar`、`TagChips`：明确 `Semantics`、tooltip、触控目标和异步状态。
- `FeedStates`：`FeedLoading`、`FeedEmpty`、`FeedError`、`FeedTail`，文案、图标和 action 通过参数注入；禁止页面私有 `_...Loading/_...Empty/_...Error` 新增。
- `AppScaffold`/`PageHeader`/`FilterBar`：统一安全区、滚动行为、标题层级和窄/宽屏排列。

作品卡片保留原始宽高比，元数据（作者、标题、统计、角标）使用固定层级和有限行数；不以统一固定高度换取“整齐”。详情页组合顺序固定为主视觉、作者/标题、主操作、标签/说明、评论/相关作品。作者头部只承载沉浸式视觉和核心身份，完整 tab/操作在其下方展开；搜索首屏的 `SearchBar`、筛选、历史和趋势是同一信息区，趋势点击发出显式 search intent。

共享组件的状态输入来自现有 Riverpod provider 或显式参数。组件不得创建 repository、HTTP client、偏好存储或平台 channel；异步操作由页面级 controller 持有，组件只报告 intent 并渲染 `idle/loading/success/failure`。

## 4. 路由与状态边界

- `go_router` 的 shell branch 对应五个一级目的地；每个 branch 保留独立导航栈和滚动位置。
- `/settings` 为 root-level route，设置子页继续使用 `/settings/...`，旧链接和返回语义保持兼容。
- `/me` 是“我的” branch 的入口；账户管理只在此上下文呈现。应用设置通过 shell action 进入，不能依赖 `MePage` 的账户菜单才能发现。
- 路由参数只传稳定 id/query/page 等可序列化值；页面间跳转继续通过 `lib/app/navigation/routes.dart` 门面，避免 features 互相 import 页面。
- tab、搜索词、详情页码和必要滚动恢复使用 `restorationScopeId`/`PageStorageKey` 或路由参数；不得把 transient controller 状态写进 URL。

## 5. 响应式规则

以 `LayoutBuilder` 的可用宽度作为唯一输入，先算内容最大宽度与列数，再决定导航形态：

```text
< 600dp       compact：底栏，图库 2 列，操作栏可横向滚动
600–839dp     medium：rail 或紧凑 drawer，图库 3 列，详情双栏按约束启用
≥ 840dp       expanded：rail/drawer + 最大内容宽度，图库 4+ 列，详情媒体/信息分栏
```

具体列数由 `illustColumnsFor(crossAxisExtent)` 这类纯函数决定，并为最小卡片宽度、间距和最大内容宽度写单测。瀑布流只根据真实测量的图片尺寸布局，不允许通过改变内容高度来消除重叠。禁止按 `Platform.isAndroid`、机型名或固定屏幕尺寸分支布局。

## 6. 视觉验收与测试

每个迁移阶段维护同一套代表场景：

1. 亮/暗主题；中文、英文、日文、俄文。
2. 320、390、600、840、1200dp，另加手机横屏。
3. 内容、骨架加载、空结果、网络错误可重试、受限作品、长标题/长用户名、Ugoira。
4. 推荐→详情→作者→我的/收藏的连续导航及返回/恢复。

自动化层：组件 semantics/widget 测试、路由/状态恢复测试、响应式纯函数测试、golden/screenshot 测试。人工层：真实设备触控/键盘/字体缩放/减少动效和页面视觉复核。报告分别记录，golden 失败不能被“更新快照”掩盖，设备未执行也不能填成通过。

## 7. 迁移顺序与兼容策略

先建立 tokens 和壳层，再完成样板链路，之后按页面族迁移：feed（推荐/排行/最新/搜索）→ 详情/Ugoira → profile/我的/收藏/关注/历史 → 小说/评论/下载/设置/登录引导。每个阶段保留旧路由可达性，使用小步提交；阶段末运行聚焦测试和 `flutter analyze --no-pub`，避免大批量重排与行为改动混在一起。

已有设置行为、网络策略、下载恢复、Ugoira 时间轴和阅读位置是高风险消费者。视觉重构只能改布局/文案呈现/入口归属；若测试显示业务状态变化，应停止该阶段并回退当前提交。

## 8. 关键风险与停止条件

- 主题 token 与 legacy/Material 组件主题隔离：若桥接后出现系统性颜色或字体错乱，停在主题阶段，先确定适配策略。
- shell branch 与现有恢复/分页 controller 生命周期冲突：若返回、滚动或分页三相语义变化，停止导航迁移，先补状态契约。
- 共享卡片抽象吞掉页面特例：若受限、动图或小说卡片需要大量布尔参数，保留命名变体或拆分组件，不制造万能组件。
- 截图矩阵膨胀：只保留能代表状态/断点的样例，新增样例必须说明覆盖的回归风险。
