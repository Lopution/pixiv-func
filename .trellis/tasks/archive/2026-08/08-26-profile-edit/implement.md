# 实现本人资料编辑 — Implementation Plan

## Start Gate

- 上级父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终planning review后由用户明确批准task.py start；中间父任务本身不作为实现目标。
- Codex保持inline模式，不派发subagent。

## Steps

1. 审查beta56完整Profile edit流程，实时核验当前API/Web capabilities并记录结论；确认 API/Web/image 三个出口均可注入 shared `NetworkAccessPolicy`。
2. 定义ProfileCapabilities、ProfileDraft、minimal ProfilePatch、typed submit outcomes、validation和account revision contract。
3. 实现表单/离开确认、图片选择裁剪、cancel/error状态。
4. 实现安全submit adapter和UserStore/AccountStore原子更新。
5. 增加 capability×dirty-field 组合、verification pending、密码短生命周期、图片/竞态/账号切换/Web安全与Widget测试。
6. 运行analyze/test/build，在受控测试账号验证修改、失败和恢复。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-profile-edit
git diff --check
```

涉及Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- draft dirty/validation/leave guard。
- image MIME/size/pixel/crop/cancel/cleanup。
- server field errors与retry。
- capability/patch field omission、confirmed vs verification pending、password no-prefill/no-persist。
- account switch/base revision race。
- success store sync、failure rollback、Web bridge secret audit。
- system proxy/VPN off 的 API/Web/image route evidence；无私有 fixed IP、SNI/Host rewrite 或第三方反代。

## Risky Files and Rollback Points

- lib/features/profile/edit/、UserStore/AccountStore update、image/WebView plugins、真实账号资料

## Completion Gate

- PRD全部acceptance criteria有真实证据，自动测试/模拟器/真机/真实账号明确区分。
- 不残留placeholder、空操作、隐藏mock、吞错或安全绕过。
- 运行trellis-check，按需更新spec，提交仅包含本任务文件，再finish/archive。

## Execution Record

### Implemented

- Added typed `ProfileCapabilities`, `ProfileDraft`, minimal `ProfilePatch`,
  typed outcomes and bounded text validation.
- Added owner/revision-fenced `ProfileEditController` with cancel, retry,
  field errors, verification-pending handling, ephemeral password clearing and
  exactly-once owned image cleanup.
- Added the authenticated read-only repository adapter, explicit unavailable
  submit outcome, bounded profile-image preprocessor and persistence-first
  `AccountStore`/canonical `UserStore` commit bridge.
- Added the beta56 field order/form, leave confirmation, localized unsupported
  state and a reachable edit action from the actual `profile.MePage` header;
  account settings remains a separate action.

### Compiled and tested

- `/opt/flutter-3.47.0/bin/flutter analyze` — passed with no issues.
- `/opt/flutter-3.47.0/bin/flutter test test/profile_edit_test.dart test/user_profile_test.dart test/settings_test.dart --reporter compact` — 25 tests passed.
- `/opt/flutter-3.47.0/bin/flutter test --reporter compact` — 321 tests passed,
  1 existing `test/icon_font_test.dart` failure caused by
  `MissingPluginException` on `pixivfunc/android_intents/events` in the VM test
  environment; no profile test failed.
- `/opt/flutter-3.47.0/bin/flutter build apk --debug` — passed.
- Debug APK SHA-256: `3800e836c199b57d01c37f51842fc127b4c4af2c3f4c6ea3c8e74bfb898141b1`.

### Device-tested

- MuMu Manager verified instance 0 at `127.0.0.1:16384`, Android 15.0,
  `is_android_started=true`; the second ADB candidate `127.0.0.1:7555` was not
  selected blindly.
- Required preflight: ADB state `device`, SDK/API `35`, global proxy `null`;
  `dumpsys connectivity` showed `wlan0`, `10.0.2.15`, `VALIDATED` and
  `NOT_VPN`. This is a host-NAT Wi-Fi path, not carrier or physical-device
  coverage.
- Installed the debug APK successfully and cold-launched
  `io.github.lopution.pixivfunc/.MainActivity`.
- With the existing signed-in account, navigated Settings → account profile →
  `编辑个人资料`. The page loaded the current account's profile fields
  (`昵称`, `自我介绍`, `网页`, `头像`, `背景图`), rendered the explicit
  unavailable reason, and exposed `保存资料` with `enabled=false`.
- No Save, profile mutation, OAuth re-login, password entry or account switch
  was performed. Sampled app logcat had no `FATAL EXCEPTION`; AndroidRuntime
  lines were the normal ADB helper processes used for launch/UI dump.

`MuMu emulator-tested, not physical-device-tested`

### Real-account and remaining boundary

The real account login/read chain is working and is not recorded as a blocker.
The remaining blocker is the absence of a reviewed official profile mutation
route. API 36 was not available in the verified MuMu image, so API 36 remains
unverified. A real mutation/recovery run and physical-device coverage remain
unverified; no security bypass or fake success path was added.
