# Reverse-image search implementation evidence

Date: 2026-08-28 (Asia/Shanghai)

## Implemented

- Added one bounded `ReverseImageSearchController` for the in-app picker and
  Android `ACTION_SEND image/*` entry points. Both paths pass an opaque
  `content://` reference through the platform adapter, copy to app-private
  cache, validate the actual image header and expose the same preview/search
  state machine.
- Added PNG/JPEG/GIF/WebP signature and dimension parsing, concrete MIME
  matching, 10 MiB encoded-size, 8192 dimension and 16 MiB pixel-budget limits,
  permission/reference validation, exactly-once owned-temp cleanup and visible
  cleanup failures. The platform bridge never sends image bytes, cookies or
  credentials to Dart diagnostics or provider code.
- Added typed provider capability/outcome/failure contracts, strict HTTPS
  result URL parsing, similarity sorting, Pixiv-ID/external-URL deduplication,
  visible unavailable-provider UI, cancel/retry/error states, and the shared
  Pixiv detail/external-link result routing for a future approved provider.
- Added the Android picker/content-URI bridge with a 64 KiB copy buffer,
  app-private `cache/reverse_image_inputs` ownership boundary and safe external
  URL launch validation. Added the common intent source integration so a valid
  shared image opens the same reverse-image page.
- Updated the frontend state-management spec with the input/provider contract.

## Compiled

- `/opt/flutter-3.47.0/bin/flutter analyze` — passed: `No issues found!`.
- `/opt/flutter-3.47.0/bin/flutter build apk --debug` — passed:
  `✓ Built build/app/outputs/flutter-apk/app-debug.apk`.
- Final debug APK SHA-256:
  `070262083ba380eb8e0f3c7db792d346e048eef25ec125475daac170fd03d2a8`.
- Kotlin compilation was rerun after the final null-safe URI policy changes;
  the APK installed successfully on the verified MuMu serial.

## Unit-tested

- The reverse-image boundary tests were written before the implementation
  contract was complete; the first run failed at compile time on the missing
  input/provider contracts. After implementation and regression additions:
  `/opt/flutter-3.47.0/bin/flutter test test/reverse_image_search_test.dart
  test/reverse_image_search_page_test.dart --reporter compact` — 12 tests
  passed (10 core lifecycle/mapper tests and 2 widget-flow tests).
- Coverage includes real PNG validation, MIME mismatch, malformed/pixel-budget
  rejection, path-free error/cleanup behavior, permission loss before copy,
  cancellation/rate-limit cleanup, safe result mapping, and both picker/SEND
  preview flows with a visible unavailable-provider failure.
- A serial whole-suite attempt was made with one `flutter test
  --concurrency=1` invocation per `test/*_test.dart`. Existing
  environment-sensitive tests did not complete cleanly: the download manager
  real-socket cancellation case timed out after 30 seconds, and the OAuth
  no-live-session exchange case entered the same external-network wait. The
  attempt was stopped after retaining those failures; this is not reported as
  a full-suite pass. The affected non-new tests had passed independently in
  the earlier hardening gates.

## Device-tested

MuMu emulator-tested, not physical-device-tested.

- MuMu Manager identified the active instance as `127.0.0.1:16384` (Android
  15.0, started); `127.0.0.1:7555` was also visible and was not selected.
  Required preflight was run immediately before the final validation:

  ```text
  adb devices -l
  adb -s 127.0.0.1:16384 get-state                 -> device
  adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk -> 35
  adb -s 127.0.0.1:16384 shell settings get global http_proxy -> null
  adb -s 127.0.0.1:16384 shell dumpsys connectivity
  ```

- Connectivity showed validated Wi-Fi, `NOT_VPN`, guest `wlan0` IPv4
  `10.0.2.15/24`, IPv6 addresses and host-NAT gateway `10.0.2.2`. This is one
  MuMu host-network sample, not physical-device or three-carrier coverage.
- The final APK installed and cold-launched with `Status: ok`; the foreground
  activity was `io.github.lopution.pixivfunc/.MainActivity`. The existing real
  signed-in account session rendered the real Home feed; no fresh OAuth,
  account switch/removal, token refresh, bookmark, follow, comment or profile
  mutation was performed.
- Search → `反向搜图` showed the explicit unavailable-provider reason, privacy
  notice and `选择图片`. The native `ACTION_OPEN_DOCUMENT` picker opened with
  `image/*`; selecting the existing `148973686_p0.gif` returned a real preview
  showing `600 × 510 · 426.9 KiB` and `开始反向搜图`.
- A valid MediaStore image URI was queried without printing its bytes:
  `content://media/external/images/media/1000000097` (`148973686_p0.gif`,
  437190 bytes). The following direct intent entered the same prepared page:

  ```text
  adb -s 127.0.0.1:16384 shell am start -W -a android.intent.action.SEND \
    -t image/gif \
    --eu android.intent.extra.STREAM content://media/external/images/media/1000000097 \
    --grant-read-uri-permission -n io.github.lopution.pixivfunc/.MainActivity
  ```

  The page again showed `图片已准备好`, `600 × 510 · 426.9 KiB` and
  `开始反向搜图`, proving the native SEND path and shared preview flow on API
  35. After pressing `取消`, `run-as io.github.lopution.pixivfunc find
  cache/reverse_image_inputs -maxdepth 1 -type f -print` returned no files.
- The unavailable provider terminal state was visible on device and no app
  `FATAL EXCEPTION` or `AndroidRuntime` application error was observed in the
  sampled log after launch, picker, SEND and cleanup operations.

## Real provider/account boundary

- The dated official-provider audit is in `research/provider-feasibility.md`.
  SauceNAO was challenge-gated at its official API entry; TinEye's documented
  multipart API requires a product-owned key and reviewed legal/privacy
  boundary; Google Lens redirected to its consumer page without a stable
  result-card contract. No real reverse-image upload was attempted, so no
  provider result-card or Pixiv hydration success is claimed.
- The existing Pixiv account is real and active on the MuMu app session. It is
  not a missing-account blocker. Fresh OAuth/token-refresh and mutating account
  operations are simply outside this read-only evidence run.

## Unverified and blockers

- Provider-success criteria (real upload, 429/network/TLS transport against an
  approved service, Pixiv hydration and result-card acceptance) remain blocked
  by the missing approved provider/credential/privacy decision. The app keeps a
  visible terminal unavailable state; it does not scrape HTML, replay a
  challenge, bundle a key, or show mock results.
- No API 36 MuMu image was available. API 36 validation remains an explicit
  platform-coverage blocker; the evidence above is specifically API 35.
- No physical device or carrier-specific sample was used. The network result
  is limited to the dated MuMu guest behind host NAT.
- The full-suite command was attempted but is not a pass because the existing
  real-socket/OAuth tests waited on the current environment. The new
  reverse-image focused suite, analyzer, debug build and both real Android
  input paths passed.
