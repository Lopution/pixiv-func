# Download and Ugoira recovery hardening implementation evidence

Date: 2026-08-28

## Implemented

- Added immutable download submission snapshots, account/credential/network/destination ownership, group state aggregation, explicit retryable/orphaned recovery states, bounded persistence and opaque output owners.
- Added exactly-once terminal/cleanup guards, owner checks across streaming and finalize, MediaStore pending-row recovery hooks, `Retry-After` preservation and stable failure classification.
- Added API 29+ MediaStore pending output ownership markers, exact row-URI finalize/owner-checked abort and recovery cleanup bridge. Unknown pending rows are reported rather than guessed at or deleted. Ugoira export now uses the same owner boundary, bounded output writes, cancellation checks and an observable `finalizing` phase.
- Added a separate versioned Ugoira recovery namespace and `ugoira-output-` owner prefix. Ugoira jobs persist immutable submission metadata before and during output, restore terminal records, and convert interrupted active post-processing to explicit `orphaned` state; startup scans this namespace before normal media recovery and never auto-retries a synthetic GIF URL.
- Added focused recovery tests and retained compatibility for existing unowned in-memory unit fakes.

## Compiled

- `flutter analyze` — passed: `No issues found!`
- `flutter build apk --debug` — passed after the Android API 35-compatible `MediaStore.MediaColumns.TITLE` owner marker adjustment, exact-row URI handling, insertion rollback and pending scan updates.
- Debug APK SHA-256 for the latest build: `389b04ca7825483776452d8a1267f797d16c1e85da96a050a766d478eecaf314`.

## Unit-tested

- `flutter analyze` — passed: `No issues found!`.
- `flutter test --concurrency=1 test/download_recovery_test.dart test/download_manager_test.dart test/ugoira_recovery_test.dart test/ugoira_test.dart --reporter compact` — 56 passed (download recovery 10, manager 30, Ugoira recovery 1, Ugoira 15).
- `flutter test test/download_manager_test.dart` — 30 passed.
- `flutter test test/ugoira_test.dart` — 15 passed, including the `finalizing` event and logout-owner assertions.
- `flutter test --concurrency=1 --reporter compact` — 285 passed.
- `dart format ...` on the changed Dart files — passed with 0 files changed.
- Task validation and `git diff --check` — passed before commit; rerun after staging the final leaf set.

## Device-tested

MuMu emulator-tested, not physical-device-tested.

- Verified serial: `127.0.0.1:16384`; `adb get-state` returned `device`.
- Before the latest device run, executed `adb devices -l`, `get-state`,
  `getprop ro.build.version.sdk`, `settings get global http_proxy` and
  `dumpsys connectivity`; the candidate list contained `127.0.0.1:7555` and
  `127.0.0.1:16384`, and the verified MuMu instance used here was
  `127.0.0.1:16384`.
- MuMu emulator API 35 (`ro.build.version.sdk=35`), global proxy `null`, active Wi-Fi `VALIDATED`, `NOT_VPN`, emulator/NAT address `10.0.2.15`; the full preflight was rerun immediately before installing the latest APK.
- Installed the latest debug APK (`389b04ca...`) and launched `io.github.lopution.pixivfunc/.MainActivity`; `topResumedActivity` was `MainActivity`, the feed rendered real Pixiv images, detail opened, and no app `FATAL EXCEPTION` was observed.
- The existing logged-in account was visible in Settings as `傅易安 / 1048052188@qq.com`. The download-mode UI opened and a real pending MediaStore path under `Pictures/PixivFunc` was observed in logcat when the image download was submitted.
- With the fixed bridge, real image downloads reached `is_pending=0`: row `1000000050` reported `137440675_p0.jpg`, `title=137440675_p0`, and `is_pending=0`; the final 14,077,254-byte file was visible under `Pictures/PixivFunc`. A second run on the latest APK reported row `1000000053`, `title=133270021_p0`, `is_pending=0`, with a 3,765,900-byte final file in the same directory. The task page showed both successful rows as `已完成`.
- With the latest APK, a fresh real-account download of illust `135211495` produced MediaStore row `1000000072`, `title=135211495_p0`, `is_pending=0`; `/sdcard/Pictures/PixivFunc/135211495_p0.png` was present with 6,370,283 bytes. The task page showed this row as `已完成` together with the earlier successful rows.
- On the latest APK, a fresh real-account image download produced MediaStore row `1000000087`, `title=132029540_p0`, `is_pending=0`; `/sdcard/Pictures/PixivFunc/132029540_p0.jpg` was present with 1,220,600 bytes. The task page showed it as `已完成`.
- On the latest APK, searching `ugoira` opened a real Ugoira detail for illust `148973686`; the viewer loaded its remote metadata/ZIP, long-press entered download mode, and `保存 GIF` finalized MediaStore row `1000000097`, `title=148973686_p0`, `is_pending=0`. `/sdcard/Pictures/PixivFunc/148973686_p0.gif` was present with 437,190 bytes; a host-side read identified it as `GIF89a`, 600x510. Logcat showed the owned `.pending-...-148973686_p0.gif` move to the final path.
- Re-launching the fixed build and opening the task manager recovered the prior persisted pending row `1000000049` through the exact row URI and removed its stale file. The earlier test row without a persisted ID was separately removed only after its owner marker was checked.
- The latest task-page capture is `/tmp/pixiv-media-recovery-latest-task-page.png`; old failed rows remain visible as historical task records and were not treated as current MediaStore success evidence. Account confirmation was captured in `/tmp/pixiv-media-recovery-latest-settings.png`.

## Real API / account boundary

- This run used the existing real logged-in app session for feed/detail/search/Ugoira metadata and ZIP/image/GIF download submissions. It did not perform a fresh OAuth login, token refresh, bookmark, follow, comment or profile mutation solely for evidence collection.
- Therefore the existing account chain is not a blocker; fresh OAuth and non-read-only mutation outcomes are simply not claimed by this leaf.
- The observed network is MuMu through host NAT, not three-carrier physical-device coverage. The result is scoped to this emulator, host network, Android API 35 and the observed app/WebView/runtime route.

## Blockers / limits

- API 36 device validation was not available; retain API 36 as an explicit platform-coverage blocker. API 35 was still built, installed and exercised, including normal image and Ugoira GIF MediaStore finalization.
- MuMu's collection query omitted the synthetic pending IDs while exact row URI queries returned them; the bridge now uses exact row URI for finalize and owner-checked abort. Invalid shell projections and the Android `content` helper's shutdown SIGSEGV were inspection failures, not app crashes.
- No claim is made for physical devices, all mainland networks, API 36, fresh OAuth, or non-read-only mutation success.
