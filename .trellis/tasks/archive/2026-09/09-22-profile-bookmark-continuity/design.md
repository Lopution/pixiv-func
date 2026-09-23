# 技术设计：作者页与收藏范围连续（W3）

设计基线：`main@8067b2d`；start 前必须 rebaseline，行号以 rebaseline 后为准。
逐文件改动方案见 `research/implementation-draft.md`（§A–J 与本文件 R 序号对应：
A→R1、B→R2、C→R3、D→R4、E→R5、F→R6、G→R7、H→R8、I→W1 消费点、J→门禁）；
逐项现状核实见 `research/codebase-*.md`；决策定案见 `prd.md` D1–D13
（其中 D4 修正了 `research/risks.md` #4 的失实断言——`UserEntity` 已有
`totalIllustSeries`/`totalNovelSeries`，`user_entity.dart` L29-30）。

## 一、阶段划分与依赖

### 启动门禁（不满足不 start）

- **W1 已合入 `main`**：`profile_edit_page.dart` `_attemptPop` 干净表单短路
  已存在、`profile_edit_test.dart` L415 断言已随 W1 改写。未合入 → 本包不
  `task.py start`；rebaseline 时复核。
- **Rebaseline**：记录当时 main SHA；复核 `research/` 行号断言；跑聚焦基线
  （`user_profile_test` / `profile_edit_test` / `bookmark_tags_test` /
  `bookmark_switch_button_test`）。
- **W2 通道状态**：re-tap 广播由 W2 在 `BranchSlidePager.selectIndex`
  same-index 路径创建（`branch_slide_stack.dart` L140-143 区域）。该广播只
  覆盖 branch root；作者页（`user/:userId` 在 commonBranchRoutes、`/me` 在
  root navigator）均为 pushed route，**不可达**该广播——本包 re-tap 全部为
  页面内 same-index 入口，不依赖 W2 产物、可先于 W2 落地；W2 若先合入，
  对齐其语义约定（纯滚动、MotionTokens、reduced-motion jumpTo），若其沉淀
  了页面内 re-tap helper 则优先复用，不自建第二通道。

### 阶段表

默认单分支 `task/09-22-profile-bookmark-continuity`；若单 PR 过大按 stage 拆
`…-s1`/`…-s2`（s1 = 阶段 1+2，s2 = 阶段 3+4+5）。

| 阶段 | 条目 | 主要文件 | 风险 |
|---|---|---|---|
| 1 | R1 等价路径 / R2 chips 常驻+re-tap | `profile_header_delegate.dart`、`user_page.dart`、`follow_switch_button.dart` | 中（crossfade 测试面、NestedScrollView 双滚动） |
| 2 | R3 统计 / R4 链接 / R5 分享 | `user_page.dart`、`profile_header_delegate.dart` | 低 |
| 3 | R6 范围/过滤/分页 | `routes.dart`、`bookmark_tags_page.dart`、`bookmark_tag_feed_page.dart`、`profile_illust_feed.dart` | 中（routes.dart 与 W2 共享，高冲突区） |
| 4 | R7 资料编辑布局 | `profile_edit_page.dart` | 中（消费 W1 契约） |
| 5 | R8 弹层迁移 | `bookmark_switch_button.dart`、`follow_switch_button.dart` | 中（store 交互、dirty 拦截） |

owning files 合计：`lib/features/profile/{user_page,profile_header_delegate,
profile_edit_page,profile_illust_feed,bookmark_tag_feed_page}.dart`、
`lib/features/bookmark/bookmark_tags_page.dart`、
`lib/app/widgets/{bookmark_switch_button,follow_switch_button}.dart`、
`lib/app/navigation/routes.dart`（门面+路由 query——**与 W2 共享，按合入顺序
串行**）、`lib/l10n/`（arb+lookup，全 leaf 高冲突区，D13）及对应测试文件。

不改的相邻文件：`bookmark_tags_controller.dart`（`loadMoreError`/重试能力已
具备，只读不写）、`author_works_download_dialog.dart`（已是 action 型）、
`app_overlays.dart`/`app_breakpoints.dart`（纯消费）、`branch_slide_stack.dart`
（W2 所有）。

## 二、契约落点

### QueryContext（父 §5.1 在本 leaf 的落点）

作者页与收藏链路的可见上下文与恢复等级声明：

| 上下文 | 载体 | 恢复等级 |
|---|---|---|
| 作者页 tab / work section | `_tabController` / `_workSection`（页面 State） | 内存 |
| 作者页收藏/关注范围 | `_restrict`（页面 State）→ feedKey | 内存 |
| 收藏标签页范围 | route `?restrict=`（**本包新增 durable**） | 路由 durable（含进程死亡） |
| 标签 feed tag+范围 | route `?tag=&restrict=`（已有） | 路由 durable（含进程死亡） |
| 标签 feed 过滤词 | 页面 State | 内存（声明） |
| 滚动位置 | `PageStorageKey(feedKey)` / `restorationId` | 内存 + Flutter restoration |
| re-tap | same-index → 页面内 `_scrollActiveTabToTop` | 行为契约（纯滚动，不带刷新） |

loading/error/empty 时上下文保留：header、tab、chips、范围标识独立于 feed
状态渲染（现状天然满足，回归测试钉住即可）。

### FormState（父 §5.3 落点）

| 表单 | 类型 | 说明 |
|---|---|---|
| ProfileEditPage | draft | 已有完整 draft 模型（`ProfileDraft.hasChanges`/`buildPatch`）；本包只动布局不动判定 |
| _BookmarkEditSheet | draft | 初始值基线 = 回填值/public+空 tags；dirty 离开确认；**confirm 失败保留草稿**（D6：await + 读 store entry.error） |
| follow sheet | action | 一次选择+提交；只对齐控件/间距 |
| AuthorWorksDownloadDialog | action | 保持现状（phase 状态机 + CancelToken 已合规） |

### BackAndCancel（父 §5.4 落点）

- 收藏 sheet 的「取消」与下滑关闭：dirty 时先弹 `showAppDialog` 确认（与 W1
  统一确认层级同形）；confirm 提交成功后的 pop 不被拦截。
- ProfileEditPage 显式返回 = W1 契约（干净直接 pop、脏才确认）；本包移动
  保存按钮/改布局时保持 `PopScope` 与测试断言不被绕开。

### ResponsiveContent（父 §5.5 落点）

- 资料编辑 form：expanded 断点（1200 起）列宽 ≤`ContentWidths.form`（520）
  居中——断点读 `AppBreakpoints`、内容宽度读 `lib/app/layout/content_widths.dart`
  角色常量（父 §5.5 角色表），不自造宽度常量。若该文件尚未存在，
  本包为首个消费者时顺手创建（全表常量一并写入）。
- 收藏 sheet：expanded 断点 builder 内 `ConstrainedBox` 限宽居中；dialog 由
  `showAppDialog` 天然限宽——入口均已合规，本包只做内容侧限宽。
- 门禁覆盖：320/390/600/840/1200dp 五档主操作可达（测试+运行时矩阵）。

### FeedbackChannels / ActionTerminology（父 §5.6/§5.7 落点）

- 全部操作反馈走 `showAppSnackBar`（不新增平行通道）；本包不消费触觉
  （共享触觉 owner 由 W4 随下载/保存创建）。
- 术语：复制链接=`copyLink`（已有）；打开链接=新 key `openLink`；已关注态
  菜单动作=`unfollow`（新 key，后果动词）；私密关注=`followPrivately`
  （新 key）；「取消」只终止当前操作（sheet/对话框内），无改动返回不叫取消。

### Tab / Hero / Route 契约（component-guidelines 落点）

- TabBar.onTap 回调内不调 `animateTo`（same-index re-tap 只滚动）；统计导航
  的 `animateTo` 来自非 TabBar 回调来源，允许。
- `ProfileIllustFeed` heroScope 追加 `bookmarkTag` 段（D9），跨 surface
  Hero 冲突归零。
- 路由数据走 typed 门面 + durable query（`?restrict=`），不走 untyped extra。

## 三、复用与禁止

**复用**：`PullToRefresh`/`FeedTail`/`FeedError`（feed_states）、
`SegmentedButton`/`OutlinedButton`/`FilledButton`、
`showAppDialog`/`showAppBottomSheet`、`outboundUrlOpenerProvider`、
`shareServiceProvider`/`shareOriginOf`、`PersonAvatar`、
`followActionsProvider`/`FollowStore`、`bookmarkActionsProvider`/`BookmarkStore`
（`addWithRestrict` 已是 `Future`，可直接 await 后读 entry.error）、
`AppBreakpoints`、`MotionTokens`、`UserBookmarkTagsState.loadMoreError`、
现有 l10n 工具链。

**禁止**：

- 新 provider/controller/repository、新路由注册（只给现有 `bookmarks/tags`
  路由加 query）、`lib/app/` 公共基件（统计控件、动作清单均为页面内私有
  widget——单消费者不满足共享阈值）；
- 第二个 re-tap 广播/信号机制（`BranchSlidePager` 归 W2）；
- feed controller 侧的过滤参数下推（本地过滤只在 widget 渲染层作用于已加载
  entities，不进入 `ProfileFeedKey`/provider family——否则 cacheKey 分叉）；
- 网络/数据层语义、`ResumeAnchor` 式滥用、三个平行枚举的合并；
- 为「无承载 tab 的统计项」造新页面（他人页 myPixiv 纯展示，D4）。

## 四、测试策略

widget 测试为主 + 少量纯断言；遵循 quality-guidelines（material_ui import、
snackbar 走 `showAppSnackBar` 通道、回归测试需能对旧实现失败、断言终态）。

- `test/user_profile_test.dart`：crossfade/collapsed 组随挂载阈值改预期
  （L358-456、L498、L594、L650）；新增——半收起区间无 collapsed chrome、
  两态动作清单等价（isMe/!isMe 各跑）、收起菜单 follow 项随 follow 态换文案
  +toggle、私密关注项弹同一 sheet、chips 常驻、re-tap 双路径回顶（outer+inner
  offset==0、reduced-motion jumpTo）、统计点击断言 tab index+`_workSection`、
  他人页 myPixiv 无 button 语义、分享直达（无 dialog、share 被调）、
  复制链接菜单项、tab 1.3x/长翻译 overflow。
- `test/profile_edit_test.dart`：保存按钮任意滚动位置可见、W1 dirty/clean
  断言保持、宽屏限宽约束断言、头像圆/背景宽条预览 finder。
- `test/bookmark_tags_test.dart`：`initialRestrict` 生效、tail 三态
  （error 行渲染+retry 触发 loadMore、hasMore=false 无 spinner）、点 tag
  传当前 restrict。
- `test/bookmark_tag_feed_test.dart`（新）：AppBar 范围副标题、本地过滤
  （已加载页过滤/清空恢复/无匹配提示/不触发 loadMore）、heroScope 含
  bookmarkTag 段。
- `test/bookmark_switch_button_test.dart`：SegmentedButton/Outlined/Filled
  finder、dirty 取消弹确认/干净直关、confirm 失败 sheet 保持+内联错误+
  草稿保留（override repository 抛非连接错误）、成功 pop、tag 残文并入。
- 路由解析：`bookmarks/tags?restrict=private` → `BookmarkTagsPage` 初始
  private（routes 门面测试惯例）。

## 五、运行时证据缺口（PR 标「未验证」）

- NestedScrollView 内层 `PrimaryScrollController` 回顶的真机手感、re-tap 与
  下拉刷新/滚动手势竞争；
- 收藏 sheet 键盘顶起 + 0.75h 上限、宽屏 sheet 实际限宽（840/1200dp 桌面）；
- reduced-motion 下 `showAppBottomSheet` noAnimation 路径与回顶 `jumpTo`；
- 1.3x 大字号 + 长翻译 tab 标签可读性、header 统计 chip 行溢出；
- TalkBack/Windows Narrator 代表路径：统计 chip「关注 · 123」单焦点读出、
  collapsed 菜单焦点顺序；
- iPad share popover 锚点（`shareOriginOf` 取触发按钮 context 后）；
- 资料编辑长表单 + 键盘 + 底栏交互（resizeToAvoidBottomInset 默认保留的
  实际行为）。
