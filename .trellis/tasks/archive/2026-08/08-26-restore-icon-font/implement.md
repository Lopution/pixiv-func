# 恢复原版 iconFont 字体资产 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 核对 beta56 资产与 codepoint，记录远端 commit 和 SHA-256。
2. 迁入 assets/icon.ttf，更新 pubspec.yaml 字体注册。
3. 核对/补齐 AppIcons 映射和 Home 图标来源，不改变视觉参数。
4. 增加资源注册及 Home 图标聚焦测试。
5. 运行资源、analyze、test、debug APK 和渲染检查，记录证据。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-restore-icon-font
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 资产 SHA-256 与参考值一致。
- AppIcons.home/ranking/n/search 的 family 和 codepoint 正确，Settings 不使用 font family。
- Flutter asset bundle 与 debug APK 包含 assets/icon.ttf。

## Risky Files and Rollback Points

- assets/icon.ttf、pubspec.yaml、lib/app/icons/app_icons.dart、lib/features/home/home_page.dart

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

