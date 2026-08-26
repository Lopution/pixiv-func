# 复刻发现、搜索与反向搜图

## Goal

作为中间父任务，协调 Ranking、New、Search 和 Reverse Image 四个独立功能，使它们复用已验证的实体、分页、收藏和 Android 输入边界。

## Confirmed Facts

- 该任务已拆为 ranking-feed、new-content-feeds、search-catalog、reverse-image-search 四个叶子任务。
- beta56 对应源码分别位于 lib/pages/ranking、new、search、search_guide 和 search/result/image。
- 第一条 Login→Recommended→Detail→Bookmark 链必须先完成，随后才能扩展这些发现入口。

## Dependencies

- 08-26-bookmark-state-sync 完成第一条链。
- 08-26-android-platform-parity 完成外部图片 intent 基础。

## Requirements

- R1: 本任务不直接实现产品代码，只维护四个叶子任务的范围、顺序、共享契约和集成验收。
- R2: Ranking、New、Search 的作品必须复用 IllustStore、BookmarkStore 和 PagedFeedController，不创建页面私有实体副本。
- R3: SearchCatalog 先于 ReverseImageSearch 提供结果路由；ReverseImageSearch 复用 Illust Detail/Store 展示 Pixiv 命中。
- R4: 四个叶子任务分别完成源码研究、用户审批、实现、检查、提交和归档；任何时效服务失效在 owning task 明确阻塞。
- R5: 中间父任务完成前执行 Home tabs、跨入口状态同步、导航返回和账号切换集成回归。

## Acceptance Criteria

- [ ] 四个叶子任务均归档，task tree、依赖和证据完整。
- [ ] Home Ranking/New/Search 可从原位置进入，tab/scroll/query 状态按 beta56 保持。
- [ ] 同一作品从 Ranking/New/Search/Reverse Image 进入详情时实体与收藏状态一致。
- [ ] 账号切换、离线/错误、返回和深链/Android SEND 集成无串状态或伪成功。
- [ ] 全量 analyze/test/debug build 及 discovery 集成 Widget/真机验证通过。

## Out of Scope

- User/Profile、Comments、Novel reader 的内部实现。
- 更换 Replica Home 导航模型。
- 新增原版没有的发现算法或推荐过滤。

## Risks and Deferred Items

- Reverse image 外部服务与 Search API 时效性高；叶子任务必须实时核验，不能把失败隐藏为空结果。

## Source Anchors

- beta56 lib/pages/ranking、lib/pages/new、lib/pages/search、lib/pages/search_guide
- 父任务 R8 与四个叶子任务 artifacts

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
