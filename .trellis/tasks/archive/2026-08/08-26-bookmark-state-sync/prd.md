# 同步跨页面收藏状态与交互

## Goal

精确复刻 beta56 收藏操作，并通过 ID-keyed BookmarkStore 消除推荐卡片与详情页状态不同步。

## Confirmed Facts

- beta56 最终 commit 本身修复了 Recommended 进入 Detail 后收藏更新问题，旧 controller 仍按 Widget/tag 局部持有状态。
- 父 PRD要求未收藏短按 public、长按 public/private sheet、pending spinner、非 optimistic、失败恢复和已收藏时禁用长按 sheet。
- Recommended 和 Detail 已共享 IllustStore，但 mutation 状态需要独立 BookmarkStore。

## Dependencies

- 08-26-pixiv-network-token-refresh、recommended-feed-paging、illust-detail-viewer 完成。

## Requirements

- R1: BookmarkStore 按 account ID + entity type + entity ID 保存 confirmed restrict、pending operation、previous state 和 error，不由页面创建副本。
- R2: API 初始 bookmark 字段进入共享 store 时遵守 merge 版本，不能覆盖本地更新后的 confirmed 状态。
- R3: 未收藏短按直接请求 public；未收藏长按打开约屏幕 35% 的 bottom sheet，提供 public/private。
- R4: 请求中图标替换为约 24px CupertinoActivityIndicator；不 optimistic 更新，API 成功后才变为 bookmarked。
- R5: 失败时恢复原 confirmed 图标并显示原版节奏的可观察错误；重复点击、跨页面并发和晚到响应不得触发重复 mutation。
- R6: 已收藏时长按不打开 restrict sheet；短按执行取消收藏并遵循同样 pending/失败状态。
- R7: Recommended card、Detail、Search、Ranking 等同一 ID 的订阅即时看到统一 confirmed/pending 状态。
- R8: 账号切换时 store 隔离；登出清理内存状态且不把账号 A 的收藏泄漏给账号 B。

## Acceptance Criteria

- [ ] 短按未收藏只发送一次 public add；长按可选 public/private；已收藏长按不显示 sheet。
- [ ] pending 阶段显示 spinner 且状态不提前翻转；成功后所有同 ID 视图同步，失败后全部恢复。
- [ ] 从 Recommended 进入 Detail、在任一页面操作后返回，两个页面状态始终一致。
- [ ] 快速重复点击、两个页面同时点击、请求取消/超时和晚到响应不产生重复请求或错误终态。
- [ ] 账号切换隔离、API refresh merge 和取消收藏路径测试通过。
- [ ] analyze、全量 test、debug build 及真实 API public/private/add/delete 验证通过。

## Out of Scope

- 自定义 bookmark tags。
- 批量收藏。
- Novel 收藏 UI，除非复用 store 的 entityType 基础接口。

## Risks and Deferred Items

- Pixiv API 可能返回旧 bookmark snapshot；需要 operation revision/confirmed timestamp 防止网络晚到覆盖本地成功。

## Source Anchors

- beta56 components/bookmark_switch_button/bookmark_switch_button.dart、controller.dart
- beta56 final commit c62b18c；Recommended/Detail shared store contracts

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
