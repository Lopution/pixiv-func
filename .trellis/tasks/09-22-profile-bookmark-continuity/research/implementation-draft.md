# Implementation Draft: W3 profile-bookmark-continuity

逐条对应父任务 design.md §4.3。文件/行号基线 `main@8067b2d`；W1 未合入（其叶子只有 TBD prd），下列「消费 W1」项按预期契约写，实际行号以 W1 合入后 rebaseline 为准。

## A. 展开/收起等价路径（Astra #8）

现状：`profile_header_delegate.dart` 收起态 `IgnorePointer(ignoring: !isFullyCollapsed)`（L220-223）包住 `_CollapsedProfile`；follow 只在展开态（L496-501），/me 的 share 只在展开态（collapsed 菜单 `if (!isMe)` L568-572），restrict/bookmarkTags/downloadAll 只在收起态菜单（L539-563）。

方案：
1. 动作集合收敛为同一来源——header delegate 接收统一的动作描述（share / follow-or-edit / restrict / bookmarkTags / downloadAll / more），展开态摆不下时进「更多」溢出，收起态全部进溢出菜单；两态由同一 List 生成，杜绝「可见但不可点/存在但缺失」。M3 flexible app bar 语义：collapsed small bar 保留 navigation+actions。
2. `IgnorePointer` 区间修正：选项 A 把 `ignoring` 阈值从 `isFullyCollapsed` 放宽到 collapsedOpacity≈1 之前更早（如 progress≥0.9），选项 B 干脆让 collapsed chrome 只在完全收起时才渲染（`collapsedOpacity>0` 已是条件渲染 L206，可把阈值对齐 `isFullyCollapsed` 直接 mount/unmount，消掉「半可见不可点」窗口）。推荐 B：与「可见即可点」最简单一致；需回归 `user_profile_test.dart` 的 crossfade/渐断测试组（L358-456 相关用例）。
3. 收起态溢出菜单补 follow 路径：他人页收起时菜单加入 follow/unfollow（复用 `followActionsProvider`/`FollowStore` 状态，不重建 FollowSwitchButton 视觉）；/me 收起菜单补 share。私密关注等长按等价路径同时进菜单（§5.6 桌面等价）。
4. `_HeaderBackButton`（L632-657）保持不变——已是常驻等价路径。

## B. 作品类型持续可见 + re-tap 回顶（Astra #9 前半，§5.1）

现状：`_onTabTap` 重击 work tab 切 `_selectorExpanded`（user_page.dart L143-150）；section chips 仅在 `expanded && isWorkTab` 时占 64dp（profile_header_delegate.dart L723、L821-849）。

方案：
1. Section 选择器改为持续可见：work tab 激活时 64dp chip 行常驻（`ReplicaProfileTabsDelegate.maxExtent` 对 work tab 恒 +64），删除 `_selectorExpanded`/`_onTabTap` 展开语义。/me 更紧凑可通过缩减 expandedExtent 或行内密度实现，**导航结构不变**。
2. Re-tap 新语义=回到顶部：需要内层滚动位置。NestedScrollView 的 body 内 `PrimaryScrollController.of(context)` 即内层 controller（NestedScrollView 自动安装）。做法：`_UserPageState` 持有 outer `ScrollController` 传给 `NestedScrollView(controller:)`（当前未传，L295）；每个 `_ProfileTabBody` 给 `GlobalKey`/回调暴露「回顶」——其 context 里 `PrimaryScrollController.of(ctx)` 取内层 controller `animateTo(0)`，outer `animateTo(0)` 展开 header。两步都要做（PixEz 只动 outer，列表仍停在深处，语义不完整）。
   - 备选：给四个 feed widget 加可选 `scrollController` 参数（它们现在的 CustomScrollView 均不传 controller）。传参更直接但要小心：NestedScrollView 内层 scrollable 用自定义 controller 后与 coordinator 的联动需实测（HeaderLocator/isNested 契约不受影响，它是 sliver 机制）。PrimaryScrollController 方案零参数改动，优先。
3. 非 work tab 的 re-tap 同样回顶（§5.1 不分 tab）。
4. 门禁回归：「重复点击当前分类/标签固定为回到顶部，不再承载展开入口」「排行/作者 header 点击和滑动规则有回归测试」。

## C. 可导航统计（Astra #9）

现状：展开态 `_Stat`×3 纯展示（delegate L456-474，缺 manga/novel/series）；about tab `_ProfileStatRow`×5 不导航（user_page.dart L457-481）。

方案：
1. 统计项=「图标+数值+标签」的可点 chip/行（Semantics button，焦点一次读出「关注 · 123」），点击行为=导航到对应目标：
   - following → 切到 following tab（页内 `TabController.animateTo`——**注意 Tab 契约：非 TabBar 回调来源的程序化切换可以 animateTo**）；
   - myPixiv → /me 的 myPixiv tab / 他人页无此 tab（他人页该项可不显示或仅展示）；
   - illust/manga/novel → work tab + 对应 section（需要把 `_workSection` 设为某值并切 tab——`_onSectionChanged`+`animateTo` 组合）；
   - series 若加统计（user.totalIllustSeries 字段需确认存在；当前 UserEntity 无此字段，见 risks）→ work tab series section。
2. 展开态 header 的 3 个 `_Stat` 扩展为同一套可导航统计（带标签）。/me 页 about 信息无 tab 承载——统计导航是 /me 上唯一看见 totals 的入口（me tabKeys 无 about），更有必要可点。
3. about tab 的 `_ProfileStatRow` 同步改为同一控件（行式而非 chip，但 onTap 相同）。

## D. 社交链接：主操作打开、次操作复制（Astra #9）

现状：about tab `SelectableText`（user_page.dart L426-429：webpage/twitterUrl/pawooUrl）。

方案：
1. 每行改为「label + 值 + 打开图标 + 复制图标」或 tap=打开/long-press/次按钮=复制。打开走 `outboundUrlOpenerProvider.openExternal`（android_intent_channel.dart L100-114，已做 http(s) 校验+失败抛错），失败 snackbar 复用 `illustDetailOpenLinkFailed` 模式（caption_rich_text.dart L46-55）。
2. 需新增 l10n key（如 `openLink`/`copyLink` 已有 copyLink；open 动词缺）——arb + `l10n/lookup.dart` 同步。
3. pixiv 内链（用户主页 URL 出现在 webpage 字段时）是否走内部路由：可选增强，参考 `CaptionRichText._resolvePixivRoute`；不作为必须项。
4. comment 字段：`SelectableText` 保留或换 `CaptionRichText`（comment 可能含 HTML 链接——PixEz 用 SelectableHtml；我们的 comment 是 `user.comment` 纯文本字段，先确认 API 是否给 HTML，见 risks）。

## E. 分享默认路径与作品一致（Astra #9）

现状：`_showProfileShare` 开 AlertDialog 二选一（user_page.dart L541-581）。

方案：onShare 直接 `shareServiceProvider.share(SharePayload.user(...), sharePositionOrigin:)`，`copiedToClipboard` 回退 snackbar（与 illust_detail_page.dart L204-222 同形）；「复制链接」降级为溢出菜单独立项（`copyLink` l10n 已有）。删除整个对话框。`shareOriginOf(context)` 的 context 应取触发按钮的 context 以正确定位 iPad popover（当前传的是页面 context=整屏 rect）。

## F. 公开/私密范围传递（Astra #10）

断点：`openBookmarkTags` 无参（routes.dart L1209-1211）、`bookmarks/tags` 路由无 query（L585-589）、`BookmarkTagsPage._restrict` 硬编码 public（bookmark_tags_page.dart L23）。

方案：
1. `BookmarkTagsPage({BookmarkRestrict initialRestrict = public})`；`openBookmarkTags(context, {restrict})`；route `bookmarks/tags?restrict=` 解析（与 `bookmarks/tag` L597-600 同模式）——范围即 durable query，顺带满足 §5.1 恢复等级。
2. /me 收藏 tab 调 `openBookmarkTags` 时把 `_restrict`（UserRestrict）映射为 BookmarkRestrict——枚举边界在 routes 门面或调用点收敛一处（建议门面做，调用方只传已有枚举）。
3. 标签页内 SegmentedButton 仍可改范围（现有行为保留），点 tag 时把**当前选中** restrict 传给 `openBookmarkTagFeed`（已是如此，L132-133）。
4. `BookmarkTagFeedPage` AppBar 增加范围副标题/标识（tag + 公开/私密），并可考虑 in-place 切范围（改 query 或重建 feedKey）；本地过滤：标签 feed 是否加搜索/过滤框由 PRD 定（§4.3「增加本地过滤」——参考 PixEz bookmark_search 的在页内关键字过滤 tag/title 思路；若要 server 侧则无接口，只能本地对已加载页过滤，语义需在 PRD 写清）。
5. `UserBookmarkTagsState.loadMoreError` 接入尾部：失败行 + 重试（当前从不渲染）；hasMore/loadingMore/error 三态齐全。FeedTail 只认 PagedFeedState，tag 列表是自定义 state——给 `_TagList` 尾部手写三态或抽轻量 tail（不新建平行组件：直接 Column 复用 FeedError 风格文案即可）。

## G. 资料编辑布局（Astra #11 后半）

前提：W1 修 `_attemptPop`（profile_edit_page.dart L137-167）dirty 判断；W3 不改判定。

方案：
1. 固定保存入口：`FilledButton.icon`（L424-433）移出 ListView——`Scaffold.bottomNavigationBar` 或 AppBar actions 放保存（M3 惯例：form 页常用 bottom bar 或 top action）。submitting/confirmed 禁用逻辑（L316-319）原样保留；密码框在 requiresCurrentPassword 时仍在表单内。
2. 预览形态：`_ImageField` 拆为两形态——avatar 用 `PersonAvatar`（圆+ring，现成）/ Image.file 圆形裁剪；background 用宽条 `PixivImage.detail(cover)` 预览（与 header 背景同形态）。选择后预览显示已选文件尺寸（现有 subtitle 保留）。
3. 宽屏限宽：form 角色按 §5.5 用 `AppBreakpoints`（expanded=1200 起限宽，form 列宽建议 ≤600-720 居中）；页面已是独立路由页，直接 `LayoutBuilder`+`Center+ConstrainedBox` 即可，不动 breakpoint 常量。
4. 键盘：`Scaffold.resizeToAvoidBottomInset:false`+padding viewInsets 评估（component-guidelines 已知坑），运行时验证。

## H. 收藏编辑 sheet FormState/控件迁移（Astra 弹层项）

现状见 codebase-overlays-forms.md §3。方案：
1. `_RestrictSelect` DropdownButton → `SegmentedButton<BookmarkRestrict>`（与 follow sheet、tags 页一致）。
2. 取消/确认 MaterialButton+headlineSmall → `OutlinedButton`/`FilledButton`（与全局 action 按钮一致）。
3. dirty 离开确认：tags/restrict 与初始值不同时（已收藏=回填值、新建=public+空 tags），cancel/下滑关闭弹 §5.3 draft 确认框；sheet 的 `Navigator.pop` 与系统返回都要拦（`showModalBottomSheet` 内嵌 `PopScope` 或 confirm 前比对）。确认按钮 pop 后提交的现状保留（mutation 经 BookmarkStore 状态机，失败走 store error snackbar L71-81——已符合「失败保留可辨」；但 sheet 关闭后草稿即弃，若 store 失败无法重填——PRD 需明确失败后是否重开 sheet 恢复草稿，当前实现不支持）。
4. 宽屏：`showAppBottomSheet` 全宽 → expanded 断点下 sheet 内容限宽居中（builder 内 ConstrainedBox，不改 app_overlays 签名）。
5. follow sheet（follow_switch_button.dart L36-114）同规检查：已是 SegmentedButton，按钮已是 Outlined/Filled——只需对齐间距/标题层级，工作量小。

## I. W1 返回契约消费点

- `ProfileEditPage`：`PopScope`（L175-179）已是 dirty-aware；W1 把 `_attemptPop` 改为 `!hasUnsavedChanges → 直接 pop`。W3 移动保存按钮/改布局时保持该契约与 `test/profile_edit_test.dart` L415 的 dirty-离开断言（测试名 'asks before leaving dirty form' 预期要随 W1 更新——无改动不再弹框）。
- `_HeaderBackButton`/`maybePop` 路径不变。
- 书签 sheet 新加的 dirty 确认须与 W1 的统一确认层级一致（同一 `showAppDialog`+同一文案结构，复用 profileEditLeave* 文案模式或新 key）。

## J. 门禁对照（implement.md §6 W2/W3）

- 跳转前后范围/类型/筛选/私密一致 → F 的 route param 测试；
- 控件在 loading/error/empty 仍可见 → header/tab 不随 feed 状态消失（已天然满足，回归即可）；
- 大字号不无限缩字 → tab 标签现有 0.55 下限缩字策略需评估是否保留/换 2 行或省略策略（design 要求「不通过无限缩字维持单行」，0.55 下限已算有界，但建议给 tab 文案加 overflow 测试）；
- re-tap 回顶回归测试；
- 资料/收藏弹层 FormState 测试：draft dirty 确认、失败保留、320/390/600/840/1200dp 主操作可达。
