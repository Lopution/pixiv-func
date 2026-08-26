# 复刻作品详情与全屏查看器 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 提取 beta56 详情、viewer、download-mode 和 tag 行为，核验 detail API/schema。
2. 实现 detail mapper/repository、route、controller 和 IllustStore merge。
3. 实现单/多页 UI、badges、作者/tags/summary、错误与受限状态。
4. 实现 Viewer page/zoom/lifecycle/返回，并接入下载模式与真实 DownloadManager。
5. 实现最小 tag result 和 block mode 的真实路径，为后续 Search 留复用接口。
6. 增加 mapper、merge、手势、下载、tag、错误和 lifecycle 测试。
7. 运行 analyze/test/build，在 Android 真机验证单图、多页、viewer、下载模式和 back。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-illust-detail-viewer
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- detail mapper 与 store merge 不倒退。
- 单/多页 badges、route initial index 和 n/total。
- 0.9/6.0 边界、双指缩放、横翻与 predictive back 冲突。
- 下载模式单页/全部页队列、失败与重复点击。
- tag tap/block、删除/受限、图片失败和旋转恢复。

## Risky Files and Rollback Points

- lib/features/illust/detail/、viewer/、Replica route、IllustStore merge、DownloadManager 集成、图片 viewer 依赖

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

