# 复刻 Ranking 榜单

## Goal

复刻 beta56 的 11 类 Ranking tabs、双列作品流和分页状态，并复用共享作品/收藏模型。

## Confirmed Facts

- beta56 RankingPage 使用 RankingMode.values，依次展示日榜、日榜 R18、男性/女性及 R18、周榜/周 R18/原创/新人和月榜，共 11 项。
- 原版为可滚动 TabBar、双列 waterfall 卡片，每个 mode 拥有独立 source/cursor。
- Recommended 已提供 IllustStore、BookmarkStore、PagedFeedController 和 card 基础。

## Dependencies

- 08-26-bookmark-state-sync 完成。
- 父 discovery task 的共享契约已审阅。

## Requirements

- R1: 按 beta56 顺序和本地化文本实现 11 个 Ranking tabs，保持可滚动标题、指示器和原版信息密度。
- R2: 每个 mode 使用独立、懒加载、账号隔离的 paging state、cursor、scroll position 和错误状态。
- R3: RankingRepository 使用当前 endpoint/mode 映射与 allowlisted next cursor，复用 Illust mapper/store。
- R4: 双列卡片复用 Recommended preview 和 BookmarkStore；点击进入同一 Detail，状态不得复制。
- R5: 切换 tab 不重复首屏请求；refresh 只影响当前 tab，账号切换清理所有 mode 的账号数据。
- R6: 处理模式无权限/空数据、initial/load-more error、取消和 API mode 变化，不能把未知 mode 静默映射到日榜。

## Acceptance Criteria

- [ ] 11 个 tabs 顺序、标签和 mode 请求与 beta56 对应。
- [ ] 每个 tab 首次懒加载，切换往返保留列表/scroll/cursor；refresh/load-more 独立。
- [ ] 同一作品在 Ranking/Recommended/Detail 的实体和 bookmark 状态同步。
- [ ] 恶意 cursor、未知 mode、空/错误/取消有明确状态且不串到其他 tab。
- [ ] controller/widget/integration tests、analyze、全量 test、debug build 和真实 API 多 mode 验证通过。

## Out of Scope

- 自定义榜单过滤。
- Search filters。
- 重排或合并 beta56 tabs。

## Risks and Deferred Items

- 当前 Pixiv 可能调整 R18 mode 可用性；不可用应按真实错误/空状态呈现，不伪造数据。

## Source Anchors

- beta56 lib/pages/ranking/ranking.dart、lib/pages/ranking/content/source.dart
- 共享 IllustStore/PagedFeed/Bookmark/Card contracts

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
