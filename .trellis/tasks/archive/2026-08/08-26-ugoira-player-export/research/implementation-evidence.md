# Ugoira implementation evidence

Date: 2026-08-28

## Implemented

- Added strict typed metadata, ZIP central-directory/local-header validation,
  ZIP-slip/duplicate/unknown-entry rejection, compression/size/frame/pixel
  limits, CRC checks, and disk-backed random-access frame reads.
- Added header-before-codec validation, bounded `ui.Image` LRU ownership,
  monotonic deadline playback, visibility/lifecycle suspension, and beta56
  inline cover/play/pause/error surfaces.
- Added a shared app-scoped Pixiv media transport and a MediaStore pending sink
  boundary for GIF post-processing. GIF quantization runs in one owned worker
  isolate, with bounded output, cancellation, cleanup, and one terminal event.

## Unit-tested

- `test/ugoira_test.dart`: metadata/schema errors, ZIP entry/path/count/ratio/
  forged-size checks, image-header and pixel gates, cache disposal, scheduler
  catch-up, worker GIF decode, export success, and owned-output cancellation.
- `test/ugoira_viewer_test.dart`: beta56 cover/play/GIF affordances and
  detail-owned long-press handoff.

## Compiled

- `/opt/flutter-3.47.0/bin/flutter analyze`
- `/opt/flutter-3.47.0/bin/flutter test --reporter compact` (`+237`, exit 0)
- `/opt/flutter-3.47.0/bin/flutter build apk --debug` (success)

## Device-tested

- MuMu manager confirmed running instance `MuMuPlayer-15.0-0`; its configured
  primary `ADB_PORT` is `127.0.0.1:16384`.
- `adb -s 127.0.0.1:16384 get-state` returned `device`; SDK is `35`;
  `http_proxy` is `null`; connectivity reported validated Wi-Fi with
  `NOT_VPN`; WebView is `com.android.webview` 110.0.5481.154.1.
- Debug APK installed and `io.github.lopution.pixivfunc` launched without a
  fatal application crash in the sampled log.
- On API 35, `am start -W` reported `Status: ok` and
  `topResumedActivity=io.github.lopution.pixivfunc/.MainActivity`; a captured
  1080x1920 screen showed the rendered home feed, image cards, and bottom
  navigation. After sending `KEYCODE_HOME`, a second `am start -W` returned
  `LaunchState: HOT` and the resumed screen rendered the same UI. This is a
  real API 35 launch/render/resume check, not a unit-test substitute.
- The rendered home feed and the Settings screen were checked while the
  existing account session was active. Settings displayed the current account
  card, so this run confirms a real signed-in account session on API 35; the
  account identifier is intentionally omitted from repository evidence.
- This is `MuMu emulator-tested, not physical-device-tested`.

## Blockers and unverified scope

- The available MuMu image is API 35, not API 36. API 36 playback/export,
  Android MediaStore behavior, and API 36 WebView behavior remain blockers.
- A real Ugoira work was not opened in this run, so real
  `app-api.pixiv.net` metadata, `pximg` ZIP/frame download, playback, and
  MediaStore GIF acceptance remain unverified. This is an unverified Ugoira
  coverage item, not a signed-in-account blocker; the unit tests use local
  fixtures only and are not evidence of production reachability.
