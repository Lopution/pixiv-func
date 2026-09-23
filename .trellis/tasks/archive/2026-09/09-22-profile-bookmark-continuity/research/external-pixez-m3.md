# External: PixEz 作者页与 Material 3 对标

来源：github.com/Notsfsssf/pixez-flutter `master`（2026-09 拉取）：`lib/page/user/users_page.dart`、`lib/page/user/detail/user_detail.dart`、`lib/page/user/bookmark/bookmark_page.dart`、`lib/page/user/bookmark/tag/user_bookmark_tag_page.dart`。M3 侧：m3.material.io/components/app-bars（guidelines/specs）。

## 1. PixEz UsersPage（他人作者页）

- 结构：`NestedScrollView(controller: _scrollController)` + `SliverOverlapAbsorber` + `SliverAppBar(pinned, expandedHeight≈width/2+205 或 300)` + `ColoredTabBar` bottom（works / bookmark / user_page_info_title 三 tab）。
- **Re-tap = 回顶**（users_page.dart ~L332-342）：`TabBar.onTap` 中 `if (_tabController.index == index && _scrollController.hasClients) _scrollController.animateTo(0, 300ms, easeOut)`，并 `HapticUtil.selectionClick()`。注意它动的是 **NestedScrollView 外层 controller**——效果是展开 header；内层列表偏移不动。我们的 §5.1「回到顶部」语义更强（应含内层列表回顶），实现需内外两个 controller（见 implementation-draft）。
- **AppBar actions**：share `IconButton` → 直接 `SharePlus.instance.share('https://www.pixiv.net/users/<id>', sharePositionOrigin: box)`，**无中间对话框**；`PopupMenuButton`：quietly_follow（私密关注）、block_user、copymessage（复制 `painter:<name>\npid:<id>`）、report、novel 页入口。即：分享主操作直达，复制是独立菜单项——与 §4.3「分享默认路径与作品分享一致，复制链接独立」一致。
- **范围传递**：页面级 `String restrict='public'`；`BookMarkNestedPage`（bookmark_page.dart）内部用 `SortGroup` chips 切 public/private 重建 `ApiForceSource`；标签入口打开 `UserBookmarkTagPage`（含 public/private TabController + 输入框），**pop 回传 `{tag, restrict}`**，bookmark 页用返回值重建 feed——范围在收藏上下文内闭环，不需要路由参数。对我们（路由化架构）的启示是「范围始终作为参数进出页面」，不一定照搬 pop 结果模式。
- `UsersPage` 本身不切换 tab 数；isMe 是另一套页面（PixEz 我的主页在 hello/account 体系），与我们 MePage 复用 UserPage 不同——我们的「自己主页更紧凑但不改导航结构」没有直接先例，按 §4.3 自行设计。

## 2. PixEz UserDetailPage（作者详情 tab）

- `DataTable` 行 = 统计/资料的可导航载体：
  - `painter_id` 行 onTap → `Clipboard.setData`（点 ID 即复制）。
  - `total_follow_users` 行 onTap → push `FollowList(id)`；`total_mypixiv_users` 行 → `FollowList(isFollowMe:true)` —— **统计行带 tap 导航**的先例。
  - `twitter_account` 行 onTap → `launchUrlString(twitter_url)`（iOS externalApplication），catch 后回退 `SharePlus.share(url)` —— 社交链接「主操作打开、失败回退分享」先例；Pawoo 同理 `launchUrlString(pawoo_url)`。
- 简介 comment 用 `SelectableHtml`（可选中富文本）——我们的 about tab 用 SelectableText，可选中但无链接解析；`CaptionRichText` 已有 pixiv 内链→内部路由、外链→outboundUrlOpener 的能力可复用。

## 3. PixEz 关注按钮

`UserFollowButton(id, followed, onPressed: follow(needPrivate:false), onConfirm: followWithRestrict)`——普通点按公开关注，需要私密时走确认流。我们 `FollowSwitchButton` 已等价：tap=toggle、long-press=范围 sheet（follow_switch_button.dart L156-159、L172-176）。差异：PixEz 在 overflow 菜单另放「悄悄关注」入口，我们把私密关注藏在长按——**长按无桌面等价路径**，§4.3/运行时矩阵要求鼠标可达，overflow 菜单项是天然等价位置。

## 4. Material 3 app bar 语义（m3.material.io/components/app-bars）

- medium flexible / large flexible app bar 滚动时**坍缩成 small app bar**，navigation icon 与 actions 槽在坍缩后保留；小断点上 trailing actions「collapse into an overflow menu」并在宽屏恢复可见——即「同一动作在展开/收起都有等价路径」正是 M3 flexible app bar 的结构语义，支持 §4.3 修复 IgnorePointer 可见不可点与 follow/share 缺口。
- On-scroll 容器填充 surface container 色以分离内容；我们的 collapsed toolbar 是自定义 Opacity 渐入，行为上对齐即可。
- 统计 chip 语义：M3 没有专门 stat 组件；按 a11y 惯例，可导航统计应是 `Semantics(button:true)`/InkWell/Material chip 级控件而非纯 Text——PixEz 用 `InkWell`/`DataCell.onTap` 正是此模式。值+标签成组读出（screen reader 一次焦点报「关注 123」）。
