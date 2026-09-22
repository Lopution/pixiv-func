# Codebase: Spotlight 特辑（feed + 文章渲染管线）（W9 调研）

基线：`main@8067b2d`。对应 Astra #22（状态"确认"）。

## 入口与路由

- 入口：搜索首页 `search_page.dart` L68-74 `FilledButton.tonalIcon` → `openSpotlight(context)`。
- 路由（routes.dart）：`spotlight` L484-488 → `SpotlightFeedPage`；`spotlight/article/:articleId` L489-500 → `SpotlightArticlePage(articleId, articleUrl: query 'url')`。两条都在共享 branch 路由集内（每个 stack root 下都挂载）。
- facade：`openSpotlight` L1157-1159；`openSpotlightArticle` L1161-…（`{stackRoot}/spotlight/article/$id`，articleUrl 走 query）。

## spotlight_feed_page.dart（228 行）

- `_category` L27 是**页面本地 state**（`SpotlightCategory.all` 初值）——不进路由、不持久化，重进页面回默认。
- AppBar L33-54：title + `PreferredSize(52)` 底部 `SegmentedButton<SpotlightCategory>`（L39-51，3 段：全部/插画/漫画，`showSelectedIcon:false`）。
- 状态机 L55-145：`AsyncValue.when` loading→`FeedLoading`、error→`FeedError(title+retry)`；data 内再分 `showInitialError`→`FeedError`、`showInitialSpinner`→`FeedLoading`、空→`FeedEmpty`（L106-117）。
- 列表 L84-141：`PullToRefresh` + `NotificationListener` 距底 500 预载（L87-97）+ `SmoothWheelScroll` + `CustomScrollView`（`PageStorageKey`/`restorationId` 均按 category 名 L101-103）。
- 条目 `_SpotlightArticleTile` L151-227：`Card`+`InkWell`，88×66 缩略图 + title(2 行) + 子分类/日期行 + chevron。**整列 `SliverPadding(12,8,12,0)` 全宽铺**（L119-127），宽屏无上限。
- 尾部 `FeedTail` L128-137（loadMore 错误/加载/见底）。

## spotlight_article_page.dart（282 行）—— W9 主战场

- 数据 L36-40：`spotlightArticleStoreProvider[id]`（列表期快照，给标题兜底）+ `spotlightArticleBodyProvider((id,url))`。
- `_url` L33：`articleUrl ?? 'https://www.pixivision.net/a/$articleId'`。
- **AppBar L42-48：只有 title（entry?.title ?? '特辑'），零 actions —— 无分享、无"打开原文"**。
- body L49-83：loading→`FeedLoading`；error→`FeedError`（retry 用 `ref.invalidate` L55-57）；data→**全宽 `ListView`**（L59-61，`PageStorageKey('spotlight-article-$id')`，padding 16/8/16/32，**无 maxWidth**）。
- 文档头 L63-78：`body.title` headlineSmall + `body.description` bodySmall，均为普通 `Text`，**不可选择**。
- 块渲染 `_SpotlightBlockView` L87-176（StatefulWidget，因为要持有/dispose recognizer L99-107）：
  - `SpotlightHeading` L113-123：titleLarge/titleMedium/titleSmall 按 level——普通 `Text`。
  - `SpotlightParagraph` L124-132：`Text.rich` + `_linkSpans`——**不可选择**。
  - `SpotlightImage` L133-136 → `_ArticleImage` L181-194：`pximg.net` 走 `PixivImage`（referer 链），其余 `CachedNetworkImage`，`fit:contain`。
  - `SpotlightIllustCard` L137 → `_SpotlightIllustCardView` L196-259：Card+InkWell→`openIllust`；72×72 图、标题 2 行、作者名（`GestureDetector`→`openUser`，无 userId 时降级普通色 L233-248）。
- `_linkSpans` L144-175：每 segment 一个 `TextSpan`；带 href 的加 `TapGestureRecognizer`（`onTap`→`_openSpotlightLink`）。注释 L141-143 明确历史：WidgetSpan+GestureDetector 曾破坏选择/读屏，才改成 span 内 recognizer —— **改为可选择时必须保住链接可点**。
- `_openSpotlightLink` L261-281：相对 `https://www.pixivision.net/` resolve；`_pixivHosts`（pixiv.net/www.pixiv.net/www.pixivision.net）内 `/artworks/<id>`→`openIllust`、`/users/<id>`→`openUser`；其余 `launchUrl(externalApplication)`。

## 数据管线

- `spotlight_article_controller.dart`（18 行）：`spotlightArticleBodyProvider` = `FutureProvider.autoDispose.family((id,url))`，带 `languageTag` 请求 HTML，`parseSpotlightArticle` 解析。**body 不进 store**（注释 L9-10）。
- `article_parser.dart`（165 行）：`article > .am__body`（`_feature` 布局再进一层 L27-33）→ `_collectBlock` 递归（L60-103）：`.illust`→卡片、`p`→段落/`p>img`→图、`h1-h4`→标题、其他容器递归、裸文本兜底段落。`_paragraphSegments` L107-128 拍平 (text,href) run；`_parseIllustCard` L147-165。空 body / 无 article 元素抛 `ApiParseError`。
- `spotlight_feed_controller.dart`（53 行）、`spotlight_store.dart`（40 行）、`spotlight_models.dart`（129 行）：`SpotlightArticle`（id/title/thumbnailUrl/subcategoryLabel/publishDate/articleUrl）、`SpotlightBlock` sealed（Heading/Paragraph/Image/IllustCard）。

## 分享/外链基础设施（现成可复用）

- `lib/core/share/share_service.dart`：`SharePayload{title,author,url}`（L8-47，`text='{title} | {author} #Pixiv {url}'`，有 illust/novel/user 三个 factory；**裸构造是 public**，spotlight 可直接构造或加 `spotlight` factory——决策点）；`ShareOutcome.openedSheet/copiedToClipboard`（L50-57）；`SystemShareService` 失败自动落剪贴板（L82-86）；`shareOriginOf(context)` 给 iPad popover 锚点（L96-100）。
- 调用范式（illust_detail_page.dart L204-222）：`await share(...)` → `copiedToClipboard` 时 `showAppSnackBar(linkCopied)`。
- 现有 l10n：`cardActionShare`/"分享"、`linkCopied`/"链接已复制"、`profileShare`/`profileShareHint`/`profileShareClose`（用户页分享弹层）；**无通用 `share`/`openInBrowser` key**——Spotlight 的分享和"在浏览器打开原文"需要新增词条（4 语言 arb + lookup）。

## 测试基线

- `test/spotlight_article_test.dart`（253 行）：HTML fixture（`_articleHtml`/`_featureHtml`）喂 parser + provider；quality spec 记录了 autoDispose `.future` 需先挂 listen、CJK mock body 需 `Response.bytes`+utf8 两个坑。
- `test/spotlight_feed_test.dart`（192 行）+ `test/helpers/spotlight_world.dart`（124 行）。
- `test/spotlight_repository_test.dart`。

## 现状问题清单（Astra #22）

- 正文全宽无行长控制（ListView padding 16，1200dp 下每行 ~1168dp ≈ 150+ 拉丁字符）。
- 标题/描述/段落/小标题全部不可选择。
- AppBar 无分享/打开原文。
- 图片 `fit:contain` 全宽——宽屏下大图占满屏幕高度，正文流被巨型图打断（行长之外的布局问题）。
