# W6 下载任务状态机现状（codebase）

基线 `main@8067b2d`。核心层不改语义（§6），但"查看结果/移除终态"可能需要 manager 侧最小增量——标记为决策点。

## 1. 状态枚举（`core/download/download_recovery.dart`）

```dart
enum DownloadStatus {            // 18-27
  queued, running, finalizing, canceling,
  succeeded, failed, canceled, retryable, orphaned,
}
enum DownloadFailureKind {       // 33-46
  auth, rateLimit, network, storage, permission, decode,
  resource, canceled, ownership, unknown, paused,
}
```

- `isTerminal`（download_task.dart:25-35）：`succeeded|failed|canceled|orphaned` 终态；`queued|running|finalizing|canceling|retryable` 非终态。
- `retryable` 双义：`failureKind==paused` = 用户暂停（保留 `resumeAnchor`）；其它 failureKind = 可重试失败/恢复待决。
- `DownloadEvent`（148-168）：`succeeded|failed|canceled|orphaned` 四终态事件，每任务恰好一次（`_complete` guard，manager:1101-1106）。

## 2. `DownloadManager` 动作 API（`core/download/download_manager.dart`）

| 方法 | 行号 | 语义 |
|---|---|---|
| `pause(taskId)` | 223-253 | queued→retryable(paused)；running→pauseRequested+cancel（收尾落 retryable）；其余 no-op |
| `cancel(taskId)` | 256-281 | queued/retryable→canceled（立即）；running→canceling；**删除保留的部分输出**；终态 no-op |
| `retry(taskId)` | 283-310 | 仅 failed/canceled/retryable → 重新 submit 新 attempt，group.jobIds 原位替换；返回新 snapshot |
| `pauseGroup/resumeGroup/cancelGroup` | 312-343 | 对子任务逐个 pause/retry/cancel；succeeded 子项不受影响 |
| `recover()` | 345- | 进程重启扫描；仅账号+目标匹配的记录 → retryable，绝不自动重试 |
| `tasks`/`groups` | 108-112 | 只读快照列表；**无 dismiss/remove/clear API——终态任务永久驻留 `_jobs`** |
| `events`/`changes` | 101-105 | 终态事件流 / 任意变更流（页面用 `changes` + setState） |

`DownloadTaskSnapshot`（download_task.dart:38-143）持有 `illustId`/`pageIndex`/`displayName`/`finalUri`/`resumeAnchor`/`submission.groupId`——"查看"动词的材料已在快照内（`openIllust(context, task.illustId)`），无需新数据源。

## 3. 当前 UI 映射表（download_tasks_page.dart）

| status（+failureKind） | 文案 | trailing 动作 | 问题 |
|---|---|---|---|
| queued | Queued | pause + cancel | ✓ |
| running | Running | pause + cancel | ✓ |
| finalizing | Running | 静态 icon（`_` 分支） | 与文案"Running"不一致但可接受 |
| canceling | Canceling | 仅 cancel（第一键省略） | 取消中再点取消无意义但无害 |
| retryable + paused | Paused | **refresh 图标 + `retryDownload` 文案 → `manager.retry`** + cancel | ⚠ Astra 项：暂停被映射为"重试"；§5.7 应为"继续/恢复"语义 |
| retryable + 其它 | Failed | 同上 refresh+retry + cancel | ✓（重试正确） |
| failed | Failed | refresh + retry | ✓ |
| canceled | Canceled | refresh + retry | ✓（重开一次操作） |
| succeeded | Completed | **仅静态 ✓ icon** | ⚠ 缺"查看"（§5.7：统一"查看"类动词）；缺移除 |
| orphaned | Failed | 静态 `info_outline` | ⚠ 无解释/无重试（retry 会拒绝？不——retry 不接受 orphaned，所以给 icon 合理，但用户无出路） |

组级（`_DownloadGroupSection`）：queued/running→pause+cancel；retryable/failed/canceled→play_arrow(`resumeDownload`→resumeGroup=逐子 retry)+cancel；succeeded/其它→静态 icon。组级暂停态文案靠 `_everyRetryablePaused` 推断（76-87）。

## 4. 层级缺口

- 页面把 `groups` 与 `tasks` 平铺渲染（58-62），**属于组的子任务仍出现在顶层 `tasks` 列表**——父子层级需要 UI 侧按 `task.groupId` 聚合（snapshot.groupId → `submission?.groupId`，task:86）。
- 无过滤/排序/批操作（对比 PixEz：状态 chips + 排序 + 批量 retry/clear）。
- 无提交后直达：三处提交点只发裸 SnackBar（`downloadQueuedMessage`，无 `action`）。
- 无完成通知消费：`manager.events` 在 UI 层仅 illust_detail/ugoira/updater 使用（grep 验证：detail:85、ugoira_viewer:592、update_download.dart:314,452）。

## 5. 测试锚点

`test/download_tasks_page_test.dart`（254 行）覆盖：组暂停/恢复/取消（83-146）、paused 子任务 refresh→retry 至成功（203-254）。动作映射变更需同步更新断言（特别是 refresh icon 的位置断言 `find.byIcon(Icons.refresh)`）。
