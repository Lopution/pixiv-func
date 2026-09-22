# Codebase: 回复页滚动结构、回复目标与发送状态（现状）

复核基线：`main@8067b2d`（HEAD）。行号以该提交为准。

## 1. 页面与路由

- `CommentsPage`（`lib/features/comments/comments_page.dart:24`）：作品根评论页，`CommentFeedQuery.root`（:41-42）。
- `CommentRepliesPage`（:135）：单条根评论的回复页，`CommentFeedQuery.replies`（:156-160）。
- 路由（`lib/app/navigation/routes.dart`）：illust 分支 `comments`→:443-446、`comments/:rootCommentId`→:448-462；novel 分支 :522-545；`openCommentReplies` :1373-1388 以 `extra: rootComment` 传首帧快照。
- 两个页面的 `Scaffold` **均未设置** `resizeToAvoidBottomInset`（默认 `true`，:50、:170），键盘弹出时整列 body 被压缩——这正是 Astra #19「键盘弹出后可用回复区域进一步缩小」的机制放大器。对照：仓库现有约定是 `resizeToAvoidBottomInset: false` + 手动 `MediaQuery.viewInsetsOf` 底部 padding（`search_page.dart:34,552`、`home_page.dart:160`；component-guidelines「Common Mistakes」明确禁止依赖默认 resize）。
- 两页均**无 `PopScope`**：面板/键盘开着时系统返回直接退页（predictive back 契约见 component-guidelines §Predictive Back）。

## 2. 回复页滚动结构（Astra #19 目标）

`CommentRepliesPage.build`（:172-214）：

```
Column(
  Expanded(
    Column(
      if (root != null) CommentItem(comment: root, ...)   // :177-182 根评论在滚动区外
      Padding(... Text(commentReplies) ...)               // :183-192 固定标题行
      Expanded(_CommentFeedView(query: replies, ...))     // :193-199 只有回复列表可滚
    )
  )
  CommentComposer(...)                                    // :204-211
)
```

根评论是滚动容器外的固定块；长根评论 + 键盘顶起时回复列表被挤到很小。`_CommentFeedView`（:272-365）内部结构：`PullToRefresh` → `NotificationListener<ScrollNotification>`（extentAfter < 1.2 视口触发 `loadMore`，:322-330）→ `SmoothWheelScroll`（桌面滚轮平滑，builder 回传 controller/physics）→ `ListView.builder`（:333-358，`itemCount = comments.length + 1`，末位是 `FeedTail`）。把根评论纳入滚动语义的最小改动点：让 replies 列表的 itemBuilder 前两个槽位渲染根评论与标题，或在 `SmoothWheelScroll` builder 内换 `CustomScrollView` slivers——注意 loadMore 监听与 `FeedTail` 槽位要随之调整，且不要破坏 `PullToRefresh` 契约（spec component-guidelines §Shared Pull-to-Refresh）。

## 3. 回复目标传递

- 根页：`_replyTarget`（:39）由 `_CommentFeedView.onReply` 设置（:57），`CommentComposer(replyTo: _replyTarget?.user.name, onCancelReply: () => _replyTarget = null)`（:62-70）。初始为 null → 引用条不显示。
- 回复页：`_replyTarget`（:154），但 `replyTarget = _replyTarget ?? root`（:166）——**默认回复根评论**；`onCancelReply` 重置为 root（:206）而非 null，即回复页引用条恒显示。
- `CommentItem` 的回复入口是显式 `_ActionPill`（comment_item.dart:263-268，icon `reply_outlined` + `commentReply` 文案）；无长按回复。
- 发送参数：根页 `_send`（:76-93）用 `parentCommentId: target?.id`、`rootCommentId: target?.rootCommentId`；回复页（:217-230）`parentCommentId: _replyTarget?.id ?? widget.rootCommentId`、`rootCommentId` 恒为路由参数。
- **点击回复不聚焦输入框**：`onReply` 只 `setState` 目标，composer 的 `_focusNode` 无从感知（Astra #19 明确要求「点击回复主动聚焦」）。
- **发送成功后 `_replyTarget` 保留**：`_sendText` 只清文本框（comment_input.dart:213），两页都不动 `_replyTarget`；根页 mutationKey 含 target id（:47-48），换目标即换 pending key。W7 必须决策「清除 or 保留」并用测试固定——Shaft 参考实现是发送成功后清除（详见 external 文件）。

## 4. 发送状态机（现状）

- 数据层非乐观：`commentActionsProvider.send`（comment_actions.dart:28-48）→ `store.beginSend`（comment_store.dart:152-185，按 `send:<kind>:<workId>:<parentId|root>` 去重并发）→ repository `addComment` → `commitSend`/`failSend`（:187-247，`MutationStatus` ∈ pending/confirmed/failed/cancelled/superseded，mutation_models.dart:10）。
- UI 层：`sending = store.mutations[mutationKey]?.pending == true`（comments_page.dart:49、:169）；composer 内另有 `_busy`（comment_input.dart:36），`_disabled = widget.sending || _busy`（:38）。
- 发送中表现：仅「所有按钮 disabled」——无 spinner/progress 指示（W7 要求「发送中可见」）。
- 失败：`onError` → `showAppSnackBar`（comments_page.dart:95-101，走 #48 共享入口）；草稿保留（`_sendText` 只在成功路径 `_controller.clear()`，comment_input.dart:211-215）——现状已正确，须保留。
- 成功：清文本框；stamp 发送成功额外关面板（:228）。**无成功反馈**（无 SnackBar/触觉），列表靠 `commitSend` 的 `prepend`（comment_store.dart:187-221）插入新评论 + `updateReplyCount`。
- 删除：`AlertDialog`（`showAppDialog`，comments_page.dart:107-132、:241-269）；回复页删根评论成功后 `pop` 退出（:262-264）。
- 错误文案区分：`commentPermissionDenied` vs `commentSendFailed`（根页 :97-99；回复页 :232-235 不区分，统一 sendFailed）。

## 5. l10n 键（app_en.arb:759-782）

已有：`commentTitle/Input/Reply/ReplyTo/CancelReply/Send/Delete*/SendFailed/LoadFailed/LoadMoreFailed/NoResults/Replies/Translate*/Emoji/Stamps/PermissionDenied`。**缺**：发送中文案（如需要）、屏幕阅读器用的面板状态/emoji 名称 label。

## 6. spec 既有契约（必须遵守）

- `frontend/state-management.md` §Comments and Replies Contract（:261-350）：store 索引、mutation key、非乐观发布、emoji/stamp 资产归属、翻译 overlay 语义；Widget 测试清单含「10/5 grids」——改列数时必须同步修订该契约与本测试。
- `component-guidelines.md`：Scaffold+autofocus 禁默认 resize（:97-99）；弹层/Dialog 经 `app_overlays.dart`；icon-only 必须有 tooltip/语义 label（:66-72）。
- W7 门禁（implement.md §6 W6/W7）：「评论输入四态互斥，失败保留草稿，发送成功后的 reply target 行为固定」「只消费 W4 已合入的 §5.6 触觉 owner；发送终态使用约定分级（明确震动）；文案按 §5.7 术语表」。
- §5.3 FormState：发送属 `action`（idle/busy/success/error）；草稿文本属 draft 语义（失败保留）。
