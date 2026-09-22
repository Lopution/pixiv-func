# 代码调研：系列分页游标 vs 阅读进度 — `lib/features/series/` + `lib/core/series/` + `lib/core/watchlist/`

对应设计 §4.4/§5.2：**更新游标不是阅读进度**；“继续阅读”必须建立在真实排序能力与章节身份之上。行号已核对。

## 1. 分页数据源：`last_order` 游标，不是 offset

- `PixivSeriesRepository`（series_repository.dart L64 注释）：系列作品分页用 Pixiv `next_url` 携带的 **`last_order`** 游标，不是 offset。
- `SeriesIllustPage.nextUrl` / `UserSeriesPage.nextUrl`（L16-30）：每页响应带回下一个游标。
- 游标校验：L186-192 拒绝 path 不匹配或与活动 feed 参数不一致的 `next_url`——防串页。
- `latest_content_id` 解析 L225、`readNextUrl` L233/301。
- `_IllustSeriesFeedController`（series_feed_controller.dart）：`nextCursor: page.nextUrl`（L37/84），把作品 merge 进 `IllustStore`、系列详情 merge 进 `IllustSeriesStore`，feed 只持有序 ID（`PagedFeedState`），符合 spec “feed 存 ID、canonical 实体在 store”。

## 2. `latestContentId` 的语义

- `IllustSeriesEntity.latestContentId`（series_models.dart L24/46）：**服务端给的最新作品 ID**，可空（L11 注释：payload 不带时保持 null），merge 时保留旧值（L61、series_store.dart L56）。
- 它是“更新检测”字段，不是阅读位置。

## 3. WatchlistReadCursor —— 本地“已看过最新”游标

`lib/core/watchlist/watchlist_store.dart` L249-275：
- 注释 L249-251：local "already seen" cursor，记录本设备上打开过的**最新 `latest_content_id`**，按 (accountId, seriesType, seriesId) 分键；从未打开=未见。
- `markSeen`（L268-275）：**只前进**——`current >= latestContentId` 直接 return（L274），写 SharedPreferences。
- 用途：watchlist/系列页的“有新内容”角标；**不是章节级阅读位置**。

## 4. IllustSeriesPage 的 markSeen 时机

- `illust_series_page.dart` L151-160：页面拿到 `detail.latestContentId` 后 `markSeen(latest)`——**打开系列页即把“已看到最新”推进到最新一话**，与用户实际读到哪一章无关。

## 5. 详情页内系列导航 — `IllustSeriesSection`（widgets/illust_series_section.dart）

- L13 注释：作品属于系列时显示卡片，含上一话/下一话导航。
- 数据：`illustSeriesStoreProvider`（L27）+ 该作品的 series context（`/v1/illust-series/illust`）。
- 导航：L98-109 两个 `IconButton`，`prevIllustId`/`nextIllustId` 为空时 `onPressed: null`（禁用态）；点击 `openNeighbour → openIllust`（L49-）带已预热实体；卡片本体 `onTap → openIllustSeries(detail.id)`（L60）。

## 6. “继续阅读”可行语义（供 implementation-draft 决策）

已有可用材料：
- 系列 feed 有真实顺序（`last_order` 游标保序）+ 每话有稳定 ID（`IllustSeriesFeedController` 的 ordered ids）；
- `WatchlistReadCursor` 只有“见过的最新 ID”，**无法回答“读到第几话”**；
- Pixiv `illust-series/illust` 上下文能给当前作品的 `prev/next`，能支持“从第 1 话/下一话开始读”。

边界（设计 §5.2 明确）：**不得把更新游标伪装成阅读进度**。若 W4 要落地“继续阅读”，只有两个诚实选项：
a) 不新增 schema：按钮语义限定为“从第 1 话开始 / 打开最新一话 / 上一话·下一话导航”，文案不得暗示续读位置；
b) 新增章节级进度 schema：按 account+series 存 lastReadContentId + 迁移/默认值/回滚文档——这超出本 leaf 的最小范围，建议留给后续任务（见 risks.md）。
