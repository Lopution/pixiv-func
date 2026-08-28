# 开源 Pixiv App 规划审查 — Implementation Plan

## Start Gate

- 本任务属于规划/研究任务；固定源码研究可在 planning 阶段完成。
- 任何产品代码实现或运行 `task.py start`，仍需本轮最终规划摘要后的后续明确批准；用户已允许创建 hardening planning tasks，故已创建 `08-27-replica-v1-hardening` 及五个叶子，但它们全部仍为 `planning`。
- 不提交、不推送、不修改远程仓库。

## Steps

1. 固定至少 3 个互补开源客户端的 commit，记录许可证、活跃度和选择理由。
2. 审查认证、网络/分页、共享 mutation、Reverse Image、Profile、Novel、下载/Ugoira、History/Settings、Widgets、Updater 与 Live 源码和测试；网络额外比较 ECH、DoH/direct connector、failure classification、WebView 和严格 TLS 反例。
3. 对照当前仓库实现与 17 个直接子任务，标记采用、受限采用、暂缓和拒绝。
4. 不改 archive；把完成后发现的缺口写入 Replica 父契约和 integration release。
5. 更新仍在 planning 的中间父任务与实现叶子，明确依赖、错误、取消、生命周期、设备 Gate 和回滚；把大陆 access 写入父/compat/network/integration 并把 network 提升为当前 Ugoira 边界之后的下一候选。
6. 校验 17 项覆盖、source pin、Trellis task schema、Markdown diff 和无产品代码变化。
7. 记录用户已确认的 Q1；完成 PRD convergence，并将已创建的 hardening parent/leaf 与原 17 项矩阵交叉引用。随后仍需用户分别批准叶子 start；不得把创建 planning task 解释为产品实现批准。

## Validation

```bash
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-open-source-pixiv-app-plan-audit
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-pixiv-func-replica-v1
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-replica-v1-integration-release
git diff --check
```

专项检查：

- `task-audit-matrix.md` 的直接子任务 ID 与 Replica 父 task `children` 做集合和计数比较。
- 所有第三方结论链接到固定 commit，不链接漂移分支作为唯一证据。
- 本轮审查新增的 `git diff` 只包含 `.trellis/tasks/`；保留既有 `pubspec`/Ugoira 工作树改动，archive 下无修改。
- 大陆 access 的父/leaf/integration交叉引用、execution order、已批准的 Q1 scope 与 external-proxy-free test matrix 一致。
- 本次不运行 Flutter analyze/test/build，因为未修改产品代码；也不声称设备、真实 API 或业务功能已验证。

## Completion Gate

- AC1–AC7 均有研究文件、规划 diff 和验证命令证据；Q1 决议及 hardening parent/5 leaves 已记录，最终规划收敛和各叶子 start 仍需用户审阅。
- 用户确认最终规划前保持 `planning`，不自动 start、commit、push 或 archive。
