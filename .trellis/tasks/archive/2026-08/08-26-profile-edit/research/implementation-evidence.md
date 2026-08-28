# Profile edit implementation evidence

Date: 2026-08-28 (Asia/Shanghai)

## Implemented

- Typed profile draft/patch/capability/outcome contracts cover field-level
  support, minimal dirty-field submission, verification-pending, server field
  errors and explicit unavailable state.
- `ProfileEditController` fences load/submit/commit by account, credential and
  network revisions; it cancels stale work, keeps field input on validation
  errors, clears ephemeral passwords in `finally`, and releases owned image
  selections on replacement, cancellation, disposal and terminal outcomes.
- The production repository intentionally uses the existing authenticated
  `UserRepository.fetchDetail` read route and returns a typed unavailable
  submit outcome because no reviewed official profile update route was found.
- Confirmed metadata is committed persistence-first to `AccountStore` and then
  merged into canonical `UserStore`; pending, failed and canceled outcomes do
  not update confirmed metadata.
- The reachable beta56-shaped form is connected from the actual profile page;
  unsupported fields and the disabled Save state are visible to the user.

## Compiled

- `/opt/flutter-3.47.0/bin/flutter analyze` — `No issues found!`.
- `/opt/flutter-3.47.0/bin/flutter build apk --debug` — passed.
- APK SHA-256: `3800e836c199b57d01c37f51842fc127b4c4af2c3f4c6ea3c8e74bfb898141b1`.

## Unit-tested

- Focused command:

  ```text
  /opt/flutter-3.47.0/bin/flutter test test/profile_edit_test.dart --reporter compact
  ```

  Result: 11 tests passed.
- Combined profile integration command:

  ```text
  /opt/flutter-3.47.0/bin/flutter test test/profile_edit_test.dart test/user_profile_test.dart test/settings_test.dart --reporter compact
  ```

  Result: 25 tests passed.
- Full command:

  ```text
  timeout 240s /opt/flutter-3.47.0/bin/flutter test --reporter compact
  ```

  Result: 321 tests passed and 1 existing `test/icon_font_test.dart` test
  failed when the VM attempted to listen to `pixivfunc/android_intents/events`
  without a registered plugin, producing `MissingPluginException`. No profile
  test failed. This is retained as a full-suite environment/plugin limitation,
  not converted to a pass.

## Device-tested

### MuMu verification

MuMu Manager instance 0 reported:

```text
adb_host_ip=127.0.0.1
adb_port=16384
android_version=15.0
is_android_started=true
player_state=start_finished
```

The required preflight was run against the verified serial:

```text
adb devices -l
adb -s 127.0.0.1:16384 get-state
adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk
adb -s 127.0.0.1:16384 shell settings get global http_proxy
adb -s 127.0.0.1:16384 shell dumpsys connectivity
```

Observed results: both candidate endpoints were visible as ADB devices; MuMu
Manager selected `127.0.0.1:16384`, state was `device`, SDK/API was `35`,
`http_proxy` was `null`, and the active `wlan0` network had `10.0.2.15`,
`VALIDATED` and `NOT_VPN` capabilities. The route is MuMu host NAT Wi-Fi, not
physical-device or three-carrier coverage.

### Read-only account flow

```text
adb -s 127.0.0.1:16384 install -r build/app/outputs/flutter-apk/app-debug.apk
adb -s 127.0.0.1:16384 shell am force-stop io.github.lopution.pixivfunc
adb -s 127.0.0.1:16384 shell monkey -p io.github.lopution.pixivfunc 1
```

Installation succeeded and the cold launch resumed
`io.github.lopution.pixivfunc/.MainActivity`. The existing signed-in account
was visible in the Settings account card (account identifiers are intentionally
omitted from this repository evidence). The flow then reached the real profile
page and the new `编辑个人资料` action.

The profile editor UI dump showed:

- `编辑个人资料` title and `取消` back action;
- current account field values for `昵称`, `自我介绍` and `网页`;
- `头像` and `背景图` image entries;
- the explicit reason `Pixiv profile editing has no approved App API or Web adapter in this build`;
- `当前通道不支持此字段` on each unsupported field;
- `保存资料` with `enabled=false`.

No Save tap, profile mutation, OAuth re-login, password entry or account
switch was performed. The sampled app logcat contained no app crash/FATAL
exception. AndroidRuntime records were only the normal `monkey`/`uiautomator`
helper process lifecycle.

`MuMu emulator-tested, not physical-device-tested`

## Real API/account boundary

The live signed-in account successfully exercised the authenticated read path
that populates the editor. This confirms the account login chain is not a
blocker. Fresh OAuth/token refresh was intentionally not repeated. A real
profile mutation and recovery were not run because the official mutation route
is not established and no harmless live test value/explicit mutation
authorization was supplied.

The route research recorded direct unauthenticated checks of the official
profile URLs and app API behavior, but did not persist cookies, tokens or
response secrets. No password scraping, cookie DOM access, third-party proxy,
fixed-IP route, Host/SNI rewrite or TLS relaxation was introduced.

## Unverified / blockers

- Product capability blocker: reviewed official App API or approved Web
  profile-update contract is still absent; the production adapter remains
  explicitly unavailable.
- Real mutation/recovery, API 36 and physical-device testing are unverified.
- Full suite has the existing VM plugin failure described above; focused
  profile and neighboring profile/settings tests pass.
