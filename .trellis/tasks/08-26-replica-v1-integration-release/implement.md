# 完成 Replica v1 集成与发布验收 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 冻结候选 commit/依赖锁，建立父 AC 与子任务 evidence matrix。
2. 运行静态、全量测试、CI、debug/release/flavor 构建和 merged-manifest 审计。
3. 执行 API 36 真机+真实账号功能、intent、lifecycle、failure 和升级矩阵。
4. 执行安全/秘密/网络/权限/clipboard/FileProvider 审计及媒体/长列表性能测试。
5. 修正 LICENSE/NOTICE/README/归属/版本/构建说明，核对第三方许可和资产 provenance。
6. 搜索并清除业务 placeholder/no-op/mock，复跑所有受影响门禁。
7. 记录最终结果和 blocker；仅在全部 AC 真实满足时完成并归档父任务。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-replica-v1-integration-release
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 全量 flutter analyze/test/integration_test 与 CI。
- debug/release、GitHub/F-Droid flavor 和 merged manifest。
- 真实账号功能矩阵、API 36 back/deep-link/SEND/MediaStore/WebView。
- token/网络/进程/旋转/低存储/取消/恶意输入失败矩阵。
- 内存/CPU/后台活动与 secret/license/placeholder 审计。

## Risky Files and Rollback Points

- LICENSE、NOTICE/ATTRIBUTION、README、pubspec version、CI、Android flavors/signing、integration_test/、跨功能配置

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

