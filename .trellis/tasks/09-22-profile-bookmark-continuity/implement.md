# 执行计划：作者页与收藏范围连续（W3）

需求见 `prd.md`，技术设计见 `design.md`，逐文件精确改动方案见
`research/implementation-draft.md`（行号基线 `main@8067b2d`，start 前必须
rebaseline）。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

不要手改 `android/local.properties`。新增 l10n key 流程：编辑
`lib/l10n/app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` →
`python3 tool/gen_l10n_lookup.py`。

验证命令（每阶段收尾都要跑）：

```bash
flutter analyze --no-pub
flutter test
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-profile-bookmark-continuity
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：
> 看似不相关的测试文件抛 `TimeoutException`、单文件重跑即绿——先按 spec
> 判定噪声再排查。

**启动门禁（先于阶段 1，不产生 commit）**：

- W1（09-22-interaction-outcome-correctness）已合入 `main`——`git log` 含其
  merge 且 `profile_edit_page.dart` `_attemptPop`（L137-167）干净表单直接
  pop 已存在。**未合入则本包不 `task.py start`**。
- Rebaseline：记录当时 main SHA；复核 `research/` 行号（已知失实：
  risks.md #4——`UserEntity.totalIllustSeries`/`totalNovelSeries` 已存在于
  `lib/core/user/user_entity.dart` L29-30）；检查 W2 是否已合入（本包
  re-tap 为页面内入口，不被 W2 阻塞；若 W2 已沉淀可复用 helper 则对齐）。
- 聚焦基线：`flutter test test/user_profile_test.dart
  test/profile_edit_test.dart test/bookmark_tags_test.dart
  test/bookmark_switch_button_test.dart`。

**并行冲突提示**：`routes.dart` 与 `lib/l10n/`（arb + lookup.dart + 生成物）
是与其他 leaf（W2/W4/W5 等）共享的高冲突区——按合入顺序串行；本分支 rebase
后必须重跑 `flutter gen-l10n` + `python3 tool/gen_l10n_lookup.py`，lookup.dart
key 表冲突手工合并（D13）。

本机不可验项（NestedScrollView 真机回顶、宽屏 sheet、TalkBack/Narrator、
iPad popover、1.3x 长翻译可读性、reduced-motion 路径等，见 design.md §五）
在 PR body 标「未验证」，不得由 widget test 推断通过。

## 阶段 1：作者页头部等价路径与 re-tap（R1, R2）

- [x] **统一动作源 + 挂载阈值**：`lib/features/profile/profile_header_delegate.dart`
      ——动作收敛为单一描述清单（share / follow-or-edit / restrict /
      bookmarkTags / downloadAll / more），`_CollapsedProfile` 溢出菜单
      （L539-573）与 `_ExpandedProfile` 动作行（L479-503）由同一清单生成：
      展开态主动作内联 + 其余进「更多」溢出，收起态全部进溢出；/me 收起菜单
      补 share（删 `if (!isMe)` 差异，L568-572）；collapsed chrome 的
      `IgnorePointer`（L220-223）连同 `collapsedOpacity>0` 条件渲染（L206）
      改为仅 `isFullyCollapsed` 挂载（D8）。`_HeaderBackButton`（L632-657）
      不动。
      测试：`test/user_profile_test.dart` crossfade/collapsed 组
      （L358-456、L498、L594、L650）预期同 commit 更新；新增「0.55–1.0
      区间无 collapsed chrome」「两态动作清单等价（isMe/!isMe）」用例。
      提交：`fix(profile): 头部展开/收起动作同源，收起 chrome 完全收起才挂载`
- [x] **收起态 follow 等价路径**：`lib/app/widgets/follow_switch_button.dart`
      抽公共入口 `showFollowRestrictSheet`（sheet 本体 L36-114 不变）；
      `profile_header_delegate.dart` 收起菜单他人页加「关注/取消关注」+
      「私密关注」（仅未关注时）项——复用 `followStoreProvider`/
      `followActionsProvider`（菜单 builder 内 Consumer 读 follow 态）；
      新 l10n：`unfollow`、`followPrivately`（四语 + lookup.dart）。
      测试：菜单项文案随 follow 态切换、点击调 `followActionsProvider.toggle`、
      私密关注项弹出同一 restrict sheet。
      提交：`feat(profile): 收起态溢出菜单补齐关注等价路径`
- [x] **section chips 常驻 + re-tap 回顶**：`lib/features/profile/user_page.dart`
      ——删 `_selectorExpanded`/`_onTabTap` 展开语义（L98、L132-150）；
      `_UserPageState` 持 outer `ScrollController` 传给
      `NestedScrollView(controller:)`（L295）；新增 `_scrollActiveTabToTop()`：
      outer `animateTo(0)` + 当前 `_ProfileTabBody`（L357-411）内经
      `PrimaryScrollController.of` 取内层 controller `animateTo(0)`
      （GlobalKey/回调暴露入口；reduced-motion → `jumpTo(0)`）。
      `profile_header_delegate.dart`——`ReplicaProfileTabsDelegate.maxExtent`
      （L723）对 work tab 恒 +64；TabBar `onTap` same-index →
      `_scrollActiveTabToTop`（same-index 不重复 animateTo，遵守 Tab 契约）；
      section `ChoiceChip` 重击已选项（`onSelected(false)`）→ 同一入口。
      测试：chips 在 work tab 常驻（不经 re-tap 展开）；tab same-index 与
      chip same-index 均断言 outer+inner offset 归零；非当前 tab 点击仍正常
      切换；「re-tap 不再承载展开入口」门禁断言。
      提交：`feat(profile): 作品类型常驻可见，重复点击固定为回到顶部`

## 阶段 2：统计、链接与分享（R3, R4, R5）

- [x] **可导航统计**：`user_page.dart`/`profile_header_delegate.dart`——`_Stat`
      （L675-699）与 `_ProfileStatRow`（L457-481）收敛为同一导航控件（header
      用 chip 形态、about 用行形态；`Semantics(button:)`，图标+数值+标签一次
      焦点读出）；统计集 = following/myPixiv/illust/manga/novel/series 六项
      （`totalIllustSeries` 已存在，D4）；导航映射 → `_onSectionChanged` +
      `_tabController.animateTo`（非 TabBar 回调来源，允许）；他人页 myPixiv
      纯展示；header chip 行超宽允许横向滚动。
      测试：各项点击断言 tab index + `_workSection`；myPixiv 在他人页无
      button 语义；series 项存在并导航到 series section。
      提交：`feat(profile): 统计项可导航，头部与 about 共用映射`
- [x] **社交链接打开/复制**：`user_page.dart` `_ProfileAbout`（L426-429）
      webpage/twitterUrl/pawooUrl 行改「值 + 打开 + 复制」——主操作 tap/图标
      = `outboundUrlOpenerProvider.openExternal`（android_intent_channel.dart
      L98-114），失败 snackbar 复用 `illustDetailOpenLinkFailed`（参照
      caption_rich_text.dart L46-55）；复制图标 = `Clipboard.setData` +
      `linkCopied`；新 l10n：`openLink`（四语 + lookup）。comment/id/account
      行保持 `SelectableText`。
      测试：override opener 断言 url、失败 snackbar 可见、复制写剪贴板。
      提交：`feat(profile): 社交链接主操作打开、次操作复制`
- [ ] **分享直达**：删 `_showProfileShare`（user_page.dart L541-581）；
      onShare → `shareServiceProvider.share(SharePayload.user(...),
      sharePositionOrigin: shareOriginOf(触发按钮 context))`（修掉整屏 rect），
      `copiedToClipboard` → `linkCopied` snackbar（同
      illust_detail_page.dart L204-222）；「复制链接」进动作清单溢出项
      （`copyLink` 已有）。
      测试：点击 share 直接调 share service（断言无 dialog）；copiedToClipboard
      → snackbar；菜单含复制链接独立项。
      提交：`feat(profile): 作者分享与作品一致直达，复制链接独立`

## 阶段 3：收藏范围传递与标签页（R6）

- [ ] **范围进路由**：`lib/app/navigation/routes.dart`——`openBookmarkTags`
      （L1209-1211）加 `{required UserRestrict restrict}` 形参，写
      `?restrict=<name>` query；`bookmarks/tags` route builder（L585-589）按
      `bookmarks/tag`（L597-600）同模式解析为 `BookmarkRestrict` 传给
      `BookmarkTagsPage(initialRestrict:)`（映射收敛门面，D7）。
      `lib/features/bookmark/bookmark_tags_page.dart`——构造接
      `initialRestrict`（默认 public），`_restrict` 初值来自它（L23）。
      `user_page.dart`——`onOpenBookmarkTags` 调用点（L310-313 区域）传
      `_restrict`。
      测试：路由解析（`?restrict=private` → 页面初始 private）；/me 收藏 tab
      私密范围下进标签页断言初始选中 private。
      提交：`feat(bookmark): 收藏范围经路由 query 传入标签页`
- [ ] **tag feed 上下文 + heroScope**：`lib/features/profile/
      bookmark_tag_feed_page.dart`（L32-46）AppBar 显示 tag + 范围副标题
      （`restrictPublic`/`restrictPrivate` 复用，不做页内切换）；
      `lib/features/profile/profile_illust_feed.dart` heroScope（L101-104）
      追加 `bookmarkTag` 段（null 为空段，D9）。
      测试：heroScope 断言（bookmarkTag null 与非空不同 scope）；AppBar
      副标题断言。
      提交：`fix(profile): 标签 feed 显示范围标识，heroScope 加 bookmarkTag`
- [ ] **tag feed 本地过滤**：`bookmark_tag_feed_page.dart` 加过滤输入
      （AppBar 内或 pinned 行），页面内存态 `_filter`；
      `profile_illust_feed.dart` 加可选过滤参数（默认 null 不过滤），渲染层
      对已加载 entities 做标题/标签 contains（大小写不敏感）；无匹配显示
      「已加载内容中无匹配」空态（新 l10n）。语义写死：仅过滤已加载页，
      不触发请求、无服务端搜索（D5）。
      测试：注入实体后过滤减少渲染数、清空恢复、无匹配提示、过滤不触发
      loadMore/refresh。
      提交：`feat(bookmark): 标签 feed 本地过滤已加载内容`
- [ ] **标签列表分页三态**：`bookmark_tags_page.dart` `_TagList`（L78-138）
      尾部——`loadingMore` → spinner、`loadMoreError != null` → 失败行 +
      重试（调 `loadMore()`，文案复用 `profileLoadMoreFailed`/`profileRetry`
      或新增 bookmark* key）、`!hasMore` → 结束态；不新建平行 tail 组件
      （Column 复用 FeedError 风格文案）。
      测试：error state 渲染重试行、点击触发 loadMore、hasMore=false 无
      spinner。
      提交：`fix(bookmark): 标签列表加载失败有尾部重试`

## 阶段 4：资料编辑布局（R7，消费 W1 契约）

- [ ] **固定保存入口**：`lib/features/profile/profile_edit_page.dart`——
      `FilledButton.icon`（L424-433）移出 ListView 至
      `Scaffold.bottomNavigationBar`（SafeArea 包裹，宽屏与表单同列限宽居中）；
      submitting/confirmed 禁用（L316-319）不变；`_attemptPop`/`PopScope`
      （L137-179，W1 产物）不动；`listenManual` 保活注释（L114-125）保持。
      测试：滚动任意位置保存按钮可见可达；W1 的 dirty/clean 返回断言保持绿。
      提交：`feat(profile): 资料编辑保存入口固定在底栏`
- [ ] **预览形态 + 宽屏限宽**：同文件——`_ImageField`（L459-508）拆分：
      头像 `PersonAvatar` 圆形（+ring）、背景 `PixivImage.detail` 宽条 cover
      预览（同 header 形态），已选尺寸 subtitle 保留；整页
      `LayoutBuilder`+`Center`+`ConstrainedBox`（expanded 断点起列宽
      ≤600-720 居中，`AppBreakpoints` 判定，不自造常量）；
      `resizeToAvoidBottomInset` 保留默认 true（无 autofocus，底栏随键盘
      上浮保持保存可达，D11），运行时验证。
      测试：宽屏断点下断言约束宽度；头像圆/背景宽条 finder 形态断言。
      提交：`feat(profile): 资料编辑预览形态与宽屏限宽`

## 阶段 5：收藏/关注弹层 FormState（R8）

- [ ] **控件一致性 + 宽屏限宽**：`lib/app/widgets/bookmark_switch_button.dart`
      ——`_RestrictSelect` DropdownButton（L470-535）→
      `SegmentedButton<BookmarkRestrict>`（与 follow sheet、标签页一致）；
      取消/确认 MaterialButton+headlineSmall（L391-462）→
      `OutlinedButton`/`FilledButton`；expanded 断点 sheet 内容
      `ConstrainedBox` 限宽居中（builder 内，不改 `showAppBottomSheet` 签名）。
      `lib/app/widgets/follow_switch_button.dart` sheet（L36-114）对齐间距/
      标题层级。
      测试：SegmentedButton/Outlined/Filled finder；awaitingPrefill 时
      confirm 禁用保持。
      提交：`refactor(bookmark): 收藏编辑弹层控件与全局一致`
- [ ] **draft 语义：dirty 确认 + 失败保留**：`_BookmarkEditSheet`——初始值
      基线（已收藏=回填值、新建=public+空 tags）与 `_isDirty` 判定；取消
      （L415）与下滑关闭在 dirty 时先弹 `showAppDialog` 确认（与 W1 统一
      确认层级同形，复用 profileEditLeave* 文案模式或新 `bookmarkEditLeave*`
      key）；confirm 改 async：submitting 禁用 → `await addWithRestrict` →
      读 `bookmarkStoreProvider[key].error`：非空 → sheet 保持打开 + 内联
      错误 + 草稿原样保留；为空 → pop（成功或连接类失败已进 actionQueue
      排队，排队视为已接受，D6）。确认时 `_tagInput` 残文并入 tags 的现状
      （L438-443）保留。
      测试：dirty 取消/下滑弹确认、干净直关；confirm 失败（override
      repository 抛非连接错误）sheet 保持 + 错误可见 + 草稿在；成功 pop；
      残文并入断言保持。
      提交：`feat(bookmark): 收藏编辑按 draft 语义，失败保留草稿`

## 收尾

- [ ] `flutter analyze --no-pub` 与全量 `flutter test` 通过（噪声按 spec
      判定）；`git diff --check` 干净；`task.py validate` 通过。
- [ ] PR：`gh pr create --fill`；CI 绿后 `gh pr merge --merge`。PR body
      逐项列出运行时「未验证」清单（design.md §五）。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随本 PR 的最后提交）。

## 边界（不做）

- `_attemptPop` dirty 判定（W1 已落地，本包只消费契约）；pager 级 re-tap
  广播通道（W2 产物，本包不自建第二个信号机制）；
- UserRestrict/BookmarkRestrict/FollowRestrict 三枚举合并（core 重构）；
  BookmarkTagsPage 标签列表过滤框；标签 feed 服务端搜索（无接口）；
- `_UserPreviewTile`/`AuthorSummary` 收敛（W6 对象基座）；novel 收藏标签页；
  阅读设置（W5）/设置（W8）/管理列表（W6）弹层；pixiv 内链内部路由；
  /me 密度额外压缩（D12）。
