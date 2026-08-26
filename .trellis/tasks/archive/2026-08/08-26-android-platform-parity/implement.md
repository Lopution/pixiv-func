# 补齐 Android API 36 平台行为 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 核对 Flutter 3.47/API 36 edge-to-edge 与 predictive back 官方契约及当前 template。
2. 设计 typed intent、back、MediaStore、FileProvider 和 WebKit capability 接口。
3. 更新 Manifest/resources/MainActivity/platform channel，并实现 Dart router/back coordinator。
4. 增加 URI/MIME/path/权限单元测试、Widget back 测试和 merged-manifest 检查。
5. 构建 APK，在 API 36 设备逐项验证 edge-to-edge、back、links、SEND、provider 和 MediaStore 失败清理。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-android-platform-parity
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- typed deep-link parser allow/deny 表。
- root/nested back 时序与生命周期。
- SEND content URI MIME/permission/size 异常。
- FileProvider path traversal 和 authority。
- MediaStore begin/finalize/abort；merged manifest 权限审计。

## Risky Files and Rollback Points

- android/app/src/main/AndroidManifest.xml、Android resources、MainActivity.kt、platform channel、navigation/root scaffold

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

