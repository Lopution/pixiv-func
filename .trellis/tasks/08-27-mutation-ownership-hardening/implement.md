# Bookmark、Follow、Comments 写操作归属 — Implementation Plan

## Start Gate

- 等待 `08-26-restricted-compat-network` 完成 account-aware transport/error contract，并在本叶子 final planning review 后启动。
- Profile edit 字段/权限仍由原 `08-26-profile-edit` 任务拥有；本叶子只接入共享 mutation ownership。

## Steps

1. 复核 `bookmark_actions.dart`、`bookmark_store.dart`、comments state/repository、follow/profile mutation 和相关 tests，列出所有乐观写入、重试和 dispose 路径。
2. 先添加 fake transport 的延迟/反序/429/401 场景，复现跨账号和快速 toggle 污染。
3. 实现 `MutationEnvelope`、owner/revision 校验、去重/superseded 和 exactly-once terminal；把 server-confirmed 更新与 pending UI 分开。
4. 将 Bookmark、Follow、Comments 和 profile edit adapter 接入同一 policy；禁止非幂等操作后台隐式重放或跨账号恢复。
5. 补充聚焦测试、真实 API 失败分类和必要设备证据，更新 integration mutation ledger。

## Validation

```bash
/opt/flutter-3.47.0/bin/flutter test test/bookmark_store_test.dart test/bookmark_flow_test.dart test/comments_replies_test.dart test/user_profile_test.dart
/opt/flutter-3.47.0/bin/flutter analyze
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-mutation-ownership-hardening
git diff --check
```

真实 API 仅记录已实际观察的认证、限流和失败；未做设备/API 验证时不得填充对应状态。

## Completion Gate

- [ ] Bookmark/Follow/Comments/profile mutation 全部携带 account/revision owner。
- [ ] 快速重复、相反操作、账号切换、取消、429 和认证失效的测试通过，旧响应不能覆盖新 revision。
- [ ] 无 durable implicit replay，归档无 diff，integration ledger 完整。
