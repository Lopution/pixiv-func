# Feed generation 结果提交边界 — Implementation Plan

## Start Gate

- 依赖 `08-26-restricted-compat-network` 的 shared transport contract 已验收，并完成本叶子 planning review 后才可启动。
- 不修改 `08-26-*` archive；当前 Ugoira 工作保持不动。

## Steps

1. 打开 `PagedFeedController`、Recommended/Ranking/New/Search/Profile controller、repository、`IllustStore`/`UserStore` 和既有 feed tests，记录当前 merge/cursor 写入点。
2. 先补 fake repository 的可控延迟和 response-order fixtures，复现 refresh、账号切换、筛选切换和 dispose 后的污染。
3. 实现最小 `FeedRequestContext`/commit gate，将 network response、parse、entity merge、dedupe、cursor 和 state 更新收束到同一 active-context 检查。
4. 为五类 feed 迁移调用点；不改变 beta56 的排序、空态、load-more UX 或 cache 范围。
5. 添加旧 generation、重复/恶意 cursor、same-ID update/delete/reorder 和 cancellation 测试；记录真实 API/设备验证边界。
6. 更新 integration release 的 feed evidence，不改原归档任务。

## Validation

```bash
/opt/flutter-3.47.0/bin/flutter test test/recommended_feed_test.dart test/ranking_feed_test.dart test/new_content_feed_test.dart test/search_catalog_test.dart test/user_profile_test.dart
/opt/flutter-3.47.0/bin/flutter analyze
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-feed-generation-commit-hardening
git diff --check
```

影响 Android/网络插件时增加 debug APK 和 API 36 设备验证；必须分开记录 `Implemented`、`Compiled`、`Unit-tested`、`Device-tested`。

## Completion Gate

- [ ] 五类 feed 的 entity/cursor/state 都受同一 generation gate 约束。
- [ ] 旧请求、旧账号、旧 filter 和 dispose 结果无法污染当前状态，且负向测试通过。
- [ ] shared network policy、归档无 diff 和 integration evidence 已核对。
