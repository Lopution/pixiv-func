# 内容扩展:插画系列与 pixivision — 设计

## 现状

feed 一律 `PagedFeedController`(ids + `FeedCommitGate` + cursor allowlist);`related_illust` 是最近模板。
`IllustStore`/`NovelStore` 为共享实体,无 series、spotlight 域。`NextPageParser._kNextPageEndpoints` 白名单
登记分页端点。详情页 sliver 序为 image → `InfoBlock` → `RelatedIllustsSlivers`;用户页 work tab 经
`ReplicaProfileTabsDelegate` 的 `_workType`(illust/manga/novel)chip 选择。`PixivImage` 仅服务 i.pximg.net
(强制 Referer);第三方流量走 `thirdPartyHttpClientProvider`。路由统一进 `_commonBranchRoutes` +
`open*` 门面(`_currentStackRoot`)。项目暂无 HTML 解析依赖。

## Wire 契约(Shaft `API.kt`/`Models.kt` + PixEz `api_client.dart`/models 双源证实)

| 端点 | 形状 |
|---|---|
| `GET /v1/illust/series?illust_series_id=<id>` | `{illust_series_detail:{id,title,caption,user,cover_image_urls,series_work_count,watchlist_added,is_concluded,display_text}, illust_series_first_illust, illust_series_latest_illust, illusts[], next_url}`;翻页游标参数 `last_order`(随 next_url 下发);illusts 最新在前 |
| `GET /v1/illust-series/illust?illust_id=<id>` | `{illust_series_context:{content_order, prev:Illust?, next:Illust?}, illust_series_detail:{...}}`;非系列作品 context/detail 为空 |
| `GET /v1/user/illust-series?user_id=<id>` | `{illust_series_details:[{id,title,caption,cover_image_urls,series_work_count,user,create_date,width,height}], next_url}` |
| `GET /v1/spotlight/articles?filter=for_android&category=<cat>` | `{spotlight_articles:[{id,title,pure_title,thumbnail,article_url,publish_date,category,subcategory_label}], next_url}`;category ∈ {all,illust,manga} |
| `GET <article_url>`(www.pixivision.net) | HTML;`article .am__body` 子节点结构化;`_feature` 变体需下钻一层;`.illust` 卡含 `/artworks/<id>` 链接 + h3 标题 + img + 用户 p;`article header` 含描述(PixEz soup_store 取证) |

## Core 扩展

### `lib/core/series/`(新域)

- `series_models.dart`:`IllustSeriesEntity{id,title,caption,userId,userName,coverUrl,workCount,watchlistAdded,isConcluded,latestContentId?}`(detail 与 user-series 条目共用一实体);`IllustSeriesContext{seriesId,contentOrder,prevIllustId?,nextIllustId?}`。
- `series_store.dart`:`IllustSeriesStore`(`Notifier<Map<int,IllustSeriesEntity>>`,account-scoped reset;merge 规则:新值非空覆盖,空字段保留旧值)。
- `series_repository.dart` `PixivSeriesRepository`:
  - `fetchSeriesWorks(seriesId,{cursor,cancelToken})` → `SeriesWorksPage{detail?,illusts,nextUrl}`;repository 不碰 store,commit 回调统一 merge。
  - `fetchIllustSeriesContext(illustId,{cancelToken})` → `{detail?,context?}`(非系列返回 null context)。
  - `fetchUserSeries(userId,{cursor,cancelToken})` → `UserSeriesPage{series,nextUrl}`。
  - `validateSeriesCursor(seriesId,cursor)` / `validateUserSeriesCursor(userId,cursor)`(pin path + 身份参数,同 `validateWorksCursor` 模式)。
- `series_feed_controller.dart`:`_IllustSeriesFeedController(seriesId)`(PagedFeedController:illust ids + `incomingIllusts` + commit 同时 merge `IllustStore`(bookmarkRevision 协议)与 `IllustSeriesStore.detail`);`_UserSeriesFeedController(userId)`(series ids + commit merge `IllustSeriesStore`)。
- `illust_series_context_controller.dart`:`illustSeriesContextProvider(illustId)` FamilyAsyncNotifier —— build 内 fetch 后 merge detail→seriesStore、prev/next→IllustStore,返回 `IllustSeriesContext?`(单次 fetch 无 commit gate,notifier 是合法写路径)。
- 白名单登记:`/v1/illust/series`{filter,illust_series_id,last_order}、`/v1/user/illust-series`{filter,user_id,offset}。

### `lib/core/spotlight/`(新域)

- `spotlight_models.dart`:`SpotlightArticle{id,title,pureTitle,thumbnailUrl,articleUrl,publishDate,category,subcategoryLabel}`;`SpotlightArticlePage{articles,nextUrl}`;`SpotlightArticleBody{title,description?,blocks}`;sealed `SpotlightBlock`:`SpotlightParagraph{segments:List<(text,href?)>}`、`SpotlightHeading{text,level}`、`SpotlightImage{url}`、`SpotlightIllustCard{illustId,imageUrl?,title,userName?,userId?}`。
- `spotlight_repository.dart`:app-api `fetchArticles({category,cursor,cancelToken})` + `validateArticlesCursor(category,cursor)`;`fetchArticleHtml(url,{cancelToken})` 走 `thirdPartyHttpClientProvider`(desktop UA + `Referer: https://www.pixivision.net/` + `Accept-Language` 跟随应用语言)。
- `spotlight_store.dart`:`SpotlightArticleStore`(account-scoped map,`mergeAll`/`get`)。
- `spotlight_feed_controller.dart`:`_SpotlightFeedController(category)` PagedFeedController(article ids + commit merge SpotlightArticleStore)。
- `article_parser.dart`:`parseSpotlightArticle(String html)` 纯函数(`package:html`,新增依赖):`article .am__body` 子节点 → blocks;`_feature` 变体下钻;`.illust` 卡提取 artworks 链接/h3/img/user;p→段落段(href 保留);h2-h4→heading;独立 img→image;空 body/无 article → `ApiParseError`。
- `spotlight_article_controller.dart`:`spotlightArticleBodyProvider((id,url))` FutureProvider → fetchHtml → parse(文章体无共享消费,不进 store)。
- 白名单登记:`/v1/spotlight/articles`{filter,category,offset}。

## UI

- `features/series/illust_series_page.dart`:AppBar + 系列头(cover/title/user→`openUser`/全 N 话/caption)+ `IllustFeedGrid` + `FeedTail` + 刷新/触底 loadMore(复用 detail 页 -500 阈值模式)。
- `features/illust/detail/widgets/illust_series_section.dart`:详情页 image 与 `InfoBlock` 之间插入 `IllustSeriesSection` sliver——`illustSeriesContextProvider` 有值时显示系列卡(系列名/第 N 话/上一话·下一话按钮 `openIllust`/整卡 `openIllustSeries`);null/错误静默不显示。
- `features/profile/user_series_feed.dart`:series 卡片网格(cover+title+全 N 话)接 `_UserSeriesFeedController`;user_page work tab 的 selector 增第 4 chip「系列」——`_workType` 升级为 `ProfileWorkSection{illust,manga,novel,series}`(定义于 `core/profile/profile_models.dart`,与 `UserWorkType` 解耦,wire `type` 参数不受影响);`_ProfileTabBody` 按 section 分发。
- `features/spotlight/spotlight_feed_page.dart`:AppBar + category SegmentedButton(all/illust/manga)+ 文章行(缩略图/标题/日期/label)+ 刷新 + 触底 loadMore + FeedTail/Error。
- `features/spotlight/spotlight_article_page.dart`:blocks 渲染;IllustCard→`openIllust`;段落内 `/artworks/`→openIllust、`/users/`→openUser、其余→`url_launcher`;图片按 host 分流(pximg→PixivImage,其余→`CachedNetworkImage`——pixivision CDN 不吃 pximg Referer)。
- `features/search/search_page.dart`:反向搜图按钮之下加 pixivision 入口行(特辑图标)→`openSpotlight`。
- `routes.dart`:`series/:seriesId`、`spotlight`、`spotlight/article/:articleId?url=` 进 `_commonBranchRoutes`;facade `openIllustSeries`/`openSpotlight`/`openSpotlightArticle`(url 缺省回退 `https://www.pixivision.net/a/<id>`)。

## 不做

- watchlist add/delete(归 `watchlist-local-library` child);novel series 页;spotlight 其余分类与文章缓存;系列内排序/编辑操作。

## 测试

- repository:四端点 wire path/query、响应解析(含 context null)、cursor pin(改 id 拒绝)、`fetchArticleHtml` header 断言。
- parser:常规/`_feature` body、illust 卡字段、artworks/users 段落链接、畸形 HTML → ApiParseError。
- feed:双 controller 首末页/取消/cursor 拒绝;store merge 空字段保留。
- widget:系列区渲染+prev/next 导航回调、work selector 系列 chip、spotlight 列表与文章页 blocks。
- l10n 四语言。

## 风险

- `html` 新依赖(`flutter pub add html`,pin ^0.15.x);pixivision.net 可达性独立错误态;`_feature` 变体结构差异以 PixEz soup 为基准。
