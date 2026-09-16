# 执行计划:内容扩展(illust series + pixivision)

> 一个勾一个 commit,提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] `feat(series): 系列模型与 wire 契约`——`core/series/`:IllustSeriesEntity/Context 模型 + IllustSeriesStore + PixivSeriesRepository(fetchSeriesWorks/fetchIllustSeriesContext/fetchUserSeries/双 cursor 校验)+ NextPageParser 白名单(/v1/illust/series、/v1/user/illust-series)+ repository/store 单测
- [ ] `feat(series): 系列 feed 与系列页`——_IllustSeriesFeedController/_UserSeriesFeedController + illustSeriesContextProvider + IllustSeriesPage(头+IllustFeedGrid+触底/刷新)+ UserSeriesFeed 卡片网格 widget + `series/:seriesId` 路由与 openIllustSeries + feed/widget 测试
- [ ] `feat(series): 详情页与用户页入口`——IllustSeriesSection sliver 接入详情页 + ProfileWorkSection{illust,manga,novel,series} 替换 _workType + tabs delegate 第 4 chip + UserSeriesFeed 挂入 _ProfileTabBody + l10n 四语言 + widget 测试
- [ ] `feat(spotlight): 文章模型与列表 feed`——`core/spotlight/`:SpotlightArticle 模型 + SpotlightArticleStore + fetchArticles/validateArticlesCursor + 白名单(/v1/spotlight/articles)+ _SpotlightFeedController + repository/feed 测试
- [ ] `feat(spotlight): 文章解析与应用内详情`——pub add html + article_parser(常规/_feature 变体、illust 卡、段落链接、ApiParseError)+ spotlightArticleBodyProvider + SpotlightArticlePage(blocks 渲染、host 分流图片、站内/外链接分发)+ `spotlight/article/:articleId` 路由 + parser/widget 测试
- [ ] `feat(spotlight): 列表页与搜索页入口`——SpotlightFeedPage(category SegmentedButton+刷新+触底)+ `spotlight` 路由 + openSpotlight/openSpotlightArticle facade + search_page 入口行 + l10n 四语言 + widget 测试
- [ ] `chore(09-16): journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`(0 issue)
- `dart format --set-exit-if-changed lib test`
- `flutter test`(全绿,含 layering_test)
- `git diff --check`

## 回滚点

每勾独立可 revert;series/spotlight 均为新增域与新页面,删除文件+还原 routes/search_page/user_page/detail_page 接线即还原;html 依赖随 parser 一同回退。
