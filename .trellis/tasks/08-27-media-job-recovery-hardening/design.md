# 下载与 Ugoira 任务恢复 — Design

## Ownership and state

`DownloadGroup` 持有 immutable submission snapshot 和 account owner；`DownloadJob`/page/frame subtask 使用 `queued/running/finalizing/succeeded/failed/cancelled/retryable/orphaned` 状态。每个 job 只有一次 terminal transition，所有回调通过 job id、owner 和 snapshot revision 校验。

## Output lifecycle

临时目录、frame files、GIF/video output 和 MediaStore pending URI 都登记 owner。写入采用 prepare -> bounded write -> verify -> finalize；成功/失败/取消/超限/重复回调均走幂等 cleanup。pending URI 在 finalize 前不可对外暴露，进程重启按 owner 和 snapshot 判断恢复或清理。

## Recovery and resource budget

重启扫描只恢复有完整 snapshot、可验证 destination 和当前账号 owner 的 pending；`running/unknown` 进入 `retryable` 或 `failed` 的明确策略，旧账号任务不自动重放。Ugoira frame count、pixel count、compressed size、concurrency 和 memory budget 在 coordinator 层约束；取消贯穿下载、解码和编码。

## Integration and rollback

本叶子在 `08-26-ugoira-player-export` 归档/完成后才拥有共同输出边界；此前只能记录 blocker。下载 manager 对外 adapter 保持现有 single/all-page UX，新增恢复策略可通过版本化状态/feature gate 回退，不删除可恢复的诊断元数据。
