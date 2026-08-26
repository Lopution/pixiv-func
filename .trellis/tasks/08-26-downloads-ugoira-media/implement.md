# 实现下载、Ugoira 与媒体流水线 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 审阅并完成download-manager-mediastore。
2. 在Detail下载模式真实可用后审阅并完成ugoira-player-export。
3. 核对共享transport/MediaStore、文件命名、settings和终态接口。
4. 运行单/多页、取消/重试、Ugoira播放/导出及大样本性能回归。
5. 记录父acceptance evidence并归档。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-downloads-ugoira-media
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 两个child状态/evidence。
- 详情/Downloader/Settings integration。
- Ugoira play/pause/offscreen/export integration。
- 大文件/多帧内存与cleanup。
- MediaStore result和duplicate completion。

## Risky Files and Rollback Points

- 两个child contracts、shared media interfaces；父任务不直接编辑业务代码

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

