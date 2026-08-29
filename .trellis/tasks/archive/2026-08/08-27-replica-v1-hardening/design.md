# Replica v1 归档补强与集成修复 — Design

## Ownership model

本任务是协调层。它不拥有 feed、mutation、Novel、media 或 Android 的生产类；五个叶子分别拥有自己的实现文件和测试。归档目录只用于证据对照，不能作为写入目标。

| 叶子 | 代码边界 | 主要证据和集成出口 |
|---|---|---|
| `08-27-feed-generation-commit-hardening` | feed controller/repository、shared entity/cursor commit | refresh-wins、账号/筛选隔离、旧响应不污染 |
| `08-27-mutation-ownership-hardening` | bookmark/follow/comments/profile mutation orchestration | owner/revision、dedupe、429、取消与 server-confirmed state |
| `08-27-novel-markup-hardening` | Novel parser/token/layout budget | typed token fixture、未知标记、长文取消 |
| `08-27-media-job-recovery-hardening` | download task/group/recovery/MediaStore handoff | exactly-once terminal、pending cleanup、重启策略 |
| `08-27-android-platform-boundary-hardening` | WebView capability、intent、FileProvider/MediaStore lifecycle | API 36 设备和平台负向验证 |

## Cross-layer contracts

1. **Transport contract**：所有网络叶子消费共享 `NetworkAccessPolicy`；目标 host、候选 route、TLS/证书校验与 failure taxonomy 在 compat leaf 统一定义。叶子只报告 `success`、`eligible fallback` 或明确失败，不拼接代理 URL。
2. **Commit contract**：异步操作携带 owner context（账号/凭据 revision、generation 或 job id）。解析成功不等于可提交，只有 context 仍 active 时才能写 shared store、cursor、MediaStore 或 UI。
3. **Evidence contract**：每个叶子维护 source pin、当前代码位置、复现步骤、自动化测试和必要的设备/API 证据。证据状态严格分开，集成任务只汇总，不替代叶子验证。
4. **Failure contract**：取消、认证失效、限流、网络不可达、平台能力缺失、解析未知 token 和资源超限分别可观察；不得把失败静默转成空列表、成功收藏或永久 pending。

## Data flow

`source-evidence.md` -> 叶子 PRD/design -> 当前代码复现 -> owner-scoped implementation -> focused tests -> device/API evidence -> integration matrix。集成只消费叶子提交的契约和证据，不直接读取归档源码作为运行时依赖。

## Sequencing

- 当前 Ugoira 工作完成后，先完成 `08-26-restricted-compat-network` 的 shared transport contract。
- 依赖网络契约的 Feed、Mutation、Media、Android 叶子逐一执行；Novel 可在网络契约之外独立执行，但仍遵守任务审批。
- Media 叶子必须在 `08-26-ugoira-player-export` 完成后开始，避免同时修改相同的 Ugoira/output 边界。
- 五个叶子都完成后，`08-26-replica-v1-integration-release` 才能关闭补强 blocker；原 17 项 direct-child matrix 不变。

## Rollback and compatibility

- 每个叶子独立提交并保留迁移/回滚说明；不得 reset、force push 或清理无关工作树。
- 新 owner/revision 字段应允许旧内存状态安全失效，不能把历史 pending 写操作跨账号恢复。
- MediaStore pending 清理采用状态开关；能力不足时返回可见 blocker，不能绕过证书校验或安全校验。
