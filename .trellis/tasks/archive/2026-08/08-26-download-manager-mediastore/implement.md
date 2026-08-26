# 实现流式下载与 MediaStore — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 定义 DownloadRequest、task state machine、scheduler、sink 和事件契约。
2. 实现共享 transport、并发 3、去重、取消、重试、progress throttle。
3. 接入 PlatformMediaStore begin/write/finalize/abort 和安全文件命名。
4. 为详情页提供单页/全部页 typed coordinator 接口与状态查询。
5. 编写并发、大流、取消、重试、重复完成、redirect/TLS 和 MediaStore fake 测试。
6. 运行 analyze/test/build，并在 API 36 真机验证单页、多页、取消与失败清理。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-download-manager-mediastore
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 并发许可和动态上限。
- 有界 streaming、未知长度、慢流和中断。
- dedupe、cancel/race/retry、终态幂等。
- MediaStore pending 清理、命名/MIME/目录。
- 恶意 redirect/TLS 失败和日志脱敏。

## Risky Files and Rollback Points

- lib/core/download/、网络 transport 接口、Android MediaStore channel、pubspec.yaml、详情下载集成点

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

