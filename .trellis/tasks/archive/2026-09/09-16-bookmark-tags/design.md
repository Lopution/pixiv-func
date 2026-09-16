# 收藏标签体系 — 设计

## 现状

`BookmarkStore`/`BookmarkOp`/`BookmarkEntry` 已有确认态+revision 协议(非 optimistic)。`BookmarkRepository` 的 add/delete 不带标签。`BookmarkSwitchButton` 长按弹 restrict sheet,已收藏时长按被 suppress(`pending || bookmarked → null`)。`ProfileFeedKey`/`fetchBookmarks`/`validateBookmarksCursor` 无 tag 维度。

## Wire 契约(Shaft `API.kt` + PixEz `api_client.dart` 双源证实)

| 端点 | 形状 |
|---|---|
| `POST /v2/{illust,novel}/bookmark/add` | `tags[]` 参数;PixEz 以**空格拼接为单字段值**(我们的 `post(Map<String,String>)` 采用此形),Shaft 重复字段,两形同被接受。对已收藏作品 add = **覆盖**(Shaft 注释:同端点互覆非叠加) |
| `GET /v2/{illust,novel}/bookmark/detail` | `{bookmark_detail: {is_bookmarked, restrict, tags: [{name, is_registered}]}}` |
| `GET /v1/user/bookmark-tags/{illust,novel}` | `{bookmark_tags: [{name, count}], next_url}` 分页 |
| `GET /v1/user/bookmarks/illust` | 已有 `user_id`/`restrict`;新增 `tag` 过滤参数(Shaft `getUserLikeIllust`) |

## Core 扩展(`core/bookmark/`,父契约:不新建全局 store)

- 模型:`BookmarkTagFacet{name, isRegistered}`、`UserBookmarkTag{name, count}`、`BookmarkDetail{isBookmarked, restrict, tags}`、`UserBookmarkTagPage{tags, nextCursor}`。
- `BookmarkOp` 增 `tags: List<String>`;`beginAdd(key, restrict, {tags})`。
- `BookmarkEntry` 增 `tags`(确认态):add commit 写入 op.tags、delete commit 清空、`observeRemote` 不触碰(远端快照无 tags 维度)。
- `BookmarkRepository`:`add{Illust,Novel}` 增 `tags`(空格拼接进 `tags[]`,空则省略);`fetchBookmarkDetail(BookmarkKey)` 按 type 分发端点;`fetchUserBookmarkTags(userId, {entityType, restrict, cursor})` 走 `NextPageParser`(firstPage 白名单注册两个 bookmark-tags 端点)。
- `BookmarkActions.addWithRestrict` 增 `tags` 透传;toggle 不变(无标签快捷收藏)。
- `/v1/user/bookmarks/illust` 白名单补 `tag`;`fetchBookmarks`/`validateBookmarksCursor`/`ProfileFeedKey` 增 `bookmarkTag`(pin 进 firstPage query 与 cursor 校验)。

## UI

- **收藏 sheet 扩展**(`bookmark_switch_button.dart` `_showRestrictSheet`):restrict 选择器之下加标签区——已选 `InputChip`(可删)+ 建议 `FilterChip`(来自 `userBookmarkTagsProvider(entityType, restrict)` 第一页)+ 新标签 `TextField`(回车/逗号入列)。已收藏时长不再 suppress:打开编辑形态,`bookmarkDetailProvider(BookmarkKey)`(FutureProvider family)拉 detail 预填 restrict+tags,loading 时 sheet 内 spinner;确认仍走 `addWithRestrict`(覆盖语义)。
- **收藏标签页** `features/bookmark/bookmark_tags_page.dart`:segmented public/private 切换,tag 列表(name+count)分页加载;`UserBookmarkTagsController`(轻量 AsyncNotifier,非 PagedFeedController——非 illust-id feed)。
- **标签过滤收藏页** `BookmarkTagFeedPage`:Scaffold 包既有 `ProfileIllustFeed`,`ProfileFeedKey(kind:bookmarks, restrict, bookmarkTag)`;路由 `bookmarks/tags` + `bookmarks/tag/:tag?restrict=`(common routes + `openBookmarkTags`/`openBookmarkTagFeed`)。
- **入口**:own profile 的 bookmarks tab 头部操作区加标签图标按钮。

## 不做

- 标签编辑/删除标签本身(改名、删标签需逐作品重打标签,无端点;Shaft 同样没有)。
- novel 收藏 feed 的 tag 过滤页(端点存在但 PRD 未列 UI;repository 层对称就绪,UI 记 backlog)。
- 建议 chips 只取第一页(标签集合分页,建议语义不需要全量)。

## 测试

- repository:`tags[]` 空格拼接、detail 解析(illust+novel)、tags 分页+白名单、bookmarks `tag` 参数与 cursor pin。
- store:op.tags 携带、commit 写 entry、delete 清空、失败回滚不动 tags。
- sheet:建议 chips 渲染/选中、新标签输入、确认回传、已收藏 prefill(provider override)。
- tags 页/过滤页:列表渲染、restrict 切换、feedKey 含 tag 时 wire 带 `tag`。
- l10n 四语言。
