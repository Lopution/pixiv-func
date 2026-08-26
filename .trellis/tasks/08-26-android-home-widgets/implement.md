# 复刻 Android Home Widgets — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 提取beta56Widget layouts/update/click行为，核验API36 AppWidget/WorkManager要求。
2. 研究并选定可实测的安全background credential/data bridge。
3. 实现providers/receivers/layouts/unique worker与bounded image renderer。
4. 实现account revision/reauth/no-account states和typed deep link clicks。
5. 增加worker uniqueness、security、account switch、bitmap和PendingIntent tests。
6. 运行analyze/test/build，在真机验证添加/resize/reboot/background/remove。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-android-home-widgets
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- multi-widget IDs、unique work、last removal。
- account/no-account/reauth/switch cache clearing。
- PendingIntent/deep link security。
- network retry/TLS/bitmap bounds。
- reboot/app update/API36 background behavior。

## Risky Files and Rollback Points

- android/app/src/main/kotlin/.../appwidget、res/xml/layout、WorkManager依赖、secure credential bridge

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

