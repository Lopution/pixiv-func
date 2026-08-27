# Feed generation commit hardening implementation evidence

Date: 2026-08-28

## Implemented

- Added immutable `FeedRequestContext` carrying `feedKey`, account ID,
  credential revision, generation, page/cursor, cancellation handle, and
  `NetworkRevision`.
- Added bounded `FeedCommitGate` telemetry and active-context checks. Late,
  cancelled, stale, account-boundary, credential-boundary, network-boundary,
  and disposed results cannot invoke their page commit callback.
- Moved shared entity writes behind the accepted page commit for Recommended,
  Ranking, New, Search, and Profile feeds. Repositories/controllers now parse
  page DTOs first; `IllustStore`, `NovelStore`, and `UserStore` are written only
  after the controller accepts the same context.
- Added credential/account revision fencing in `AccountStore`, stable ID
  dedupe and server-order preservation, refresh generation replacement, bounded
  duplicate-cursor rejection, and loading-phase recovery when a current request
  is invalidated by a network boundary.
- Preserved existing feed UI phases, empty/error behavior, selector-specific
  provider families, strict cursor allowlists, and no implicit retry/cache.
- Updated `.trellis/spec/frontend/state-management.md` with the seven-part
  generation-scoped feed commit contract and the async notifier boundary-watch
  gotcha.

## Unit-tested

- `test/feed_generation_commit_test.dart`: 7 focused tests covering refresh
  winning over a late append, same-ID update/duplicate/order/delete behavior,
  repeated cursor rejection before entity/cursor commit, network revision
  invalidation, account switch fencing, disposal fencing, and direct gate
  account/network checks.
- Focused migration/regression command passed:

  ```text
  /opt/flutter-3.47.0/bin/flutter test test/feed_generation_commit_test.dart test/recommended_feed_test.dart test/ranking_feed_test.dart test/new_content_feed_test.dart test/search_catalog_test.dart test/user_profile_test.dart --reporter compact
  ```

  Result: `00:04 +34: All tests passed!`.
- Full suite passed:

  ```text
  /opt/flutter-3.47.0/bin/flutter test --reporter compact
  ```

  Result: `00:20 +260: All tests passed!`.

## Compiled

- `/opt/flutter-3.47.0/bin/flutter analyze` passed with `No issues found!`.
- `/opt/flutter-3.47.0/bin/flutter build apk --debug` passed.
- APK SHA-256 used for the device smoke: `b31670fa26be47835761276696dc4f58d7c079918a66f580893a7f9498a112ac`.
- `python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-27-feed-generation-commit-hardening` passed.
- `git diff --check` passed before evidence capture.

## Device-tested

- MuMu Manager had identified `127.0.0.1:16384` as the verified primary
  instance; `127.0.0.1:7555` remained a separate visible candidate and was not
  selected blindly.
- Required preflight passed on `127.0.0.1:16384`:

  ```text
  adb devices -l
  adb -s 127.0.0.1:16384 get-state -> device
  adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk -> 35
  adb -s 127.0.0.1:16384 shell settings get global http_proxy -> null
  adb -s 127.0.0.1:16384 shell dumpsys connectivity -> validated Wi-Fi, NOT_VPN
  ```

- The debug APK installed successfully, `MainActivity` launched in the
  foreground, and the current logged-in session rendered the real Recommended
  waterfall. A scroll through the feed remained responsive; no app fatal
  exception was present in the sampled logcat.
- This is explicitly `MuMu emulator-tested, not physical-device-tested`.
  The guest is behind host NAT/Wi-Fi and this smoke does not establish API 36,
  physical-device, or three-carrier coverage. The generation race itself is
  covered by deterministic delayed-response unit tests, not claimed as a
  manually reproducible UI race.

## Blockers and unverified scope

- Only an API 35 MuMu image was available; API 36 validation remains a real
  blocker for the API 36 matrix and is not claimed as passed.
- The real logged-in session was present and the Recommended feed loaded; this
  leaf did not perform a fresh OAuth exchange, token refresh, bookmark
  mutation, or account mutation. Those are unverified for this leaf, not a
  missing-real-account blocker.
- No new physical-device or broad Mainland China network coverage claim is
  made. Existing restricted-network evidence remains the source for its wider
  route/account/WebView observations.
