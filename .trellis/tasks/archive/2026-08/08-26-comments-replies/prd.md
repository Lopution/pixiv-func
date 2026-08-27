# 复刻评论与回复

## Goal

复刻beta56作品评论、回复和输入体验，并用正确的Comment ID/parent ID模型消除本地更新错误。

## Confirmed Facts

- 父PRD要求avatar→user、emoji、stamp、translate、reply icon、删除本人评论和load replies。
- beta56最终行为使用reply icon，不是旧教程的long-press reply。
- beta56 assets/emojis与assets/stamps提供10列emoji、5列stamp输入素材；当前仓库尚未迁入。

## Dependencies

- 08-26-illust-detail-viewer、08-26-user-profile-follow、08-26-pixiv-network-token-refresh 和 08-26-settings-parity 完成。

## Requirements

- R1: 定义CommentEntity，明确commentId、parentCommentId、root/thread关系、owner、content、stamp、createdAt和replyCount，禁止复用ID字段。
- R2: CommentStore按illust/thread ID管理分页、replies和本地mutation revision；同一comment只存一份。
- R3: 评论项保留avatar跳User、文本/emoji/stamp、translate、reply icon、删除本人评论和加载回复。
- R4: 输入支持文本、10-column emoji、5-column stamps；迁入固定beta56资产并记录hash/provenance。
- R5: 发送reply使用明确parent comment ID；API成功后插入正确thread并更新reply count，失败不提前伪成功。
- R6: 删除仅允许当前账号自己的comment，成功后从正确parent/root移除并更新计数，失败恢复。
- R7: pagination/replies分别区分initial/load-more/error，取消/晚到response/重复send不会错位或重复。
- R8: translate请求不记录private comment，失败明确且不覆盖原文；无provider配置时显示真实不可用。

## Acceptance Criteria

- [ ] root comments和replies真实分页，reply icon打开对应输入上下文；无long-press reply。
- [ ] emoji/stamp grid列数、资产和选择/发送符合beta56。
- [ ] 回复A不会因ID混用更新B；插入/删除/reply count在root和reply列表一致。
- [ ] avatar路由、translate、删除本人评论及权限错误通过测试。
- [ ] 快速回复/删除、晚到分页、账号切换和网络失败不重复或串thread。
- [ ] analyze、全量test、debug build和真实API受控发评/回复/删除验证通过。

## Out of Scope

- Live弹幕。
- 私信。
- 批量管理评论。
- 长按回复旧行为。

## Risks and Deferred Items

- 真实发评/删除会改变外部账号数据，设备验收必须使用用户授权测试账号和可清理内容。

## Source Anchors

- beta56 lib/pages/illust/comment/*、reply/*、components/comment_item/*、comment_input/*
- beta56 assets/emojis、assets/stamps；UserStore/Detail routes

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
