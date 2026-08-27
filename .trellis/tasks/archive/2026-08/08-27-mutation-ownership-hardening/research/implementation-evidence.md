# Mutation ownership hardening implementation evidence

Date: 2026-08-28

## Implemented

- Added immutable `MutationEnvelope` and `MutationBoundary` carrying the
  account, credential, entity, operation, client mutation ID, creation time,
  network revision and cancellation owner without storing credentials.
- Added bounded `MutationLedger` ownership: exact-operation dedupe, opposite
  operation supersede, cancellation, terminal cleanup and metadata-only
  discard events. It has no durable queue and never restores a pending write;
  a Riverpod rebuild can reopen only an empty ledger while retaining bounded
  discard telemetry.
- Bookmark, Follow and Comments stores/actions/repositories now pass the
  envelope cancellation signal, fence commit/fail by the account/credential/
  network boundary, keep server-confirmed values authoritative, and expose
  pending/confirmed/failed/cancelled lifecycle state. Late responses cannot
  update a switched account or a superseded operation.
- Mutation repositories call `PixivHttpClient` with
  `allowAuthReplay: false`. A shared token refresh may complete once, but a
  non-idempotent request whose body may have been sent is not replayed.
- Added regression tests for envelope identity, duplicate/reverse operations,
  account switching, late responses, cancellation, status transitions,
  cancellation-token forwarding, bounded-ledger reopen and non-replayed POST
  refresh.

## Compiled

- `/opt/flutter-3.47.0/bin/flutter analyze` — passed with no issues.
- `/opt/flutter-3.47.0/bin/flutter build apk --debug` — passed.
- Debug APK SHA-256: `b6a1397c9262b404b1dc8c016f5e1a29388853f5d64f7fef7a1067e496eafe1a`.

## Unit-tested

- Focused mutation/bookmark/network/profile/comments tests — passed.
- Full `/opt/flutter-3.47.0/bin/flutter test --reporter compact` — passed,
  `273` tests.
- `git diff --check` and task validation remain part of the pre-commit gate.

## Device-tested

- MuMu manager-verified serial: `127.0.0.1:16384`.
- Required preflight passed: ADB state `device`, Android SDK `35`, global
  `http_proxy` `null`, active network `WIFI`, capabilities include
  `NOT_VPN`/`VALIDATED`; the package scan found no third-party VPN/proxy
  package (only the Android system VPN dialog package).
- Installed the debug APK with `adb -s 127.0.0.1:16384 install -r` and
  launched `io.github.lopution.pixivfunc/.MainActivity`. The recommended feed
  rendered on screen and the activity reached `Fully drawn`; no fatal Flutter
  exception was observed in the bounded logcat check.
- `MuMu emulator-tested, not physical-device-tested`.

## Real API / account boundary

- The existing authenticated session was retained by the installed app and
  reached the account-backed recommended surface. This validates the
  account-present/UI boundary on the API 35 MuMu run; no account identifier or
  credential was written to this evidence.
- This leaf did not perform a fresh OAuth exchange, forced token refresh, or
  bookmark/follow/comment write against the active account. Those actions are
  therefore unverified for this leaf, not a missing-account blocker; no real
  account data was changed.

## Blockers / limits

- The verified emulator is API 35. API 36 coverage is unavailable in the
  current MuMu image and remains an explicit blocker; this result is not an
  API 36 or physical-device claim.
- MuMu uses host NAT/Wi-Fi, so this evidence does not represent carrier or
  physical-device coverage.
