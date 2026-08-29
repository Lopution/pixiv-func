# Replica v1 归档补强与集成修复 — Implementation Plan

## Start Gate

- 本父任务只做协调，不运行 `task.py start` 承载产品实现。
- 五个子任务仍保持 `planning`，每个叶子须在自己的 PRD/design/implement 审阅完成、依赖满足并获得明确批准后单独启动。
- 本阶段不修改归档、不改动产品代码、不提交或推送；当前 Ugoira 工作树改动保持原样。

## Steps

1. 复核 `source-evidence.md`、原 Replica v1 父任务和五个叶子目标文件，确认每个叶子的代码 ownership 不重叠。
2. 在 `08-26-restricted-compat-network` 完成并验收 shared transport contract 后，按 Feed -> Novel -> Mutation -> Media -> Android 的顺序一次启动一个叶子；若依赖或实测证据不足，保持 blocker。
3. 每个叶子先用当前测试/日志复现缺口，再实现最小 owner/generation/capability 边界；任何跨叶子公共契约先写入对应 design，并由集成任务登记。
4. 每个叶子完成聚焦测试、`flutter analyze`、适用的 debug build 和设备/API 验证后，回填四层状态：`Implemented`、`Compiled`、`Unit-tested`、`Device-tested`。
5. 叶子归档后只更新本父任务和 `08-26-replica-v1-integration-release` 的 blocker/evidence matrix；不把新发现写入 archive，也不扩大原 17 项产品范围。
6. 五个叶子的证据齐全后运行最终集成回归，复核大陆无外部代理矩阵、权限/许可证、失败恢复和 beta56 可见行为。

## Validation

规划阶段执行：

```bash
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-replica-v1-hardening
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-feed-generation-commit-hardening
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-mutation-ownership-hardening
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-novel-markup-hardening
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-media-job-recovery-hardening
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-android-platform-boundary-hardening
git diff --check
```

实现阶段由各叶子执行最小聚焦测试，再按影响范围补充全量 `flutter analyze`、`flutter test`、`flutter build apk --debug` 和必要的 API 36 真机证据。未执行的层级必须明确标记，不能由父任务代填。

## Completion Gate

- [ ] 五个叶子均完成并提交独立证据，且没有归档目录 diff。
- [ ] 原 Replica v1 的 17 个 direct children 和依赖顺序仍可追溯；最终集成任务已接收五个 hardening blocker 的关闭证据。
- [ ] 旧请求/账号/任务不会提交到新状态；取消、限流、认证失效、重启、能力缺失和资源超限都能被观察。
- [ ] 大陆无外部代理目标按真实出口和样本边界诚实报告，未引入关闭证书校验、改写 `Host` 或第三方反代。
- [ ] 用户明确批准前，父任务和叶子仍为 `planning`，不自动 start、archive、commit 或 push。
