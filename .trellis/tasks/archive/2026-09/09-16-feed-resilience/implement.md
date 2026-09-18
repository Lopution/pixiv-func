# 执行计划：Feed 快照冷启动与离线动作队列

> 一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [ ] `feat(feeds): feed_snapshots 存储与 FeedSnapshotStore`——`feeds.db`/`feed_snapshots` 表 + 读写/LRU 逐出/过期判定 + codec 接口 + sqflite_ffi 单测
- [ ] `feat(feeds): PagedFeedController 快照冷启动`——build 首查快照直出 + 后台 refresh 调度 + 提交后写快照 + illust/novel codec + controller/store 测试（恢复/过期/账号边界）
- [ ] `feat(feeds): ActionQueue 持久化队列`——`action_queue` 表 + enqueue/coalesce/串行 drain/RetryScope 分档/maxAttempts + telemetry + 单测
- [ ] `feat(feeds): 收藏与关注离线入队重放`——bookmark/follow store 网络失败入队 + handler 走同一 repository + pump 触发点（启动/前台/成功调用）+ 集成测试（重放幂等/coalesce/冷却）
- [ ] `chore(09-16): feed-resilience journal + 收尾`——add_session 记录

## 验证命令

- `flutter analyze`（0 issue）
- `dart format --set-exit-if-changed lib test`
- `flutter test test/feed_snapshot_test.dart test/action_queue_test.dart` + paged feed / bookmark / follow 相关测试
- `flutter test`（全量）
- `git diff --check`

## 回滚点

每勾独立可 revert。快照路径是 build() 前置分支——摘掉 `snapshotCodec` 即全回退；队列只在 mutation 失败分支入队，摘掉 enqueue 调用即回同步失败语义。

## 风险

- 快照 JSON 体积：60 ids × 实体 ~200KB/feed，64 feed ~13MB——接受范围内，LRU 兜底；实测过大再降容量。
- `build()` 里 await 快照读增加首帧延迟：本地 SQLite 单行读 <10ms，且只在有快照时提前返回——无快照路径不多等。
- Riverpod 3 build 重试坑（quality-guidelines 已录）：快照读失败走正常首载，不抛进 build。
- 重放风暴：账号刚登录队列积压时逐条串行 + gapMs 节流，429 即整队冷却，不会打爆服务器。
