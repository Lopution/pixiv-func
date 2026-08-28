# Implementation and verification evidence

## Implemented

- Added a version-1 `TransferEnvelope` with canonical standard base64,
  strict JSON keys, bounded payloads, UTC expiry, clock-skew checks, random
  nonce and an explicitly unkeyed SHA-256 corruption checksum.
- Added a secure-storage-backed target-local replay store that persists only
  nonce digests and expiry; no credential or raw nonce is persisted there.
- Added the exact-host App API pre-import verifier, one optional OAuth refresh,
  server-authoritative `/v1/user/detail` metadata and the existing atomic
  `AccountStore` boundary.
- Added the Android clipboard channel with API 33+ sensitive metadata,
  five-minute conditional clear and a fingerprint check that preserves later
  user clipboard content. The Dart adapter intentionally has no generic
  clipboard fallback because that would lose these guarantees.
- Added explicit Settings long-press export and Login help paste UI with
  localized residual-risk wording.

## Compiled and unit-tested

- `flutter analyze` on the changed source/test set: passed with no issues.
- Focused transfer, account-store and HTTP tests: 39 passed.
- Full `flutter test --reporter compact`: 347 tests passed; one unrelated
  existing `test/icon_font_test.dart` test fails because the VM test process
  has no implementation for `pixivfunc/android_intents/events` and raises
  `MissingPluginException`. The same failure reproduces when that test is run
  alone; no transfer test failed.
- `flutter build apk --debug`: passed.
- Debug APK SHA-256:
  `1b982a7fd811d635dc371c517ad426f0419393f1969cd48b5fd2d7f8bfb11bc8`.
- `./gradlew :app:compileDebugKotlin --no-daemon`: passed.

## MuMu emulator-tested, not physical-device-tested

- MuMu Manager verified instance 0 is `127.0.0.1:16384`; the other visible
  endpoint `127.0.0.1:7555` was not selected blindly.
- Required preflight on 2026-08-28: ADB state `device`, SDK/API `35`, global
  proxy `null`; `dumpsys connectivity` showed validated Wi-Fi on `wlan0`,
  `10.0.2.15`, `NOT_VPN`, with the usual host-NAT gateway. This is not carrier
  or physical-device coverage, and no API 36 MuMu image was available.
- Installed the debug APK and cold-launched
  `io.github.lopution.pixivfunc/.MainActivity`.
- With the existing real signed-in account, Settings account-card long press
  showed the localized export confirmation. Login help rendered the explicit
  paste action and warned that other apps may read the clipboard and that the
  format provides neither encryption nor sender authentication.
- A real-account paste attempt was exercised without printing or persisting
  clipboard contents. The account remained available after force-stop and
  relaunch, and the clipboard was empty after the recognized transfer attempt.
  This validates the API 35 app/channel path on one emulator; it is not a
  two-device migration claim.

## Remaining verification boundaries

- API 36 remains unverified because no API 36 MuMu image is available.
- A second independent Android device and physical-device coverage remain
  unverified. The two-device acceptance criterion is therefore recorded as a
  real environment boundary, not silently converted into API 35 success.
- Fresh OAuth/WebView login and mutating account operations were not needed
  for this leaf and were not performed. The existing real account login/read
  chain is not a blocker.
