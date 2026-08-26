# 复刻 Novel 阅读器 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 核对 beta56 reader 行为和当前 Novel API/markup，记录样本与 schema。
2. 定义 Novel entity/content blocks、repository、error 和 version contract。
3. 实现 layout engine、bounded cache、stable anchor 和可取消计算。
4. 实现 reader UI、30% tap zones、百分比、设置/主题/旋转/lifecycle。
5. 增加 mapper、布局边界、anchor 恢复、缓存失效、长文本性能和 Widget 手势测试。
6. 运行 analyze/test/build，并在多篇真实 Novel 上验证旋转、字号和连续阅读。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-novel-reader
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- API mapper、未知 markup、空/超长段落。
- 左右 30% tap、swipe、首尾页和百分比。
- 旋转/字号/行距/theme 后 anchor 恢复。
- cache key/eviction/cancel 和性能阈值。
- 删除/受限/网络错误及无伪 save/share。

## Risky Files and Rollback Points

- lib/core/entity/novel_entity.dart、lib/features/novel/、layout engine/cache、字体设置与 History 接口

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

