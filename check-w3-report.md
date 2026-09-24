# Check Report: W3 作者页与收藏范围连续 (PR #62, merge 7e4a041)

工作目录: /root/Pixiv-func-check3 (detached HEAD @ b31846c)
基准 diff: `git diff 7e4a041^ 7e4a041`

## 增量检查记录

### R7 profile_edit_page.dart — PASS(初查)
- `ContentWidths.form` 被消费：`_profileEditContentMaxWidth` (profile_edit_page.dart:670-673) 是断点门控函数，`>= AppBreakpoints.expanded` 时返回 `ContentWidths.form`(520)，否则 `availableWidth`；未自造宽度数值常量。符合 design.md §5.5 "列宽 ≤ContentWidths.form（520）...不自造宽度常量"。
- 实现用 `MediaQuery.widthOf` 而非 implement.md 写的 `LayoutBuilder`——功能等价（Scaffold body 约束=屏宽），不属违规。
- `bottomNavigationBar` 固定保存条：SafeArea+Align(heightFactor:1.0)+ConstrainedBox(contentMaxWidth)+viewInsets padding 抬升——实现且注释解释了 keyboard 上浮（D11 变形：resizeToAvoidBottomInset 默认 + 自抬 viewInsets，语义保持"保存可达"）。
- `PopScope canPop: !hasUnsaved`（原 `session == null || !hasUnsaved`）：session==null 时 hasUnsaved=false → 等价。`_attemptPop` 本体未在 diff 中出现，待核对。
- 头像 PersonAvatar ring / 背景 AspectRatio(3.2) PixivImage.detail cover 预览拆分实现（R7）；已选尺寸 subtitle 保留。

### R1/R2/R3 profile_header_delegate.dart + user_page.dart — PASS(含1疑)
- 两态同源：`ReplicaProfileHeaderDelegate._actions` 单一 `_ProfileHeaderAction` 清单（share/editProfile/toggleFollow/followPrivately/restrict×2/bookmarkTags/downloadAll/copyLink），展开态 `where(primary)` 内联 + `_ProfileHeaderMoreButton(includePrimary:false)` 溢出余项，收起态 `includePrimary:true` 全进溢出。满足 R1 同源。
- `collapsed chrome` 仅 `geometry.isFullyCollapsed` 挂载，Opacity+IgnorePointer 整体删除——无"可见不可点"窗口（D8）。`_HeaderBackButton` 不动。
- chips 常驻：`ReplicaProfileTabsDelegate.minExtent = kToolbarHeight + (_isWorkTab ? 64 : 0)`，`maxExtent == minExtent`，`expanded` 参数删除（R2）。
- re-tap：`_onTabTap` same-index → `_scrollActiveTabToTop`（inner PrimaryScrollController → outer animateTo(0)，MotionTokens.enabled 决定 animate/jumpTo）；`_onSectionChanged` same-section → 同一入口。
- **疑点1**：`PrimaryScrollController` 是 NestedScrollView 共享内层 controller——keep-alive 的各 tab body 位置均 attach，`animateTo(0)` 会把**所有**已挂载 tab 的滚动位置归零，不只是当前 tab。spec 写死了此机制，副作用需记录。
- **疑点2（偏离 spec）**：series 统计值 = `totalIllustSeries + totalNovelSeries`（user_page.dart:281），测试断言 `系列, 7`。但 PRD R3 写死「series 用 totalIllustSeries」+ D4「novel series 无承载 section 不列项」，且目标 `UserSeriesFeed` 仅展示 illust series——计数与目标内容不符。判定：可修偏离。

### R4/R5 社交链接+分享 — PASS
- `_ProfileSocialLinkRow`：值 SelectableText + open/copy 两可见 IconButton（profile-link-open/copy-$id）；`_openProfileSocialLink` 用 `outboundUrlOpenerProvider.openExternal` + `illustDetailOpenLinkFailed` snackbar；`_copyProfileSocialLink` → Clipboard+linkCopied。comment/id/account 保持 SelectableText。
- `_showProfileShare` AlertDialog 整体删除；`_shareProfile(originContext)` 直接 `shareServiceProvider.share` + `shareOriginOf(originContext)`（header share IconButton 用 Builder 拿按钮 context 修 popover 锚点）；copiedToClipboard → linkCopied；`copyLink` 独立溢出项 `_copyProfileLink` → Clipboard+snackbar。

### R6 范围路由/过滤/分页/heroScope — PASS
- `openBookmarkTags(context,{required UserRestrict})` → `_bookmarkRestrictFromUser`(UserRestrict→BookmarkRestrict) `.name` 写 `?restrict=`；route builder `_bookmarkRestrictFromQuery` 解析 → `BookmarkTagsPage(initialRestrict:)`。bookmarks/tag 路由同收共用 helper。枚举映射收敛在 routes 门面（D7）；`BookmarkTagFeedPage` 内 BookmarkRestrict→UserRestrict 为唯一反向映射点。
- `BookmarkTagsPage` initialRestrict→`_restrict`（initState+didUpdateWidget 同步）。
- 标签 feed：AppBar tag+范围副标题（restrictPublic/Private，无页内切换）；AppBar bottom TextField 本地过滤 `_filter` 页面内存态 → `ProfileIllustFeed.localFilter` 渲染层 contains 过滤 title/tags(含 translatedName)，大小写不敏感；不进 feedKey/provider（无 cacheKey 分叉）。空匹配 → `bookmarkTagFilterEmpty`。
- heroScope `'profile:uid:kind:workType:restrict:bookmarkTag??""'`（D9，null 空段）。
- 标签列表尾部三态：`loadingMore`→spinner、`loadMoreError`→失败行+重试 loadMore()、`!hasMore`→bookmarkTagsEnd 结束态。

### R8 收藏 sheet draft — PASS(机制已验证)
- `SegmentedButton<BookmarkRestrict>` 替换 `_RestrictSelect` DropdownButton（类已删）；OutlinedButton/FilledButton 替换 MaterialButton+headlineSmall。
- draft 基线 `_initialRestrict/_initialTags`；`_isDirty` = restrict/tags 与基线不同（listEquals）。prefill 晚到时：stillPristine 则套用基线值，否则保留用户已输入、只移基线。
- 三路径 dirty 确认：取消按钮→`_attemptClose`；barrier/系统返回→PopScope(canPop:!_closableFreely) 拦（maybePop 走 disposition，routes.dart:988 证实）；下滑→外层 GestureDetector onVerticalDragEnd 抢先（比 BottomSheet 内层 drag recognizer 更深，赢得 arena）→`_attemptClose`。确认弹窗复用 profileEditLeave* 文案。
- imperative `Navigator.pop` 不查 popDisposition（navigator.dart Route.didPop→didComplete）→ 确认后关闭不递归。代码注释与此一致。
- `_confirm` async：submitting 禁用→`await addWithRestrict`→读 `bookmarkStoreProvider[key].error`：非空→sheet 保持+`bookmarkOperationFailed` 内联错误+草稿原样（_tagInput 残文也还在输入框）；空→pop（成功或已排队，D6）。残文并入 tags 保留。
- sheet 宽屏 `Align(heightFactor:1.0)+ConstrainedBox(ContentWidths.form)`（>=expanded）；`showAppBottomSheet` 签名未动。
- follow sheet 抽公共入口 `showFollowRestrictSheet`，ContentWidths.form 同形限宽；控件本来就是 SegmentedButton+Outlined/Filled。
- 观察项（非违规）：`_isDirty` 不含 `_tagInput` 未提交残文——spec 明确 dirty=tags/restrict 不同，按字面合规；残文在失败/取消路径视觉保留。

### l10n — PASS
- 新 key×6（followPrivately/unfollow/openLink/bookmarkTagFilterEmpty/bookmarkTagFilterHint/bookmarkTagsEnd）四语 arb+生成物+lookup.dart 同步。bookmarkOperationFailed/profileSeries 等复用已有 key。

### PopScope/_attemptPop(W1契约) — PASS
- `_attemptPop` 本体零改动（干净直 pop、脏→dialog→controller.cancel()→pop）；AppBar back 直接调 `_attemptPop`；`canPop` 改 `!hasUnsaved`（session==null 时等价 true）。

### 验证
- flutter analyze --no-pub：No issues found（需先 flutter pub get，本 worktree 无 .dart_tool）。
- 聚焦测试：进行中。

### 提交粒度/PR — PASS
- 14 个 feat/fix commit 与 implement.md 14 个勾选框一一对应 + journal + archive；分支名 `task/09-22-profile-bookmark-continuity` 合规。
- PR #62 body 含 ContentWidths 首消费者说明（W9 #61 先合入同值文件，rebase 取齐）、下滑拦截机制说明、§五 未验证清单全列。

### 发现的问题
- **FIXED** `user_page.dart:281` — series 统计值 `totalIllustSeries + totalNovelSeries` 偏离 PRD R3「series 用 totalIllustSeries」/D4「novel series 无承载 section 不列项」；目标 `UserSeriesFeed` 仅展示 illust series，求和计数与目标内容不符。已改为 `user.totalIllustSeries`，测试断言 `系列, 3`。
- **FIXED** AC 门禁缺两项回归测试：feed error 下 header/tab/动作可见性（父 §6 W2/W3）、1.3x 大字号 tab 标签无溢出/单行（D10+AC）。已在 test/user_profile_test.dart 补 `header tabs and actions stay mounted while the work feed fails` 与 `tab labels stay on one line at 1.3x text scale`（fake repo 加 `worksFailure` 字段，抛 `ApiNetworkError`）。
- **NOTED（报告，未修）** `_scrollActiveTabToTop` 经共享内层 `PrimaryScrollController.animateTo(0)`——NestedScrollView 内层 controller 被所有 keep-alive tab 的 ScrollPosition 共享，animateTo 会把**全部**已挂载 tab 归零，不只当前 tab。这是 D2 指定机制的固有副作用（PRD 写死"内层经 PrimaryScrollController.of animateTo(0)"），精确定位活跃 position 需 element 祖先遍历或 feed 级 controller 注入（违背 D2 零参数约定）。建议 owner 知情决策。
- **NOTED** `_isDirty` 不含 `_tagInput` 未提交残文——spec 字面定义 dirty=tags/restrict 不同，合规；残文在输入框中视觉保留。
- **NOTED** `_submitting` 期间 sheet 可自由关闭（`_closableFreely`），失败后错误留在 store entry 无额外提示——代码注释说明为有意取舍，边界情形。
- **NOTED** `ReplicaProfileTabsDelegate.shouldRebuild` 不比较 tab index（`_isWorkTab` 读 `controller.index` 活值，新旧 delegate 共享 controller 故比较无意义）；tab 切换时 body 换掉触发 viewport relayout，extent 同帧刷新——实际无陈旧窗口。

### 验证结果（本 worktree 实测）
- `flutter analyze --no-pub`: **No issues found**（前提：先 `flutter pub get`，本 worktree 原本无 .dart_tool）
- `flutter test` 聚焦五文件（user_profile/profile_edit/bookmark_tags/bookmark_switch_button/bookmark_tag_feed_page）: **65 passed**（修复后 user_profile 23 项含新增 2 项全绿）
- `git diff --check`: 干净

### 修复落地
- 分支 `fix/09-22-w3-check`（自 origin/main@b31846c），commit `4016ad5`，PR **#79** 已开。
- 改动：`user_page.dart` series 统计回 `totalIllustSeries`；`user_profile_test.dart` +2 门禁回归测试、series 断言同步、`_FakeUserRepository` 加 `worksFailure`。

## 总结
R1–R8 全部落地且与 spec 契约一致（动作同源、isFullyCollapsed 挂载、re-tap 双路径回顶、统计共用映射、社交链接开/复、分享直达、restrict 路由 query、本地过滤渲染层、尾部三态、heroScope+bookmarkTag、ContentWidths.form 消费、sheet draft 三路径确认+失败保留草稿）。发现 1 个 spec 偏离（series 求和）+2 个门禁测试缺口，均已修复入 PR #79；4 项设计层面观察如实记录未动。
