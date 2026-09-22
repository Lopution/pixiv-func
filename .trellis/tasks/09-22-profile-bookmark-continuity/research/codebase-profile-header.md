# Codebase: 作者页 header 结构与动作清单

基线：`main@8067b2d`（HEAD，PR #54 合并后）。W1 叶子（09-22-interaction-outcome-correctness）仍只有 TBD prd，返回契约尚未落地。

## 1. 页面骨架（lib/features/profile/user_page.dart）

- `UserPage`（L30-44）：`userId`/`isMe`/`onEditProfile`；`MePage`（L48-89）在 build 时解析当前账号再构造 `UserPage._me`，`onEditProfile` 缺省为 `openProfileEdit(context, account.userId)`（L84）。
- 状态（_UserPageState L94-101）：`_tabController`、`_selectorExpanded`（默认 false）、`_workSection`（默认 illust）、`_restrict`（默认 `UserRestrict.public`）、`_selectedIndex`。
- tabKeys（L106-119）：
  - isMe：`profileBookmarked / profileFollowing / profileFans / profileMyPixiv / profileWork`（workTab 在 index 4）
  - 他人：`profileWork / profileBookmarked / profileFollowing / profileAbout`（workTab 在 index 0，about 是纯文本 tab 而非 feed）
- `_onTabChanged`（L132-141）：切 tab 时收起 `_selectorExpanded`。
- `_onTabTap`（L143-150）：**重复点击当前 work tab → 切换 `_selectorExpanded`**；其余 tab 重击无行为。这违反 §5.1「重复点击固定为回到顶部」。
- `_feedKeyFor`（L152-196）：`ProfileFeedKey(userId, kind, workType, restrict, bookmarkTag)`。`_restrict` 只进 `bookmarks` 与 `following` 两个 kind；fans/myPixiv 无 restrict 维度（仓库侧也只有 following 接受 restrict，user_repository.dart L246）。
- `showRestrictSelector`（L273-274）：`isMe && (_selectedIndex == 0 || _selectedIndex == 1)`——只有 /me 的收藏/关注 tab 才显示范围入口，且只渲染在**收起态**溢出菜单里（见下）。
- `canBulkDownload`（L276-279）：work tab 且 section ∈ {illust, manga} 时溢出菜单出现「下载全部作品」。
- 结构（L295-350）：`NestedScrollView`（**未传 controller**）+ 两个 pinned `SliverPersistentHeader`（`ReplicaProfileHeaderDelegate` + `ReplicaProfileTabsDelegate`）+ `TabBarView`。
- `_ProfileTabBody`（L357-411）：`AutomaticKeepAliveClientMixin`；`ValueKey(_feedKeyFor(index) ?? _tabKeys[index])`（L338-340）——restrict/section 变化会换 key 重建整个 body。分发：series → `UserSeriesFeed`（L392-397）；workType==novel → `ProfileNovelFeed(userId)`（L398-403，**不带 restrict**）；relation → `ProfileUserFeed`；其余 → `ProfileIllustFeed`。
- `_ProfileAbout`（L413-485）：条目式 SelectableText：id、account、comment、webpage、twitterUrl、pawooUrl（L426-429）+ `_ProfileStatRow` 五个**不可导航** ListTile（L457-481：following/myPixiv/illust/manga/novel 总数）。
- `_showProfileShare`（L541-581）：`showAppDialog` AlertDialog，SelectableText 显示 `https://www.pixiv.net/users/<id>\n<name>`，「复制链接」写 `payload.text`，「分享」走 `shareServiceProvider.share(SharePayload.user)`。与作品分享（`illust_detail_page.dart` L204-222、`illust_card_actions.dart` L228-263 均为**直接 share**+剪贴板回退 snackbar）路径不一致——§4.3 要求分享默认路径与作品一致、复制链接独立。

## 2. Header delegate（lib/features/profile/profile_header_delegate.dart）

- `ReplicaProfileHeaderGeometry`（L17-108）：`expandedAvatarRadius=52`；`expandedIdentityExitProgress=0.78`、`expandedDetailsFadeStart=0.55`；`isFullyCollapsed` = shrinkOffset ≥ collapseRange−0.5；`collapsedOpacity` 从 0.55 起渐入。
- `ReplicaProfileHeaderDelegate`（L116-273）：`minExtent = kToolbarHeight + topInset`（L150），`maxExtent = max(expandedExtent, minExtent)`；expandedExtent /me=350、他人=320（user_page.dart L305）。
- 三层 Stack（L174-252）：背景带透明度淡出（L181-190）→ expanded identity 整块平移+详情淡出（L191-205）→ collapsed toolbar `Opacity(collapsedOpacity)` 内包 `IgnorePointer(ignoring: !isFullyCollapsed)`（L206-240）→ 常驻 `_HeaderBackButton`（L246-250，latch 首个 canPop 值，L632-657）。
- **Astra #8 断点**：collapsed toolbar 在 fade-in 过程中可见但不可点（IgnorePointer 到完全收起才解除，L220-223）；且收起后无任何「关注」入口（见下）。

### 展开态 `_ExpandedProfileDetails`（L401-507）
- 名字（L434-445）、account（L448-454）。
- 三个 `_Stat`（L456-474）：`AppIcons.follow`+totalFollowUsers、`AppIcons.friend`+totalMyPixivUsers、`palette_outlined`+totalIllusts —— 纯 Row 文本+图标，**无语义、不可点**（L675-699 `_Stat` = Expanded+Row）。
- 动作行（L479-503）：share `IconButton`（恒有）；isMe → edit `IconButton`（L489-495）；他人 → `FollowSwitchButton`（L496-501）。**无 restrict 选择器、无书签标签入口、无 downloadAll** —— 这些只在收起态溢出菜单。
- 无背景图时走主题色（`onArtwork` 分支 L421-425）；有背景图时白色文字压在 0x59/0x66/0x8C 黑色渐变 scrim 上（`_ProfileBackground` L368-397）。

### 收起态 `_CollapsedProfile`（L509-625）
- 溢出菜单条目（L539-573）：`restrictPublic/restrictPrivate` CheckedPopupMenuItem（仅 `isMe && showRestrictSelector`）、`bookmarkTags`（仅 isMe+收藏 tab，onOpenBookmarkTags）、`downloadAll`、`editProfile`、**`share` 仅 `!isMe`**（L568-572）。
- **缺口清单（对照 §4.3「展开/收起状态保留返回、分享、更多和关注/编辑的等价路径」）**：
  - 收起态无 FollowSwitchButton——他人页收起后无法关注/取关（share 在菜单里，但 follow 完全没有）。
  - /me 收起态无 share 项（`if (!isMe)`）。
  - 展开态无 restrict/bookmarkTags/downloadAll 入口。
  - collapsedOpacity>0 但未 fullyCollapsed 时菜单按钮可见不可点（IgnorePointer）。
- 标题居中 `Text(user.name)` key `profile-toolbar-title`（L600-607）；溢出 `PopupMenuButton`（L614-619）。

### Tab 条 `ReplicaProfileTabsDelegate`（L702-861）
- `minExtent=kToolbarHeight`；`maxExtent = kToolbarHeight + (expanded && _isWorkTab ? 64 : 0)`（L723）——64dp 的 section chip 行只在展开态出现。
- 标签字重缩放：LayoutBuilder 量最宽译文 → `scale` clamp 0.55–1.0（L764-791）+ `FittedBox(scaleDown)`（L807-813）。§W2/W3 门禁「大字号/长翻译不通过无限缩字维持单行」相关，当前有 0.55 下限但仍以缩字为主策略。
- `onTap: onTabTap`（L803）——TabBar 自身 animateTo + 回调里只切 `_selectorExpanded`（符合 Tab 契约，没有二次 animateTo）。
- Section chips（L821-849）：`ProfileWorkSection.values` → illust/manga/novel/series ChoiceChip；`series` 走独立 feed。

## 3. 关键缺口汇总（供 implementation-draft 引用）

| §4.3 要求 | 现状 | 断点位置 |
|---|---|---|
| 展开/收起保留等价路径 | 收起无 follow、/me 收起无 share、展开无 restrict/tags/download | profile_header_delegate.dart L220-223、L488-501、L539-573 |
| 作品类型持续可见 | section chips 只在 re-tap 展开的 64dp 区出现 | user_page.dart L143-150；delegate L723、L821 |
| 可导航统计带标签 | 3 个无标签 `_Stat` + about 页 5 个 ListTile 均不导航 | delegate L456-474；user_page.dart L457-481 |
| 社交链接打开/复制 | 仅 about tab `SelectableText` | user_page.dart L426-429 |
| 分享与作品一致 | profile 走 AlertDialog 两步 | user_page.dart L541-581 |
| re-tap = 回顶 | re-tap = 展开 selector | user_page.dart L143-150 |
