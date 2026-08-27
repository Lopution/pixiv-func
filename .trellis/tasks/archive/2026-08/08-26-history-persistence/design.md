# 实现浏览与 Pixiv History — Design

## Objective

以单一数据库生命周期和可见性计时重建本地/Pixiv浏览历史，保持beta56页面与删除交互而避免全JSON和每秒Timer。

## Architecture and Boundaries

- HistoryDatabase由Riverpod keepAlive repository拥有单实例和versioned migrations。
- HistoryRecord只存typed ID、timestamps、duration/account和minimal snapshot；实体详情从stores/repository补全。
- HistoryTracker以route/content key管理visibility sessions和Stopwatch，foreground/background触发pause/resume。
- PixivHistoryOutbox合并未提交duration并幂等flush，和本地记录事务边界分离。

## Data Flow

visible content → tracker start/pause/stop → local upsert + optional account outbox → async Pixiv submit；History page → indexed page → hydrate shared entities → delete/clear transaction。

## Compatibility, Security, and Migration

- 旧仓库当前没有新History DB，无需自动迁移beta56 DB；schema从v1即采用现代结构。
- Settings关闭不销毁用户数据，除非用户明确清空。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。

