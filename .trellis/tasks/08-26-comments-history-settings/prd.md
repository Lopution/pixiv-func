# 复刻评论、历史与设置

## Goal

作为中间父任务，协调评论、历史和设置三个独立叶子任务，并确保它们共享账号、实体、数据库和配置契约而不互相耦合。

## Confirmed Facts

- 该范围已拆为comments-replies、history-persistence、settings-parity。
- beta56三者分别位于illust/comment、history/db和settings/services，生命周期与风险不同。
- Comments依赖Profile/Detail，History依赖可见性/DB，Settings为多个服务提供持久化配置。

## Dependencies

- 08-26-bookmark-state-sync完成。
- 08-26-user-profile-follow为Comments头像/用户route提供基础。

## Requirements

- R1: 本中间父任务不直接实现业务代码，维护三个叶子任务范围、顺序、共享settings/history hooks和集成验收。
- R2: settings-parity先稳定默认值/schema；history-persistence接入history toggles；comments-replies接入translate/asset配置。
- R3: 三个叶子分别规划审批、实现、验证、提交和归档，数据库/asset/settings migration各自有回滚。
- R4: 完成前验证Settings toggles即时影响History/下载/内容行为，Comments用户route和History详情状态使用共享stores。
- R5: 不得将评论内容、浏览记录或翻译凭据混入同一SharedPreferences大JSON。

## Acceptance Criteria

- [ ] 三个叶子任务均归档且无placeholder。
- [ ] Settings默认值/切换正确驱动History和相关服务；重启后状态一致。
- [ ] Comments/History进入共享User/Illust/Novel页面且状态同步。
- [ ] 账号切换、DB migration、离线/删除、translation失败不串数据或崩溃。
- [ ] 全量analyze/test/debug build及三个功能集成真机验证通过。

## Out of Scope

- 把三个功能合并成单一repository。
- 新增原版没有的云历史或评论功能。

## Risks and Deferred Items

- 设置schema和History DB会跨多个后续任务；必须版本化并冻结public contract。

## Source Anchors

- beta56 lib/pages/illust/comment、lib/pages/history、lib/app/db/history_db.dart、lib/pages/settings
- 三个叶子task artifacts与父PRD

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
