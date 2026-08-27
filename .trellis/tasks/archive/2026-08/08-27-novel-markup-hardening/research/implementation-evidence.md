# Novel markup hardening implementation evidence

Date: 2026-08-28

## Implemented

- Replaced the novel regex-only projection with a bounded typed markup AST for
  text, `newpage`, chapter, ruby, page/URI jump, Pixiv image, uploaded image,
  unknown, malformed, and budget-exceeded tokens. Raw marker text and
  attributes remain observable; malformed or unsupported input is never
  silently discarded.
- Added allowlisted Pixiv URI validation and validated image load requests.
  The reader never executes HTML or arbitrary URI/path input, and image tokens
  expose identifiers rather than untrusted URL strings.
- Added bounded source/token/marker/diagnostic budgets, cancellable chunked
  parsing, progress callbacks, bounded layout pagination, explicit page and
  chapter boundaries, and cancellation-aware layout progress.
- Added generation/content/chapter/page/cancellation commit fencing to the
  reader. Late or disposed layouts cannot replace the current `PageView`
  state, while the existing horizontal paging, tap zones, font, and theme
  behavior remain intact.
- Preserved loaded novel markup when a metadata-only entity merges into the
  store, preventing a later shallow response from erasing the reader body.
- Updated `.trellis/spec/frontend/state-management.md` with the seven-part
  typed-markup and reader-commit contract.

## Compiled

- `/opt/flutter-3.47.0/bin/flutter analyze` passed with `No issues found!`.
- `/opt/flutter-3.47.0/bin/flutter build apk --debug` passed.
- APK SHA-256 used for the device smoke:
  `27ed3c7f9fac3465cb1a9392d6589486110d55d3888db19aa1d702ee0b928d39`.
- `git diff --check` passed after implementation changes.

## Unit-tested

- Added `test/novel_markup_hardening_test.dart` covering typed token mapping,
  Unicode and raw-attribute preservation, nested/malformed markers, safe jump
  and image validation, parser budgets/progress/cancellation, document layout
  page/chapter boundaries, layout budgets, and stale/disposed reader commits.
- Focused command passed:

  ```text
  /opt/flutter-3.47.0/bin/flutter test test/novel_reader_test.dart test/novel_markup_hardening_test.dart --reporter compact
  ```

  Result: `00:12 +12: All tests passed!`.
- Full suite passed:

  ```text
  /opt/flutter-3.47.0/bin/flutter test --reporter compact
  ```

  Result: `00:17 +266: All tests passed!`.

## Device-tested

- MuMu Manager identified the active primary instance as `127.0.0.1:16384`;
  `127.0.0.1:7555` remained a separate visible candidate and was not selected
  blindly.
- Required preflight passed on the verified serial:

  ```text
  adb devices -l
  adb -s 127.0.0.1:16384 get-state                         -> device
  adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk -> 35
  adb -s 127.0.0.1:16384 shell settings get global http_proxy -> null
  adb -s 127.0.0.1:16384 shell dumpsys connectivity          -> validated Wi-Fi, NOT_VPN
  ```

- The debug APK identified by the SHA above installed successfully and
  `io.github.lopution.pixivfunc/.MainActivity` launched in the foreground.
  Home/recommended and illustration detail rendered, and the sampled logcat
  contained no `FATAL EXCEPTION` or Flutter fatal error.
- This is explicitly `MuMu emulator-tested, not physical-device-tested`.
  The guest is API 35 behind host NAT/Wi-Fi; it is not evidence for API 36,
  physical-device, or three-carrier coverage.

## Real API / account boundary

- A real logged-in account session was present on the MuMu instance; this run
  did not lack an account and does not classify account availability as a
  blocker.
- A fresh Novel API request, OAuth credential submission/token exchange,
  token refresh, or novel mutation was not executed in this device session.
  Those behaviors are therefore unverified for this leaf rather than claimed
  as passed or blocked by account state.
- The native Novel detail route was not reached during the navigation smoke;
  parser, layout, and reader fencing are covered by deterministic unit tests.

## Blockers / limits

- Only an API 35 MuMu image was available. API 36 validation remains a real
  blocker for the API 36 acceptance matrix; no API 36 success is claimed.
- The device result is one MuMu host-network sample with system proxy/VPN
  disabled. It does not establish broad Mainland China availability or
  physical-device coverage.
