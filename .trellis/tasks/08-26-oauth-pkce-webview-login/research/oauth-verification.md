# Pixiv OAuth PKCE real-time verification (2026-08-26)

## Sources checked

1. `shenshengkafei/pixiv_dart_api` (HEAD, `lib/pixiv_auth.dart`) — maintained
   successor of the `pixiv_dart_api` dependency beta56 vendored locally.
2. `MnyaCat/pixiv_dart` (HEAD, `lib/src/client/auth_client.dart`).

Both independent implementations agree on the following current facts.

## Verified OAuth contract

- Authorize page: `https://app-api.pixiv.net/web/v1/login`
  with query `code_challenge=<S256>`, `code_challenge_method=S256`,
  `client=pixiv-android`.
- WebView success redirect: `pixiv://account?code=<authorization_code>`.
- Token endpoint: `https://oauth.secure.pixiv.net/auth/token` (POST,
  form-urlencoded).
- Exchange body: `client_id`, `client_secret`, `code`, `code_verifier`,
  `grant_type=authorization_code`, `include_policy=true`,
  `redirect_uri=https://app-api.pixiv.net/web/v1/users/auth/pixiv/callback`.
- Client identity (public, embedded in every OSS Pixiv client):
  - `client_id = MOBrBDS8blbauoSck0ZfDbtuzpyT`
  - `client_secret = lsACyCD94FhDUtGTXi3QzcFE2uU1hqtDaKeqrdwj`
- Request headers: `User-Agent: PixivAndroidApp/<ver> (Android 11; <device>)`,
  `App-OS: android`, `App-OS-Version`, `App-Version`, `Accept-Language`.
- Verifier charset (RFC 7636 unreserved): `A-Z a-z 0-9 - . _ ~`, secure
  random; pixiv_dart_api uses length 128, MnyaCat 32. We use 64.
- Challenge: `base64url(sha256(ascii(verifier)))` without padding
  (pixiv_dart_api uses `base64Url`; we follow RFC 7636 base64url).
- Token JSON: `access_token`, `refresh_token`, `user.id`, `user.name`,
  `user.mail_address`, `user.profile_image_urls.main`, ...

## Deltas vs beta56

- beta56 routed the token request through a fixed IP
  (`targetIPGetter: 210.140.92.183`) with a spoofed `Host` header. Replica
  normal path uses real DNS + strict TLS per parent PRD R5; the fixed IP is
  legacy/emergency only.
- beta56 injected `cheat.js` into the WebView to read the password form.
  Forbidden by this task's PRD R4 — not implemented.
- beta56 captured the web session cookie natively at redirect time. The
  current callback only carries `code`; app-api requests need only the Bearer
  token. `Credential.cookie` stays null until a later task needs web cookies.

## Device verification status

- Automated tests cover PKCE vectors, session lifecycle, callback whitelist,
  exchange against a local strict-TLS HTTP server, and widget navigation.
- A real Android WebView login with a real Pixiv account has NOT been
  performed in this environment (no device/account available) and remains an
  explicit open acceptance item.
