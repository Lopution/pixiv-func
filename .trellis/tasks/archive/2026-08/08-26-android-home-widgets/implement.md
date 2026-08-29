# 复刻 Android Home Widgets — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 提取beta56Widget layouts/update/click行为，核验API36 AppWidget/WorkManager要求。
2. 定义并实现atomic/versioned/account-revision WidgetSnapshot；先完成无秘密native渲染路径。
3. 以API36进程冷启验证headless Flutter是否能复用现有AccountStore/PixivHttpClient/TokenRefreshGate 与 shared `NetworkAccessPolicy`；失败则记录blocker，不新建native credential/refresh/DoH/IP/proxy栈。
4. 实现providers/receivers/layouts、按family/revision命名的unique worker、resize去抖与bounded image/IPC renderer。
5. 实现account switch/reauth/no-account清理、same-account last-good和typed deep link clicks。
6. 增加snapshot schema/age/corrupt/atomicity、worker uniqueness、security、account switch、bitmap/IPC和PendingIntent identity tests。
7. 运行analyze/test/build，在真机验证添加/resize风暴/reboot/process-death/background/remove。

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
- snapshot schema/account revision/atomic replace、same-account last-good vs account-invalid clear、RemoteViews IPC budget。
- reboot/app update/API36 background behavior。
- system proxy/VPN off 的 headless direct-first/compatible route 与 carrier/date evidence；Widget 不私有化兼容路径。

## Risky Files and Rollback Points

- android/app/src/main/kotlin/.../appwidget、res/xml/layout、WorkManager依赖、secure credential bridge

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

---

## 任务关闭（2026-08-29）

实现范围已交付并有 API 35 真机证据（8 张截图见 `research/screenshots/`）。
**未完成部分已转入 `08-29-replica-v1-completion` R5**：

- API 36 镜像下的 WebView / MediaStore / headless Worker / intent 生命周期矩阵
- 系统重启后的 widget restore 完整矩阵

归档状态为 `completed` 表示**本任务关闭**，不代表上述验收已通过。

另：`widget_feed_loader` 的 `ApiUnauthorized` 分支在本任务交付时存在缺陷——它比较凭据版本，
而处理 401 本身就会推进该版本，导致每次真实鉴权失败都被判成 `superseded`。已在
`08-29-defensive-code-removal` 中修复，详见
`08-29-replica-v1-completion/research/archived-evidence-drift.md`。
