# 复刻 Ranking 榜单 — Design

## Objective

复刻 beta56 的 11 类 Ranking tabs、双列作品流和分页状态，并复用共享作品/收藏模型。

## Architecture and Boundaries

- RankingMode domain enum 显式映射 API value 和本地化 key；不依赖外部 enum 顺序。
- RankingController 管 mode→PagedFeedController 映射，按需创建并在账号切换统一 dispose。
- RankingPage 只管理 tab/scroll，卡片通过 ID watch shared stores。

## Data Flow

select mode → lazy controller → RankingRepository → page/map/upsert → mode ID list → shared cards；tab switch 保存 controller/scroll。

## Compatibility, Security, and Migration

- beta56 tab 顺序冻结；API mode 差异由 mapper 适配。
- 后续 API 取消某 mode 时保留 tab并显示明确不可用，除非用户批准 UX 改动。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚已独立提交的叶子任务。

