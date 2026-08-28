# Live feasibility evidence — 08-26-live-player

Date: 2026-08-28 (all probes executed on this date; conclusions are date-scoped and must be revalidated if revisited)

Feasibility gate result: **NOT PASSED**. The App API live list is reachable with a real account, but zero live objects are currently returned, so no valid live id, detail payload, or HLS manifest could ever be obtained. Per PRD R10 and implement.md step 2, the probe stops here: no player/codec dependency, no Live feature code, no mock or fixture success path.

## Probe method

- One temporary Flutter entrypoint (`tool/live_probe.dart`, deleted after the run) booted the app's real provider graph in the installed debug app on the verified MuMu instance, so requests went through the production `PixivHttpClient` → `PixivPolicyHttpClient` → `NetworkAccessPolicy` chain with the real account credential from secure storage (existing refresh chain active).
- Output was sanitized at the source: status classification, JSON top-level keys and counts only. No token, cookie, response body, account id, or user content was printed or persisted.
- Client identity sent: `PixivAndroidApp/5.0.234 (Android 11.0; Pixel 5)` (`App-Version 5.0.234`).

## Device-tested (MuMu emulator API 35, not physical-device-tested)

Preflight on `127.0.0.1:16384` passed at probe time:

```text
adb devices -l                                  -> 127.0.0.1:16384 device (selected; 7555 also present, not used)
adb -s 127.0.0.1:16384 get-state                -> device
adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk -> 35
adb -s 127.0.0.1:16384 shell settings get global http_proxy -> null
adb -s 127.0.0.1:16384 shell dumpsys connectivity -> Wi-Fi VALIDATED, NOT_VPN, wlan0 10.0.2.15/24 gw 10.0.2.2 (host NAT)
```

In-app probe run (2026-08-28 15:25, logcat, sanitized):

```text
real_account_present=true
client_identity_app_version=5.0.234
list          ok keys=[live_info, lives, next_url] lives=0 has_next_url=false
list_android  ok keys=[live_info, lives, next_url] lives=0 has_next_url=false
list_ios      ok keys=[live_info, lives, next_url] lives=0 has_next_url=false
detail_id0    http 404
registry_refusal enforced=registry_refused:appApi
```

Reading:

- `GET https://app-api.pixiv.net/v1/live/list` (also `?filter=for_android`, `?filter=for_ios`): HTTP 200, schema top-level keys `live_info,lives,next_url`, `lives` empty, no `next_url`. The endpoint, auth and schema are real and working through the strict policy stack.
- An earlier probe run the same day (14:56–14:57) first observed the expired-token shape (`http 400`, OAuth auth failure classification) and then HTTP 200 after the app's existing single-flight refresh chain ran — the documented refresh behavior, not a new code path.
- `GET /v1/live/detail?live_id=0`: HTTP 404. `live_id=0` is an explicitly invalid probe value; with `lives=0` there is no valid live id to request, so the detail schema cannot be validated either way.
- `registry_refusal`: a policy-client request toward `sketch.pixiv.net` was rejected with `PixivDestinationException(appApi)` before any network I/O — R2a's exact-host enforcement is real at runtime.

External availability (host-side, status codes only, same date):

```text
https://sketch.pixiv.net/            -> HTTP 200
https://sketch.pixiv.net/lives       -> HTTP 307 -> /lives/closed -> final HTTP 410
https://sketch.pixiv.net/@pixiv/lives-> HTTP 307 -> /lives/closed -> final HTTP 410
```

These are current-day external observations only. The web Sketch surface redirects its live index to a closed page; this does not by itself prove the App API live service is retired (the App API list still answers 200), but it shows no public live directory is browsable today.

Restored-state screenshot evidence (`research/screenshots/`):

- `01-restored-home-no-live-entry.png`: after the probe the normal debug APK (built this session, SHA-256 under Compiled) was reinstalled on the verified instance, cold-launched (`Status: ok`), and screenshotted. The Recommended waterfall renders with the real signed-in account's feed, and the bottom bar shows home / ranking / novel / search / settings with **no Live entry** — visible confirmation that no player page or Live UI exists after this task, matching the gate decision.
- No Live playback screenshot exists because there is nothing to play; fabricating one would violate R10.

## Implemented

- Nothing. The feasibility gate stopped at implement.md step 2 by design.
- Dependency diff proof for AC "dependency diff 证明 player 依赖只在 feasibility 通过后加入": `pubspec.yaml`/`pubspec.lock` gained no video/HLS/player dependency in this task (zero diff), and `lib/` gained no live feature directory. No mock, fixture, placeholder, or no-op player path exists.
- No fixed-IP, proxy, TLS bypass, or Host/SNI rewriting was introduced; the only network code touched by the probe was the pre-existing shared policy stack, used as-is.

## Compiled

- `flutter pub get`: OK.
- `flutter analyze`: `No issues found!`.
- `flutter build apk --debug`: passed. Installed-on-device APK SHA-256: `21d863316d1b488b6b2f944276f48c711657db07b30642fe175423d9980d2ec3` (same build used for the restored-state screenshot).
- `git diff --check` and `task.py validate` run at completion.

## Unit-tested

- No new live tests: the gate stopped before any product code existed, and there is nothing to unit-test without inventing a fixture-driven player (forbidden by R10).
- Full suite: `flutter test` → `00:14 +348: All tests passed!`.
- Pre-existing red found and fixed during this gate run, unrelated to Live: `test/icon_font_test.dart` golden test failed on a clean HEAD because the platform-boundary hardening commit (`ff4dd2e`) added an `EventChannel` subscription to `HomePage.initState` that the icon font test never stubbed, so the `MissingPluginException` escaped through the golden capture's real async zone. Fixed test-side by injecting a stub `AndroidIntentSource` through `HomePage`'s own `intentSource` seam (platform boundary stub only; no production code changed).

## Blockers and unverified scope

- **No live object available (external blocker)**: `lives=0` on all three list filters with a real account means list→detail→HLS playback evidence is impossible today. Detail schema, HLS variant/redirect host behavior, and the whole player surface (R3–R9) remain unimplemented and unverified — by decision, not by omission.
- **HLS host allowlist**: without a real manifest URL, no HLS host can be evidenced into `PixivDestinationRegistry`; adding speculative hosts would violate R2a and stays undone.
- **API 36**: only MuMu API 35 was available; API 36 device validation remains the standing blocker recorded across the replica v1 leaves.
- **Physical device / carrier coverage**: MuMu host-NAT Wi-Fi only; no physical-device or three-carrier claim is made.
- If the gate is ever retried, every conclusion above must be revalidated on that day; a future `lives>0` list response reopens implement.md steps 3–7 rather than licensing mock playback.
