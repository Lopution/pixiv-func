# 复刻 Live、Widgets 与 Updater — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 在实现当天审阅Live feasibility、Widget snapshot/headless/IPC和Updater signed-manifest/signer平台research。
2. 依次完成live-player、android-home-widgets、updater-flavors。
3. 核对无秘密snapshot、headless复用、worker uniqueness、orientation/wakelock、FileProvider、update trust和flavor权限。
4. 运行进程/账号/网络/RemoteViews IPC/invalid update signature/flavor/background跨功能回归；网络部分覆盖 system proxy/VPN off、shared route 与 headless failure，不允许私有 proxy/IP fallback。
5. 记录父acceptance evidence并归档。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-live-widgets-updater
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 三个child状态/evidence。
- Live lifecycle+follow integration。
- Widget no-account/refresh/click/security。
- GitHub/F-Droid merged manifest/build。
- 进程重启和后台资源审计。

## Risky Files and Rollback Points

- 三个child contracts、Android manifests/flavors/background；父任务不直接编辑业务代码

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。
