# Codebase: tab/feed 参数流、公开/私密范围传递断点、分页状态

## 1. 范围/类型参数模型

- `UserRestrict {public, private}`（lib/core/user/user_repository.dart L370-373，`wireValue`）。
- `BookmarkRestrict {public, private}`（lib/core/bookmark/bookmark_models.dart L11，`bookmarkRestrictWire` L14-15）——与 `UserRestrict` 平行的第二个枚举，**跨边界手工映射**。
- `FollowRestrict`（core/user/follow_models.dart）——第三个平行枚举，follow sheet 使用。
- `ProfileFeedKey`（lib/core/profile/profile_models.dart L26-62）：`userId + kind + workType + restrict + bookmarkTag`，`==`/hashCode 全覆盖，直接作 provider family 参数、`PageStorageKey`（profile_illust_feed.dart L70）与 `restorationId`（L71）。
- `ProfileWorkSection {illust, manga, novel, series}`（L10），`wireWorkType`（L15-20）：series 回落 illust 但实际走 `/v1/user/illust-series`。

## 2. Feed 控制器（lib/core/profile/profile_feed_controller.dart）

- `_ProfileIllustFeedController`（L14-91）：`feedKey='profile:illust:$key'`；`localFilterEnabled` 仅 `kind==work && workType!=novel`（L29-30，C9 发现流过滤；收藏/关注列表不过滤）；`filterMinVisible=24`、`filterMaxRefillPages=3`（L33-36）。fetch：work → `fetchWorks(userId, type, cursor)`；其他 → `fetchBookmarks(userId, restrict, tag, cursor)`（L48-61）。游标校验分别走 `validateWorksCursor`/`validateBookmarksCursor`（L74-90），把 restrict+tag 钉进校验。
- `_ProfileUserFeedController`（L95-144）：relation = following/fans/myPixiv；`restrict` 进 `fetchRelation`，但仓库只在 `following` 时上线（user_repository.dart L246 `if (relation == UserRelation.following)`）。
- provider family：`profileIllustFeedProvider` / `profileUserFeedProvider`（L146-158）。novel 分支不在此 controller——`userNovelFeedProvider`（core/novel/novel_feed_controller.dart），profile 小说页只带 userId，**无 restrict 维度**。
- 全部继承 `PagedFeedController`（core/paging/paged_feed_controller.dart）：`PagedFeedState` 三相（initial/refresh/loadMore）+ `exhausted` + `showLoadMoreSpinner/showLoadMoreError`（L60-68）——FeedTail 已能区分可加载/加载中/失败/到底。

## 3. 页面侧 feed 现状

| 文件 | 结构 | 尾部/空态 |
|---|---|---|
| profile_illust_feed.dart L49-119 | PullToRefresh(isNested)+NotificationListener 加载更多（extentAfter<1.2×viewport）+CustomScrollView(PageStorageKey(feedKey))+IllustFeedGrid+FeedTail | FeedTail 有 error/retry，但 `endMessage` 未传（L107-114）→ 到底无文字标记 |
| profile_novel_feed.dart L48-102 | 同上结构，SliverList+NovelCard；`PageStorageKey('profile-novel-$userId')` 不含 feedKey/restrict | 同上，endMessage 未传（L86-94） |
| profile_user_feed.dart L53-108 | 同上，`_UserPreviewTile`（openUser + FollowSwitchButton compact） | 同上 |
| user_series_feed.dart L56-112 | 同上，系列封面卡→openIllustSeries | 同上 |
| bookmark_tag_feed_page.dart L32-46 | 普通 Scaffold+AppBar(title: Text(tag))，body=`ProfileIllustFeed`（feedKey 带 bookmarkTag+restrict） | 顶部只有标签名，**无范围可见性、无本地过滤、无返回上下文之外的信息** |

注意：`ProfileIllustFeed` heroScope（L101-104）= `profile:<uid>:<kind>:<workType>:<restrict>`，**不含 bookmarkTag**——同一用户公开收藏流与标签流同 scope，若两页同时挂载同 ID 会撞 Hero tag（详见 risks）。

## 4. 收藏标签链路（范围传递断点）

```
/me 收藏 tab (UserRestrict _restrict, user_page.dart L100)
  └─ onOpenBookmarkTags → openBookmarkTags(context)   [routes.dart L1209-1211]
       │  不接收任何参数；route 'bookmarks/tags' 也无 query（routes.dart L585-589）
       ▼
BookmarkTagsPage（lib/features/bookmark/bookmark_tags_page.dart）
  - 本地 _restrict = BookmarkRestrict.public（L23）★ 断点：作者页当前范围未传入
  - SegmentedButton 切换（L37-51）；query=(illust, _restrict)（L25）
  - userBookmarkTagsProvider（bookmark_tags_controller.dart L113-118）：
    非 PagedFeedController，自管 nextUrl/loadingMore/loadMoreError（L11-44）
  - _TagList（L78-138）：tail item 只在 `hasMore||loadingMore` 时出现且恒为 spinner（L112-125）；
    `state.loadMoreError` 存了但**从不渲染**（grep 确认无读取点）→ 失败无尾部重试
  - 无本地过滤/搜索框
  - onTap → openBookmarkTagFeed(context, tag, restrict)（L132-133）
       ▼
route 'bookmarks/tag?tag=&restrict='（routes.dart L590-603，restrict query 解析正确）
       ▼
BookmarkTagFeedPage → ProfileIllustFeed（范围经 feedKey 进 API）
```

结论：范围只在 `书签标签页 → 标签 feed` 一跳里通过 route param 传递；`作者页 → 标签页` 断掉（BookmarkTagsPage 无入参、route 无 query、openBookmarkTags 无参数）。PixEz 的等价实现（UserBookmarkTagPage）把 tag+restrict 作为 `Navigator.pop` 结果回传给 bookmark 页，范围始终在同一上下文内流动（见 external 文档）。

## 5. 路由与恢复

- `openUser`（routes.dart L1149-1151）：`/<stackRoot>/user/<id>`，所有分支通用；`/me` 是 root navigator 上的 app 级路由（L932-941），其下挂 `_commonBranchRoutes`，所以从 /me 内可以继续 push user/bookmarks 等。
- `openProfileEdit`（L1409-1411）：`/<root>/profile/<userId>/edit`（route 注册 L502-507）。
- 书签两个路由在 commonBranchRoutes（L585-603），即每个分支都能 push；`restrict` 仅 bookmark tag feed 路由有 durable query，`bookmarks/tags` 没有——**恢复后 BookmarkTagsPage 始终 public**，这与 §5.1 durable 恢复等级声明有关（叶子需声明：当前 restrict 连内存级都未传递）。
- 各 feed `restorationId` 已存在；`_ProfileTabBody` 的 keep-alive 保证切 tab 不丢滚动位置，但换 restrict/section 因 feedKey 变化重建（ValueKey）。

## 6. l10n 现状

已有 key（lookup.dart 已映射）：restrictPublic/restrictPrivate/restrictSelector（后者当前无消费点）、bookmarkTags、profileShare/profileShareHint、copyLink、cardActionShare、linkCopied、profileEdit*、downloadAuthorWorks*、followUser/followed、profileSeries 等。
**缺失**：社交链接「打开链接」类动词（无 openLink/openExternally key）；统计导航若需独立文案也要新增。新增 key 需同步 arb + lookup.dart 表。
