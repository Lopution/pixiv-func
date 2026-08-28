# Android platform boundary implementation evidence

Date: 2026-08-28 (Asia/Shanghai)

## Implemented

- Added strict Dart `AndroidIntentInput`/`AndroidIntentResult` validation in
  `lib/core/platform/intent_router.dart`. VIEW routes require the exact
  allowlisted Pixiv URI shapes; SEND requires one `android.intent.extra.STREAM`
  content URI, a readable grant, a concrete image MIME subtype and a positive
  10 MiB limit. Malformed channel maps and unexpected extras become typed
  rejections.
- Added `AndroidIntentChannel` and `WebKitCapabilityChannel` to the Android
  activity. The native bridge forwards metadata only and checks a content URI
  with `ContentResolver`; it does not grant additional permissions or copy file
  contents into the channel.
- Added versioned `WebViewRouteSession`, exact destination host snapshots,
  network-revision fencing, capability/adapter gates, owner leases and
  idempotent lifecycle cleanup. Concurrent owners share one adapter handle, and
  a handle that completes after session shutdown is closed immediately. The
  production provider has no loopback adapter, so it remains direct-only and
  fail-closed; the supported and unsupported capability paths are covered by
  fakes in the focused tests.
- Added PKCE state generation and one-use callback validation, login WebView
  navigation rejection, lifecycle cancellation, and lifecycle-aware root
  double-back handling.
- Added the AndroidX WebKit dependency and capability probe without changing
  TLS validation, SNI, Host headers, fixed-IP security decisions or global
  proxy settings. Existing non-exported FileProvider and MediaStore contracts
  remain bounded by the merged manifest and recovery owners.

## Compiled

- `flutter analyze`: passed with `No issues found!`.
- `flutter build apk --debug`: passed.
- APK SHA-256 at final device validation: `01b96ab7cb0a60efb06987f91a3e60a631552e0a8be993224fc270f153e1435e`.
- Merged package inspection showed `minSdk=24`, `targetSdk=36`, the
  `io.github.lopution.pixivfunc.fileProvider` provider with
  `exported=false`, and no storage/install/cleartext permission in the
  requested permission set.

## Unit-tested

- The pre-implementation boundary suite was run first and failed on the
  missing route/session/intent/state contracts, establishing the regression
  boundary before implementation.
- Focused tests passed for `android_platform_boundary_test.dart`,
  `intent_router_test.dart`, `oauth_service_test.dart`,
  `root_back_coordinator_test.dart` and `login_navigation_test.dart`.
- The focused cases cover exact host/revision/owner cleanup, unsupported and
  supported capability gates, duplicate release, callback state and one-use
  behavior, malformed/ambiguous URI, illegal SEND payloads, MIME/permission/
  size limits, and root lifecycle/back handling.
- Final test gate: every one of the 37 `test/*_test.dart` files passed in
  separate serial `flutter test --concurrency=1` invocations, including the
  two new loopback race tests. A combined full-suite attempt also exposed the
  repository's existing real-socket environment-flaky cancellation timeout
  and a later cross-file test-harness stream race; the affected tests passed
  when run independently and the split gate completed successfully.

## Device-tested

All Android checks used the verified MuMu endpoint `127.0.0.1:16384`, not the
other visible ADB endpoint. Required preflight was run before each validation
sequence:

```text
adb devices -l
adb -s 127.0.0.1:16384 get-state
adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk
adb -s 127.0.0.1:16384 shell settings get global http_proxy
adb -s 127.0.0.1:16384 shell dumpsys connectivity
```

Observed results: the verified serial was `device`, SDK/API `35`, global
`http_proxy` was `null`, and the active Wi-Fi network was `VALIDATED` and
`NOT_VPN`. The guest used host-NAT networking (`wlan0`, IPv4 `10.0.2.15/24`,
IPv6 addresses present, default gateway `10.0.2.2`). This is MuMu guest/host
NAT evidence, not three-carrier or physical-device coverage.

`MuMu emulator-tested, not physical-device-tested`.

The debug APK installed and launched as
`io.github.lopution.pixivfunc/.MainActivity` with no app `FATAL EXCEPTION`.
With the existing real account still present, Settings displayed
`傅易安 / 1048052188@qq.com`. Opening Account Management → Add Account → Login
opened the real Android WebView. The page exposed the existing Pixiv session,
including `此账号正在pixiv.net登录中`, `傅易安`, and `继续使用此账号`; the WebView
title was `登录 | pixiv`. No login submit, account switch/removal, credential
exchange or token refresh was performed.

The WebView provider was confirmed with `dumpsys webviewupdate` as
`com.android.webview 110.0.5481.154.1`. Returning Home/backgrounding the WebView
and relaunching showed the visible `页面已暂停，请重新打开` error. On the logged-in
Home route, the first back showed `再按一次退出`; a second back within the
window moved the top activity to `app.lawnchair/.LawnchairLauncher`, after which
the app relaunched normally. These checks exercised WebView/page lifecycle and
root back behavior without mutating the account.

## Unverified and blockers

- No API 36 MuMu image was available, so the API 36 emulator/real-device
  acceptance criterion remains a real blocker. API 35 evidence above is kept
  explicitly separate and is not claimed as API 36 evidence.
- No physical device or carrier-specific sample was used. The network result is
  limited to the dated MuMu host-NAT route and its observed IPv4/IPv6 state.
- A fresh OAuth code submission/token exchange and token refresh were not
  executed because the emulator already has a real logged-in account. This is
  an unverified mutation path, not a blocker caused by missing account
  credentials; the existing real-account WebView chain was verified.
- The production loopback adapter is intentionally absent because the API
  35/WebKit capability result does not authorize an unbounded compatibility
  listener. Direct strict WebView navigation works; loopback remains an
  explicit capability blocker rather than a silent fallback.
