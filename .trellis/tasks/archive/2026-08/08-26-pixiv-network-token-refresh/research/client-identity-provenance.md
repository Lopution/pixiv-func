# Pixiv client identity / network provenance (2026-08-26)

## Verified sources (same session as oauth-verification.md)

- `shenshengkafei/pixiv_dart_api` HEAD `lib/pixiv_auth.dart`
- `MnyaCat/pixiv_dart` HEAD `lib/src/client/auth_client.dart`

## Centralized identity (single source: `PixivClientIdentity`)

- OAuth client: `MOBrBDS8blbauoSck0ZfDbtuzpyT` /
  `lsACyCD94FhDUtGTXi3QzcFE2uU1hqtDaKeqrdwj` (unchanged across all clients).
- OAuth token endpoint: `https://oauth.secure.pixiv.net/auth/token`.
- App API base: `https://app-api.pixiv.net`.
- Request headers: `User-Agent: PixivAndroidApp/<ver> (Android 11; <device>)`,
  `App-OS: android`, `App-OS-Version: 11.0`, `App-Version`, `Accept-Language`.
  UA app version observed 5.0.234 and 6.21.1 across current clients; we pin
  5.0.234 (matches OAuth task) in one place.
- beta56 routed traffic via fixed IP `210.140.92.183` with spoofed `Host` —
  forbidden on the normal path (legacy/emergency only, separate task).

## Security posture

- Transport: `package:http` over system DNS, direct HTTPS, system CA
  verification; no global cleartext, no `badCertificateCallback` override.
- Token refresh: per-account single-flight gate; 401 handling compares the
  request's token against the current stored token before refreshing.
