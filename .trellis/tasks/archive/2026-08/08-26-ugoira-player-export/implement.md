# 实现 Ugoira 播放与 GIF 导出 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 核验当前metadata/ZIP及beta56视觉/时序，收集小/大/损坏样本；确认 repository 只依赖 shared Pixiv media transport，兼容网络由后续 P0 leaf 接入。
2. 定义 typed archive/frame/pixel limits、owned temp/index/frame source/cache/scheduler/export job contracts。
3. 实现stream ZIP、header-before-decode 安全索引、有界decode/dispose和deadline播放。
4. 实现cover/play/pause/visibility/lifecycle/error UI。
5. 实现task-group exactly-once、有界GIF export、owned pending commit与DownloadManager/MediaStore集成。
6. 增加zip攻击/伪造size/ratio/dimensions/pixel budget、scheduler、cache/dispose、cancel/cleanup和GIF tests。
7. 运行analyze/test/build，在真机测峰值内存、离屏和导出。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-ugoira-player-export
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- metadata/frame order/delay和deadline drift。
- cache window/eviction/ui.Image dispose。
- zip-slip/bomb/mismatch/corruption。
- entry/frame count、compressed/uncompressed bytes、ratio、header dimensions/pixels 和 format allowlist。
- tap/pause/offscreen/lifecycle/cancel。
- GIF output delay/order/terminal/MediaStore cleanup。
- media transport injection/failure/cancel contract；不得新增固定 IP、代理 URL 或独立 TLS override。

## Risky Files and Rollback Points

- lib/core/ugoira/、lib/features/illust/ugoira/、archive/image codec依赖、temp storage、GIF encoder

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。
