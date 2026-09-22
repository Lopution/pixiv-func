# 作者页与收藏范围连续（Roadmap W3）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图）。
规划基线：`main@8067b2d`。现状断言见本 leaf `research/codebase-*.md`（已对该基线
逐条核实），逐文件改动方案见 `research/implementation-draft.md`，决策点见
`research/risks.md`（本文件 Decisions 已全部定案；其中 risks #4「UserEntity 无
series 计数字段」经复核失实——`totalIllustSeries`/`totalNovelSeries` 已存在于
`lib/core/user/user_entity.dart` L29-30，按 D3 处理）。

## Goal

打通「作者页 → 收藏标签页 → 标签 feed」的范围与上下文连续性，让作者页头部在
展开/收起两态提供同一组动作的等价路径，并把资料编辑、收藏编辑两个表单收敛到
§5.3/§5.5 语义。本 leaf 只做页面行为与弹层语义：不改网络/数据层契约、不预建
其他工作包才会使用的公共基件（父 design.md §4.11）、不重写 W1 已落地的返回
判定规则。

前置（门禁）：**W1（09-22-interaction-outcome-correctness）必须先合入 `main`**
——本包 R7 在其 `_attemptPop` 干净短路契约上做布局。W1 未合入时本包不
`task.py start`；rebaseline 时复核该契约与全部行号。

## Requirements

### R1. 展开/收起等价路径（Astra #8）

现状：`profile_header_delegate.dart` 收起工具栏由
`IgnorePointer(ignoring: !isFullyCollapsed)`（L220-223）包住，
`collapsedOpacity>0` 但未完全收起时菜单**可见不可点**；follow 只在展开态
（L496-501），/me 的 share 只在展开态（收起菜单 `if (!isMe)` L568-572），
restrict/bookmarkTags/downloadAll 只在收起态菜单（L539-563）。

- 两态动作由同一动作描述清单生成：展开态主行动作内联、其余进「更多」溢出；
  收起态全部进溢出菜单。任何状态下不出现「一态有路径、另一态完全无路径」的
  动作。
- 收起态溢出菜单补：他人页 关注/取消关注 + 私密关注项（未关注时）、
  /me 的 share 项；展开态溢出承接 restrict/bookmarkTags/downloadAll 等条件项。
- 「可见即可点」：collapsed chrome 仅在 `isFullyCollapsed` 时挂载（D8），
  fade-in 区间不再出现可见不可点窗口；`_HeaderBackButton`（L632-657）常驻不变。

### R2. 作品类型持续可见 + re-tap 回顶（Astra #9 前半 / §5.1）

现状：`_onTabTap` 重击 work tab 切 `_selectorExpanded`（user_page.dart
L143-150）；section chips 仅 `expanded && isWorkTab` 时占 64dp
（profile_header_delegate.dart L723、L821-849）。

- work tab 激活时 64dp section chip 行常驻（`ReplicaProfileTabsDelegate.maxExtent`
  对 work tab 恒 +64）；删除 `_selectorExpanded` 展开语义。
- 重复点击当前 tab、或重击已选中的 section chip = 回到顶部：外层
  `NestedScrollView` controller `animateTo(0)` 展开 header，内层列表经
  `PrimaryScrollController` `animateTo(0)` 回顶——两步都做（PixEz 只动外层，
  列表停深处，语义不完整）。纯滚动：不带刷新、不承载任何其他入口；
  reduced-motion 下 `jumpTo(0)`。
- 非 work tab 的 same-index re-tap 同样回顶（§5.1 不分 tab）。
- /me 与他人页共用同一导航结构；本包不新增/删除 tab（§4.3「更紧凑」为允许项，
  不执行，见 D12）。

### R3. 可导航统计带标签（Astra #9）

现状：展开态 `_Stat`×3 纯展示（delegate L456-474、L675-699）；about tab
`_ProfileStatRow`×5 不导航（user_page.dart L457-481）。

- 统计集 = following / myPixiv / illust / manga / novel / series 六项（series 用
  `totalIllustSeries`，字段已存在）；控件 = 图标+数值+标签，
  `Semantics(button:)` 一次焦点读出「关注 · 123」。
- 点击 = 导航：following → following tab；myPixiv → /me myPixiv tab（他人页
  无承载 tab，该项纯展示不标 button 语义，D4）；illust/manga/novel/series →
  work tab + 对应 section（`_onSectionChanged` + `animateTo`——非 TabBar 回调
  来源的程序化切换允许 animateTo）。
- header chip 行与 about 行共用同一导航映射；/me 无 about tab，header 统计是
  其唯一 totals 入口，六项全部可导航。

### R4. 社交链接：主操作打开、次操作复制（Astra #9）

现状：about tab 三个 URL 字段为 `SelectableText`（user_page.dart L426-429）。

- webpage/twitterUrl/pawooUrl 行：主操作 tap=打开
  （`outboundUrlOpenerProvider.openExternal`，已含 http(s) 校验与抛错），
  次操作=复制（`Clipboard.setData` + `linkCopied` snackbar）；两个动作都是
  可见控件，鼠标/键盘可达，不把复制藏进长按。
- 打开失败 snackbar 复用 `illustDetailOpenLinkFailed` 模式
  （caption_rich_text.dart L46-55）。
- comment/id/account/简介行保持 `SelectableText`（`user.comment` 为纯文本字段；
  实现期确认是否含 HTML，若含再评估 `CaptionRichText`，默认不换）。

### R5. 分享默认路径与作品一致（Astra #9）

现状：`_showProfileShare` 开 AlertDialog 二选一（user_page.dart L541-581）。

- 主操作直接 `shareServiceProvider.share(SharePayload.user(...))`，
  `copiedToClipboard` 回退走 `linkCopied` snackbar——与作品分享
  （illust_detail_page.dart L204-222）同形；「复制链接」降级为溢出菜单独立项
  （`copyLink` l10n 已有）。
- 删除整个分享对话框；`shareOriginOf(context)` 取触发按钮 context 以正确定位
  iPad popover（当前传页面 context=整屏 rect，需修正）。

### R6. 公开/私密范围传递 + 本地过滤 + 分页状态（Astra #10）

现状：`openBookmarkTags` 无参（routes.dart L1209-1211）、`bookmarks/tags` 路由
无 query（L585-589）、`BookmarkTagsPage._restrict` 硬编码 public
（bookmark_tags_page.dart L23）——作者页当前范围在「作者页 → 标签页」一跳断掉；
标签 feed 顶部只有标签名（bookmark_tag_feed_page.dart L32-46），无范围可见性、
无本地过滤；标签列表 `loadMoreError` 存了但从不渲染（无读取点）；feedKey
heroScope 不含 `bookmarkTag`（profile_illust_feed.dart L101-104）。

- /me 收藏 tab 当前 `_restrict`（UserRestrict）经
  `openBookmarkTags(context, {required restrict})` → 路由 `?restrict=` →
  `BookmarkTagsPage(initialRestrict:)`；跨枚举映射收敛在 routes 门面（D7）。
- 标签页内 SegmentedButton 切换保留；点 tag 把**当前选中** restrict 传给
  `openBookmarkTagFeed`（现状已如此，L132-133）。
- `BookmarkTagFeedPage` AppBar 显示 tag + 范围副标题/标识；不做页内切范围
  （返回标签页切换即可，范围是 durable query，状态不丢）。
- **本地过滤语义写死**：标签 feed 提供关键字过滤，只作用于**已加载页**的
  标题/标签（contains、大小写不敏感）；无服务端搜索接口、不做持续拉取补偿；
  过滤词清空恢复完整已加载列表；过滤状态为页面内存态（D5）。
- 标签列表分页尾部三态齐全：可加载/加载中 spinner/失败行+重试
  （`UserBookmarkTagsState.loadMoreError` 渲染出来，重试调 `loadMore()`）。
- `ProfileIllustFeed` heroScope 追加 `bookmarkTag` 维度，杜绝同一作品 ID 在
  /me 收藏流与标签流同挂载时的跨 surface Hero 冲突（D9）。

### R7. 资料编辑固定保存入口与真实预览（Astra #11 后半）

前提：W1 已把 `_attemptPop`（profile_edit_page.dart L137-167）改为干净表单
直接 pop；本包不动判定与 `PopScope`（L175-179），只在其契约上做布局。

现状：保存 `FilledButton.icon` 在 ListView 末尾（L424-433），长表单需滚到底；
`_ImageField` 头像/背景共用 54×54 方形预览（L459-508）；整页 ListView 全宽
（L321-322）无宽屏限宽。

- 保存入口固定：`Scaffold.bottomNavigationBar`（SafeArea 内，宽屏与表单同列
  限宽居中）；submitting/confirmed 禁用逻辑（L316-319）原样保留；
  `requiresCurrentPassword` 密码框仍在表单内。
- 预览形态拆分：头像 = `PersonAvatar` 圆形（+ring），背景 = 宽条
  `PixivImage.detail` cover 预览（与 header 背景同形态）；已选文件尺寸
  subtitle 保留。
- 宽屏限宽：expanded 断点（`AppBreakpoints`，1200 起）表单列宽 ≤600-720 居中，
  `LayoutBuilder`+`Center`+`ConstrainedBox` 实现，不新造断点常量。
- 键盘：`resizeToAvoidBottomInset` 保留默认（页面无 autofocus，底栏随键盘上浮
  保持保存可达），长表单+键盘行为列入运行时验证（D11）。

### R8. 收藏/关注编辑弹层 FormState 迁移（弹层条目）

现状：本域弹层入口已是 `showAppBottomSheet`/`showAppDialog`
（codebase-overlays-forms.md §1），**无入口替换工作**；差距在控件与语义：
收藏 sheet 范围选择用 `_RestrictSelect` DropdownButton 药丸
（bookmark_switch_button.dart L470-535），而 follow sheet 与标签页用
SegmentedButton；取消/确认用 MaterialButton+headlineSmall（L391-462）；
取消直接 pop 不查 dirty（L415）；确认 pop 后 fire-and-forget 提交
（L433-444），失败时草稿已弃。

- `_RestrictSelect` → `SegmentedButton<BookmarkRestrict>`；取消/确认 →
  `OutlinedButton`/`FilledButton`；回填到达前 confirm 禁用（L431-432）保留。
- FormState=draft：初始值 = 已收藏回填值 / 新建 public+空 tags；tags/restrict
  与初始值不同即 dirty；取消与下滑关闭在 dirty 时先弹 `showAppDialog` 确认
  （与 W1 统一确认层级同形，复用 profileEditLeave* 文案模式或新 key）。
- 确认 → 提交：**失败保留草稿**（D6）；确认时把 `_tagInput` 未提交残文并入
  tags 的现状保留（L438-443）。
- 宽屏：expanded 断点下 sheet 内容限宽居中（builder 内 ConstrainedBox，不改
  `showAppBottomSheet` 签名）。
- follow sheet（follow_switch_button.dart L36-114）已是 SegmentedButton +
  Outlined/FilledButton——只对齐间距/标题层级，不重写。
- `AuthorWorksDownloadDialog` 已是 action 型 FormState（phase 状态机+
  CancelToken），行为保持，仅复核。

## Acceptance Criteria

- [ ] 每项 R 有 ≥1 个 focused test（新增或更新）；既有受影响测试同步修正——
      `user_profile_test.dart` crossfade/collapsed 组（L358-456、L498、L594、
      L650）、`profile_edit_test.dart` L415 dirty 断言随 W1 契约走。
- [ ] 父 implement.md §6 W2/W3 门禁逐项有证据：
      - 跳转前后范围一致：`bookmarks/tags?restrict=` 路由参数测试 +
        /me 私密范围→标签页初始 private 用例；
      - header/tab/动作在 feed loading/error/empty 下仍可见（回归）；
      - tab 标签不靠无限缩字维持单行（0.55 有界下限 + 1.3x/长翻译 overflow
        测试）；
      - 作者 header 点击/滑动规则回归测试；
      - re-tap 固定为回到顶部、不再承载展开入口；
      - 资料/收藏弹层 FormState：draft dirty 确认、失败草稿保留、
        320/390/600/840/1200dp 主操作可达。
- [ ] re-tap 双路径测试：tab same-index 与 section chip same-index 均断言
      outer+inner 滚动位置归零；reduced-motion 走 `jumpTo`。
- [ ] heroScope：bookmarkTag 为 null 与非空的两 feedKey 产生不同 tag 断言。
- [ ] 收藏 sheet：dirty 取消/下滑弹确认、confirm 失败 sheet 保持+草稿保留+
      内联错误、成功/离线排队 pop。
- [ ] 恢复等级声明（design.md 表格）：收藏范围 = 路由 durable（本 PR 新增），
      tab/section/scroll = 内存 + Flutter restoration，本地过滤 = 内存态。
- [ ] 每 commit 对应 implement.md 一个勾选框；分支
      `task/09-22-profile-bookmark-continuity`。
- [ ] `flutter analyze --no-pub`、相关 `flutter test`、`git diff --check`、
      `task.py validate` 全绿；运行时矩阵中本机不可验项在 PR 显式标「未验证」。

## Decisions（规划定案）

- **D1 收起态 follow = 溢出菜单项**：FollowSwitchButton（96/116dp）塞不进
  48dp toolbar slot；菜单项复用 `followActionsProvider`/`FollowStore` 状态
  （toggle 已含取关）。新 l10n：`unfollow`（已关注态动作文案）、
  `followPrivately`（私密关注，仅未关注时显示，打开与长按同一 sheet——抽出
  公共入口 `showFollowRestrictSheet`，覆盖长按的桌面等价路径）。
- **D2 re-tap 实现 = outer controller + PrimaryScrollController**：`_UserPageState`
  持 outer `ScrollController` 传给 `NestedScrollView(controller:)`（L295 现未传）；
  内层经 `_ProfileTabBody` context 的 `PrimaryScrollController.of` 取 NestedScrollView
  自动安装的内层 controller——profile feed 是 NestedScrollView 内层 scrollable
  （profile_illust_feed.dart L65 注释，无 SmoothWheelScroll 干扰），零参数改动。
  放弃给四个 feed 注入显式 controller 的备选。
- **D3 re-tap 通道消费 W2、不自建**：回顶通道由 W2 在 `BranchSlidePager`
  same-index 路径创建（branch 级广播）；作者页（含 /me）是 root navigator 上的
  pushed route，不在 branch 内、不可达该广播——本包 re-tap 全部是**页面内**
  same-index 入口（tab bar 与 section chips 共用同一 `_scrollActiveTabToTop`
  通道），不自建第二个广播/信号机制；W2 若先合入并沉淀页面内 re-tap helper，
  rebaseline 时优先复用。
- **D4 统计集六项含 series**：`totalIllustSeries`/`totalNovelSeries` 已存在
  （修正 risks #4）；series → work tab series section；novel series 无承载
  section 不列项。他人页 myPixiv 纯展示（无 tab 承载）。
- **D5 本地过滤只过滤已加载页**：关键字匹配已加载 entities 的标题/标签；
  不触发额外请求、无服务端搜索、无自动补页；过滤态为页面内存态。PRD 写死此
  语义，验收不得按「持续拉取补偿」理解。
- **D6 收藏 sheet 失败保留草稿**：confirm 改 async——submitting 禁用 →
  `await addWithRestrict` → 读 `bookmarkStoreProvider[key].error`：非空 →
  sheet 保持打开 + 内联错误 + 草稿原样（满足父 §6「用户输入在失败时保留」）；
  为空 → pop（成功，或连接类失败已进 actionQueue 排队——排队视为已接受，
  队列会重放）。不采用「pop 后 fire-and-forget、失败仅 snackbar」的简化项。
- **D7 枚举映射收敛在 routes 门面**：`openBookmarkTags` 形参用调用方已有的
  `UserRestrict`，门面内写 `restrict.name` wire query；route builder 按
  `bookmarks/tag`（L597-600）同模式解析为 `BookmarkRestrict`；
  `BookmarkTagFeedPage` 内 `BookmarkRestrict→UserRestrict` 的 feedKey 转换
  （bookmark_tag_feed_page.dart L40-42）是唯一保留的反向映射点，不再新增
  第三处；三个平行枚举的合并属 core 重构，出本包范围。
- **D8 IgnorePointer 修复采用方案 B**：collapsed chrome 仅在 `isFullyCollapsed`
  挂载（`collapsedOpacity>0` 渲染阈值对齐为 fully-collapsed 直接 mount/unmount），
  消掉「半可见不可点」窗口；放弃阈值放宽方案 A。既有 crossfade 测试预期
  （0.55–1.0 区间可见性）同 commit 更新。
- **D9 heroScope += bookmarkTag**：scope 字符串追加 `bookmarkTag` 段
  （null 时为空段），保证 /me 收藏流与任一标签流同 ID 作品不撞 Hero tag。
- **D10 tab 缩字保留有界策略**：0.55 下限 + FittedBox 已是有界，不违反「不
  无限缩字」门禁；补 1.3x 大字号/长翻译 overflow 回归测试，运行时矩阵实测
  可读性（不可读再升级两行/省略策略，列为证据缺口）。
- **D11 保存入口 = `Scaffold.bottomNavigationBar`**：固定保存条（SafeArea
  包裹、宽屏同列限宽居中）；`resizeToAvoidBottomInset` 保留默认 true——页面
  无 autofocus，底栏随键盘上浮反而让保存可达；若运行时验证发现布局异常再按
  component-guidelines 方案（false + viewInsets padding）调整。
- **D12 可选项不执行**：pixiv 内链走内部路由（`_resolvePixivRoute` 先例）、
  /me 额外密度压缩、tag feed 页内切范围、BookmarkTagsPage 标签列表过滤框——
  均不纳入本包。
- **D13 l10n 并行冲突管理**：新 key 走 `app_{en,ja,ru,zh}.arb` →
  `flutter gen-l10n` → `tool/gen_l10n_lookup.py` 全流程；arb/lookup.dart 与
  W2/W4/W5 等并行 leaf 是高冲突区，实现期按合入顺序串行，rebase 后重跑
  gen-l10n，lookup.dart key 表冲突手工合并。

## Out of scope

- `_attemptPop` dirty 判定规则与 novel/search/settings/login 各页（W1 已落地）；
  pager 级 re-tap 广播通道本身（W2 产物，本包只消费语义不自建）；
- UserRestrict/BookmarkRestrict/FollowRestrict 三枚举合并（core 重构）；
  BookmarkTagsPage 的标签列表过滤框；标签 feed 服务端搜索（无接口）；
- `_UserPreviewTile` 与 `AuthorSummary` 收敛、novel 收藏标签页 UI（W6/无消费
  者）；阅读设置弹层（W5）、设置弹层（W8）、管理列表（W6）、作者页之外页面
  的分享/统计改造。
