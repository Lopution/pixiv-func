# 实现浏览与 Pixiv History

## Goal

以单一数据库生命周期和可见性计时重建本地/Pixiv浏览历史，保持beta56页面与删除交互而避免全JSON和每秒Timer。

## Confirmed Facts

- beta56 HistoryDB每次操作openDatabase、全局锁，并存完整Illust JSON。
- 父PRD要求singleton repository、single DB lifecycle、indexed compact schema、async transaction。
- Pixiv history计时不得Timer.periodic(1 second)，应使用visibility、Stopwatch和lifecycle。

## Dependencies

- 08-26-settings-parity提供enableHistory/enablePixivHistory。
- IllustStore/Detail和NovelReader提供typed view events。

## Requirements

- R1: 建立应用级HistoryDatabase/Repository单一open/close生命周期，schema版本化且migration可测试。
- R2: 本地记录使用content type+ID、lastViewedAt、minimal display snapshot/version等紧凑字段，不存完整巨大API JSON。
- R3: 按内容ID upsert并按最近访问排序，提供分页、count、删除单项、清空和事务接口及索引。
- R4: History页面保持beta56卡片/分页、long press约35%确认sheet、删除/清空反馈，并通过shared store补全实体。
- R5: enable local history关闭时不写新本地记录；现有记录处理策略明确且不自动删除。
- R6: Pixiv history使用页面visibility+Stopwatch+foreground lifecycle累计有效可见时长，以single-shot/离开事件提交，不每秒轮询。
- R7: 网络提交失败使用有界重试/下次合并，按account隔离并避免重复计时；退出/崩溃恢复策略明确。
- R8: DB损坏/migration失败、实体已删除、离线、账号切换和并发读写返回明确状态。

## Acceptance Criteria

- [ ] 应用生命周期内数据库只打开一次，所有操作通过repository/transaction，无每操作open/global mutex。
- [ ] 同一作品重复访问upsert到顶部，分页/删除/清空/count和索引查询正确。
- [ ] DB不含完整API JSON或token；migration/损坏测试有明确恢复或blocker。
- [ ] local/Pixiv history toggles即时生效；Stopwatch只累计可见前台时间且无每秒Timer。
- [ ] 提交重试、离线、进程/账号切换不重复累计或串账号。
- [ ] analyze、全量test、debug build、DB migration及真机前后台计时验证通过。

## Out of Scope

- 云同步本地History。
- 后台持续计时。
- 未经确认自动删除旧历史。

## Risks and Deferred Items

- Pixiv history endpoint/最小时长可能变化；开始实现时核验并记录，失败不影响本地history一致性。

## Source Anchors

- beta56 lib/app/db/history_db.dart、lib/pages/history/*
- 父PRD History要求；Settings、IllustStore、NovelReader view events

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
