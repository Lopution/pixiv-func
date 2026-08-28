# 下载与 Ugoira 任务恢复 — Implementation Plan

## Start Gate

- 必须先完成 `08-26-ugoira-player-export`，再完成 `08-26-restricted-compat-network` contract，并通过本叶子 final planning review。
- Ugoira 在途期间不编辑 `lib/core/ugoira/` 或同一 output/finalize 文件；归档目录始终只读。

## Steps

1. 复核 `download_manager.dart`、`download_task.dart`、coordinator/sink、`media_store_channel.dart`、Ugoira output 和现有 manager tests，列出 state/cleanup/restart 缺口。
2. 用 fake filesystem/MediaStore/transport 构造 crash points：download、decode、finalize、pending、duplicate callback 和 account switch。
3. 实现 immutable submission snapshot、group/job state machine、owner gate 和 exactly-once terminal/cleanup；保留 single/all-page UI 行为。
4. 加入 restart recovery policy、orphan cleanup、MediaStore pending 管理和 bounded Ugoira resource checks。
5. 补充取消、错误分类、空间/权限/超限、重启和 API 36 设备测试，更新 integration recovery evidence。

## Validation

```bash
/opt/flutter-3.47.0/bin/flutter test test/download_manager_test.dart
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter build apk --debug
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-media-job-recovery-hardening
git diff --check
```

Ugoira/MediaStore 真实设备验证单独记录；只跑 unit test 时不能声称重启或平台行为已验证。

## Completion Gate

- [ ] Ugoira 完成后，group/job/output/pending 的 owner 和 snapshot 可追溯。
- [ ] 成功、失败、取消、重启和重复回调 exactly-once；无 orphan temp/pending。
- [ ] frame/pixel/memory 有界，归档无 diff，integration recovery evidence 已回填。
