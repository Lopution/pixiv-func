# Feed 基础设施：快照冷启动与离线动作队列

## Goal

冷启动直出上次会话内容；离线/失败变更持久化重放不丢操作。

## Requirements

- feed 快照：`{feedKey, accountId, ids, entities, cursor, ts}` 持久化（SQLite）；provider 构造时同步恢复为 initial IDs 后正常刷新；过期阈值 + 容量上限显式定义
- 离线动作队列：mutation envelope 持久化表 + 连接恢复后重放，重放走同一 repository 调用；与 revision/账号边界协议兼容
- 可观测：丢弃/重放/恢复计数走既有 telemetry

## Acceptance Criteria

- [ ] 冷启动首帧为上次内容，后台静默刷新替换；快照过期/损坏安全回退
- [ ] 断网下收藏/关注入队，恢复后重放成功且 UI 确认态一致；重复重放幂等
- [ ] fake clock/transport 测试：恢复、过期、重放、边界

## References

- Shaft：`snapshot/`、`feeds/cache/RoomFeedCacheBackend.kt`、`actionqueue/` 模块
- 本仓：`core/paging/`、`core/mutation/`、`core/history/history_database.dart`

## Dependencies

建议在主要 feed 面稳定后做（wave 4）。
