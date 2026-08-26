# 实现推荐作品流与可靠分页 — Design

## Objective

用真实 Pixiv API 打通登录后的 Recommended Illust 首页，并建立后续 Ranking/Search 共用的实体与分页基础。

## Architecture and Boundaries

- core/entity/illust_entity.dart 和 IllustStore 提供账号隔离的 canonical entity map。
- core/paging/page.dart 表达 API page；PagedFeedState 分离 IDs、cursor 和 operation-specific status。
- RecommendedIllustRepository 只负责 endpoint/mapper，controller 负责 refresh/loadMore/cancel/dedupe，Widget 只渲染状态。
- ScrollController 与 feed provider 生命周期绑定 Home tab，而不是页面每次 build 重建。

## Data Flow

Home Recommended → controller initial load → repository/PixivHttpClient → map entities → IllustStore upsert → ordered ID state → cards watch IDs；refresh/loadMore 各自维护请求代次和取消。

## Compatibility, Security, and Migration

- 后续 Ranking/Search 复用 IllustStore/Paging，不复制实体状态。
- beta56 arbitrary nextUrl 行为改为 allowlisted endpoint/query，用户可见分页保持不变。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

