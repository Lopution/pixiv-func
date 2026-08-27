# Restricted compatibility network implementation evidence

Date: 2026-08-28

## Implemented

- Added a central, exact-host `PixivDestinationRegistry` with purpose-scoped destinations for App API, OAuth, accounts Web, Pixiv Web, and images. Suffixes, trailing-dot names, IP literals, userinfo, fragments, non-HTTPS URLs, and non-443 ports are rejected.
- Added a shared `NetworkAccessPolicy` with `Automatic` direct-first routing, `DirectOnly`, network revisions, cancellation, route health isolation, bounded diagnostics, and a failure taxonomy. Only safe transport failures can select another strict route; HTTP, auth, rate-limit, parse, cancellation, TLS, and certificate failures do not trigger fallback.
- Added strict native transport and secure resolver contracts. Candidate IP connections retain the original URL hostname for TLS/SNI/Host and system certificate validation. Resolver candidates are validated as public, host-bound, revision-bound A/AAAA results with bounded TTL/response handling. DoH is optional and is not the default production route.
- Integrated the shared factory with `PixivHttpClient`, `OAuthService`, image cache file service, and download transport. Removed fixed Pixiv IP/mirror URL rewriting and the old no-op login proxy switch; the visible switch now controls the real policy mode.
- Added a fail-closed ECH capability gate and a direct, exact-host WebView route policy. Unsupported WebKit reverse-bypass capability does not install a listener or proxy override; the direct WebView path remains available.
- Preserved mutation/token no-replay behavior and cancellation through the request boundary. Diagnostics exclude query strings, cookies, tokens, request/response bodies, and complete user addresses.

## Unit-tested

- `test/restricted_compat_network_test.dart`: 12 focused tests covering exact-host validation, failure classification, direct-first safe GET fallback, POST no-replay, HTTP/auth/TLS/cancel no-fallback, diagnostic redaction, secure DNS parsing, revision/mode pool invalidation, private candidate rejection, cancellation, shared factory identity, ECH fail-closed behavior, and WebView lifecycle.
- Focused and regression commands passed:

  ```text
  /opt/flutter-3.47.0/bin/flutter test test/restricted_compat_network_test.dart --reporter compact
  /opt/flutter-3.47.0/bin/flutter test test/restricted_compat_network_test.dart test/pixiv_http_client_test.dart test/settings_test.dart test/download_manager_test.dart test/login_navigation_test.dart --reporter compact
  ```

- Full suite passed: `00:40 +253: All tests passed!`.

## Compiled

- `/opt/flutter-3.47.0/bin/flutter analyze` passed with `No issues found!`.
- `/opt/flutter-3.47.0/bin/flutter build apk --debug` passed.
- APK SHA-256 for the device run: `fbd477c97cc97433a9493191023c14c991c83b6d5657d03d881cc64a8a7ba54d`.
- `git diff --check` and the task validation were run after implementation changes.

## Device-tested

- MuMu Manager identified the active primary instance as `127.0.0.1:16384`; `127.0.0.1:7555` was also visible as another candidate and was not selected blindly.
- Required preflight on the verified serial passed:

  ```text
  adb devices -l
  adb -s 127.0.0.1:16384 get-state       -> device
  adb -s 127.0.0.1:16384 shell getprop ro.build.version.sdk -> 35
  adb -s 127.0.0.1:16384 shell settings get global http_proxy -> null
  adb -s 127.0.0.1:16384 shell dumpsys connectivity
  ```

- Connectivity showed validated Wi-Fi, `NOT_VPN`, and no external proxy application package. The MuMu guest exposed IPv4 and IPv6 configuration. Android WebView was `110.0.5481.154.1`.
- The APK identified by the SHA above installed and launched on the verified instance. Both cold and hot launch returned `Status: ok`; the second launch returned `LaunchState: HOT`, and `MainActivity` was foreground.
- The current account session loaded the Recommended waterfall with real Pixiv image cards. Settings showed the existing logged-in account card. Account management showed the existing account, the login page showed `自动兼容网络` enabled, and the accounts WebView reached the existing-session page with options to continue that account or log in to another account. No credential or account mutation was performed.
- This is explicitly `MuMu emulator-tested, not physical-device-tested`. The observation is one MuMu guest behind host NAT/Wi-Fi; it is not evidence for three-carrier coverage or physical-device coverage.

## Blockers and unverified scope

- Only an API 35 MuMu image was available. API 36 clean-device validation remains a real blocker for the API 36 acceptance matrix; no API 36 success is claimed.
- The existing real account session is confirmed by the settings/account-management UI and accounts WebView. A fresh OAuth credential submission, token exchange, and token refresh were not executed in this run because doing so would alter the active account session; those behaviors remain unverified, not blocked by a missing account.
- Real bookmark mutation was not executed. Normal download, Ugoira/ZIP extraction, and MediaStore persistence were not opened in this network validation run; their shared transport integration is unit-tested, while device acceptance remains unverified here.
- WebView loopback reverse-bypass and ECH are not claimed as available. The current WebKit capability result is fail-closed/direct-only, and the ECH gate is not approved without endpoint, transport, trust, cancellation/stream/pool, and API 36 evidence.
- The run used one MuMu host-network sample with system proxy/VPN disabled. It does not establish broad Mainland China availability or mobile/Unicom/Telecom coverage.
