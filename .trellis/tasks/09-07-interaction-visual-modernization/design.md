# 技术设计：交互与视觉现代化（child F）

本文件从 parent `prd.md` R10、parent `design.md` §13.3 和本 child `prd.md` 派生，并以 A–D 已归档后的
`origin/main` `973ca71` 为事实快照。C 已提供组件层、`routes.dart` 门面、`MotionTokens`、Hero clip 和
`PagedFeedController`；B 已提供 per-ABI 体积门禁；F 在这些 owner 上完成 UI 与导航迁移。

## 1. 当前状态与范围

| 项目 | A–D 后快照 |
|---|---|
| Flutter | 3.47.2 stable |
| UI imports | `package:flutter/material.dart` 84 个文件；`package:flutter/cupertino.dart` 4 个文件 |
| 路由 | `go_router` 17.5.0 已锁定但未使用；23 处 push；`ReplicaPageRoute` 27 个引用/13 个文件 |
| 图片依赖 | `cached_network_image` 3.4.1 |
| 首页 | 5 个根 tab：推荐、排行、最新、搜索、设置；C 的 lazy visited-tab + `IndexedStack` |
| 恢复 | 7 个 `PageStorageKey`；尚无 app/router restoration scope |
| 返回 | `HomePage`、`ProfileEditPage` 两处 `PopScope`；manifest 尚未启用 predictive back |
| golden | `test/goldens/home_bar.png` 1 份 |
| 体积门禁 | fdroid 实测 27,435,321 / 23,226,795 B；默认阈值 28,435,321 / 24,226,795 B；arm64 硬上限 32,000,000 B |

本 child 是 parent 中唯一允许改变用户可见视觉与导航模型的 child，但不改变 Pixiv 请求、账号、下载、历史、
Ugoira 解码或 updater 协议。第五个 shell 分支以当前 `homeShellTabs` 的 `SettingsPage` 为准；`MePage` 仍是设置
分支内的当前账号资料页。

F0 在真正开工时重新解析依赖并记录版本/API。目标仍是兼容 Flutter 3.47.2 的 `material_ui`、`cupertino_ui`、
`go_router >= 18`、`cached_network_image >= 4`；约束写入值以 `pub` 实际可解析组合和迁移 dry-run 为准。

## 2. `material_ui` / `cupertino_ui` 迁移

### 2.1 迁移面

- `lib/`、`test/` 和 app 自有工具代码统一使用新 UI 包导出的 widget/type；生成的 l10n 文件由
  `flutter gen-l10n` 重建，不手改。
- `AppLocalizations.localizationsDelegates` 当前聚合了 SDK Material/Cupertino delegate。F 改用显式列表：
  `AppLocalizations.delegate`、解析版本提供的 material/cupertino delegates、Flutter widgets delegate；不再使用
  生成类的聚合列表。
- `MaterialUiCompatibilityBridge` 只在 app builder 放一层，覆盖 F0 从依赖图和 package source 确认仍返回 SDK
  Material widget 的插件。F0 将插件名和触发页面写入 `research/modernization-recount.md`。
- `uses-material-design: true` 和现有 icon font 保持；`cupertino_icons` 不重新加入。

### 2.2 体积

F1 用 B 的同一命令构建 fdroid arm64/armeabi-v7a split，记录迁移前后 APK 文件字节和主要压缩桶。若迁移后的
实测超过当前默认阈值，同一提交把 `ci.yml`、`release.yml` 默认值调到实测 + 1,000,000 B；arm64
32,000,000 B 硬上限不变。F8 对最终 UI HEAD 再测一次并写回 `backend/release-artifacts.md` 与 parent 研究记录。

## 3. Material 3 主题

继续由 `lib/app/theme/replica_theme.dart` 单点产出 light/dark `ThemeData`，避免为改名扩散调用点。M3 token 映射：

| Theme 字段 | 来源/约定 |
|---|---|
| seed / primary / secondary | `FuncTokens.primary` |
| light background/surface/text | `lightBackground` / `lightSurface` / `lightText` |
| dark background/surface/text | `darkBackground` / `darkSurface` / `darkText` |
| subdued/onSurfaceVariant | 对应 brightness 的 `*Subdued`，由 ColorScheme 组合 |
| error/onError | `FuncTokens.error` / `FuncTokens.lightBackground` |
| AppBar | 0 elevation、透明 surface tint、现有标题字号与颜色 |
| NavigationBar/TabBar | primary indicator/selected，surface 背景，未选使用 onSurfaceVariant |
| Card/Chip/Dialog/BottomSheet | ColorScheme surface/container 与统一圆角；不在 feature 页面重复颜色 |
| SnackBar | 保持 floating，动画继续读 `MotionTokens` |

删除 `useMaterial3: false`，使用新 UI 包的默认 M3。`ReplicaSwitchTile` 内部从 `CupertinoSwitch` 改为 M3
`Switch`，对外构造参数保持不变。golden 在每个可见组件提交中只更新受影响快照。

## 4. Router 拓扑

### 4.1 owner 与 app 启动

- `lib/app/navigation/routes.dart` 继续是唯一允许集中 import feature page 的 app 文件，同时承载
  `GoRouter` factory、route path/codec 和现有 `open*` 门面。这样 C 的 layering 白名单不增加第二个环。
- `PixivFuncApp` 持有稳定的单个 router，改为 `MaterialApp.router`。settings 未完成加载时仍用 defaults 构建
  theme/locale，并由 `StartupGate` 显示进度或读取错误；router 不因 settings/account 更新而重建。
- `StartupGate` 从“直接返回目标 page”改为 app builder 的启动门：状态未就绪时覆盖 router child；就绪后引导到
  welcome/login 或显示已恢复的 route。登录成功通过 `go` 进入推荐分支。
- app builder 从外到内组合 UI compatibility bridge、startup gate、全局 intent bridge 和 router child，各 owner
  只保留一份。

### 4.2 Navigator 与路径表

`StatefulShellRoute.indexedStack` 使用 5 个 `StatefulShellBranch`，每个独立 navigator key、observer 和
restoration scope，`preload: false`。默认实现只在分支首次访问时创建 Navigator，保留 C 的“冷启动只构建当前
tab”约束。

| Navigator | 路径 | 页面/用途 |
|---|---|---|
| root | `/welcome` | `WelcomePage` |
| root | `/welcome/language` | `LanguagePage` |
| root | `/welcome/theme` | `ThemePage` |
| root | `/login` | `LoginPage`；`first`/`return` 为 query |
| root | `/login/web` | `LoginWebViewPage`；`create` 为 query，OAuth service 从 provider 读取 |
| root | `/login/callback` | 活跃 PKCE session 的一次性 callback |
| root | `/reverse-image` | `ReverseImageSearchPage`；分享图片输入走 route extra |
| shell branch 0 | `/recommended` | `RecommendedHomePage` |
| shell branch 1 | `/ranking?mode=<name>` | `RankingPage` |
| shell branch 2 | `/new` | `NewPage` |
| shell branch 3 | `/search` | `SearchHomePage` |
| shell branch 4 | `/settings` | `SettingsPage`（当前第五个 tab） |
| 当前 branch | `/<branch>/illust/:illustId` | `IllustDetailPage` |
| root overlay | `/<branch>/illust/:illustId/viewer/:page?quality=<code>` | `ImageViewerPage`，隐藏 shell chrome |
| 当前 branch | `/<branch>/illust/:illustId/comments` | `IllustCommentsPage` |
| 当前 branch | `/<branch>/illust/:illustId/comments/:rootCommentId` | `CommentRepliesPage` |
| 当前 branch | `/<branch>/user/:userId` | `UserPage` |
| 当前 branch | `/<branch>/me`、`/<branch>/profile/:userId/edit` | `MePage`、`ProfileEditPage` |
| 当前 branch | `/<branch>/novel/:novelId` | `NovelPage` |
| 当前 branch | `/<branch>/history` | `HistoryPage` |
| search branch | `/search/input?q=<text>&type=<name>` | `SearchInputPage` |
| search branch | `/search/results?...` | `SearchResultPage`，完整 query 见 §5 |
| search branch | `/search/tag/:keyword` | `TagSearchPage` |
| settings branch | `/settings/account`、`theme`、`language`、`translate` | 对应 settings pages |
| settings branch | `/settings/translate/credentials/:provider` | `TranslationCredentialsPage` |
| settings branch | `/settings/network`、`network/probe`、`network/advanced` | 网络设置与 probe |
| settings branch | `/settings/browse`、`download`、`download/destination` | 浏览/下载设置 |
| settings branch | `/settings/history`、`blocked`、`tasks`、`about`、`about/licenses` | 其余设置与 `LicensePage` |

`<branch>` 是当前 shell 根名（`recommended|ranking|new|search|settings`）。共享 detail routes 由一个私有 route
factory 为五个 branch 生成，因此从哪个 tab 打开详情就压入哪个 navigator；切换 tab 后该栈仍保留。外部 illust/user
深链统一进入 recommended branch，形成唯一 canonical 落点。

viewer、登录 WebView、反查页使用 root navigator；dialog 和 bottom sheet 仍是所属页面的临时 overlay，不进入 URL。

### 4.3 转场与 route observer

- branch 内 page 使用一个 `CustomTransitionPage` builder，读取 `MotionTokens.pageTransition` /
  `pageCurve`，保持 C 的 300ms 右进右出节奏。分支切换由 indexed stack 完成，不叠加 page transition。
- `routes.dart` 的 `openIllust/openUser/openNovel/openSearchResults/...` 改为 `context.push/go/replace`，调用方仍传
  id/typed query；settings 的 `openSettingsPage(context, Widget)` 改为明确的 path facade。
- root 与五个 branch 各有自己的 `RouteObserver<ModalRoute<dynamic>>`。route page builder 用
  `RouteObserverScope` 暴露所属 observer，`HistoryVisibility` 订阅最近的一份，避免把一个 observer 同时挂到多个
  Navigator。`HomePage` 不再自行订阅旧全局 observer。
- `homeShellMetricsProvider` 继续由 shell 测量真实 `NavigationBar` top/height，Hero clip 不使用固定高度。

## 5. Route 数据与恢复边界

稳定标量写入 path/query；已经在 store 中的实体可作为 route extra 加速首帧，但 route page 在 extra 缺失时仍按 ID
读取现有 controller/repository。

| 页面 | path/query 中的 durable 数据 | route extra |
|---|---|---|
| illust detail | `illustId` | 可选 `IllustEntity`、feed Hero scope/image URL |
| image viewer | `illustId`、`page`、`ViewQuality.code` | 可选 entity 与 source Hero tag；缺失时从 `IllustStore`/detail controller 取得 URL |
| user/novel/profile edit | 对应 numeric id | 无 |
| comment replies | `illustId`、`rootCommentId` | 可选 `CommentEntity`；缺失时由 comment store/feed 取得 root |
| search input | `q`、`SearchResultType.name` | 无 |
| search results | `type`、`q`、`target`、`sort`、`duration`、`start`、`end` | 无；编码/解码回到现有 `SearchQuery/SearchFilters` |
| ranking | `RankingMode.name` | 无 |
| reverse image | route 只标识页面 | `ReverseImageInputReference`（ACTION_SEND 的 content URI） |
| OAuth callback | route 只标识 callback | `AccountCallbackRoute`，交给当前 `OAuthService` session |

`MePage` 的 edit 动作改为调用 `openProfileEdit`，不再把 `VoidCallback` 当 route 数据。分享图片 URI 和 OAuth code/state
不写入可恢复 URL：reverse-image page 可恢复为空的选择页，callback page 本身不参与 restoration，登录进程重建后重新发起。

## 6. 状态恢复

恢复分两层，均使用 Flutter/go_router 已有机制：

1. `MaterialApp.router(restorationScopeId: 'pixiv-func')`、`GoRouter(restorationScopeId: 'router')`、shell 与五个
   branch 的稳定 scope id，恢复当前 tab 和各分支 route stack。
2. 所有 feed 列表给 ScrollView 同时设置稳定 `PageStorageKey` 与 `restorationId`。前者保持同一进程中的 tab/
   mode 切换，后者使用 `Scrollable` 自带 restoration bucket 恢复进程重建后的 offset。

动态 key 按页面已有 identity 构造：ranking mode、`NewFeedKey`、search query cache key、profile tab、history scope。
`SearchInputPage` 的输入框使用 `TextField.restorationId`；已提交的搜索条件完整进入 route query。`RankingPage`
切换 mode 时 replace 当前 query；viewer 翻页时 replace `:page`，不新增另一份 current-page owner。

widget 测试用 `RestorationTester`/`restartAndRestore()` 覆盖 tab、嵌套路由、搜索词、viewer 页码和一个代表性 feed offset；
Android 真机再用“不保留活动”与 `am kill` 验证同 tab、同 route、同滚动区间。

## 7. 返回模型与 Predictive Back

- Android main manifest 在 `<application>` 开启 `android:enableOnBackInvokedCallback="true"`。
- go_router 先 pop 当前 branch 栈；只有 branch 已在根页面时，shell 的 `RootBackCoordinator` 才处理两次返回退出。
  切换 branch 或 push route 会解除已显示的退出窗口。
- `ProfileEditPage` 保留未保存确认，使用 `PopScope.onPopInvokedWithResult`；app route 不新增 `WillPopScope`。
- Android 14+ 验证系统手势预览与完成/cancel；API 29 验证返回顺序和两次返回退出不变。

## 8. 拖拽关闭与 Hero

新增 `lib/app/motion/drag_to_dismiss.dart`，由 still viewer 和 Ugoira image surface 共用：

- 只在 1x viewer 接受向下手势；图片已缩放时 `InteractiveViewer` 继续拥有 pan，水平拖动继续翻页。
- 拖动进度只控制当前 surface 的纵向位移、缩放和背景透明度；未达到距离/速度阈值时按
  `MotionTokens.fast` 回位。
- 达到阈值后调用当前 route 的 `pop`，由同一 `CustomTransitionPage` 和匹配的 Hero tag 完成反向 flight；
  `HeroRectClip` 仍负责 shell/app bar 遮挡边界。
- Ugoira detail surface 使用同一 wrapper；正常 tap 播放、长按下载与生命周期释放保持原 owner。

测试覆盖取消回位、成功 pop、zoomed viewer 不触发 dismiss，以及 Ugoira route pop；现有 Hero 几何、viewer zoom/page、
Ugoira playback tests 继续通过。

## 9. M3 组件替换

| 当前实现 | 目标 | 状态/owner |
|---|---|---|
| 首页 `BottomAppBar + TabBar` | `NavigationBar` + 5 个 `NavigationDestination` | selected index 来自 shell；增加四语言 tab label；继续发布实际 bar metrics |
| 搜索 guide box / AppBar `TextField` | `SearchBar` / `SearchAnchor` | suggestion 数据仍由 `searchAutocompleteProvider`，submit 仍走 typed `SearchQuery` |
| Browse settings 的三组 `_qualityTile(Object, Object)` | 三个 typed `SegmentedButton<T>` | Preview/Detail/ViewQuality 各自调用现有 settings notifier |
| `ReplicaSwitchTile` 的 `CupertinoSwitch` | M3 `Switch` | 对外 API 不变 |
| feature 自定义 Dialog/BottomSheet 样式 | 主题集中配置 | 业务内容和返回值不变 |

`FollowSwitchButton` 已使用 `SegmentedButton<FollowRestrict>`，F 只让它继承新主题；bookmark restriction 的 dropdown
不是本次 quality selector，保持现有交互。

## 10. 回滚点与完成边界

- F1 是依赖/import/compatibility bridge 回滚点；主题或本地化在 legacy 插件 subtree 中不可用时停在该提交处理依赖组合。
- F3 router 是导航回滚点；lazy branch 请求、独立栈、HistoryVisibility 或 root back 任一契约不成立时先修该提交。
- F4 restoration、F5 predictive back、F6 drag、每个 M3 component 各自独立提交。
- `PagedFeedController` loading/loaded/error+tail、Pull-to-Refresh、Hero clip、下载/history/updater 持久化格式保持原 owner。
- F 不增加 material_ui/go_router 之外的 UI/路由库，也不增加平板专用页面结构。
