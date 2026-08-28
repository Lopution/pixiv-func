# Clipboard account-transfer threat model

## Scope and guarantees

The transfer envelope is a short-lived, versioned transport for the existing
copy/paste UX. It is not an encrypted or authenticated message. The payload
contains a Pixiv credential, so the clipboard must be treated as sensitive
plaintext while it exists.

| Boundary | Implemented control | Actual guarantee |
| --- | --- | --- |
| At rest on the app | `CredentialStore` and replay state use secure storage | Credentials and replay digests are not put in `SharedPreferences` or task artifacts |
| Accidental clipboard damage | Canonical JSON, strict base64/size/schema checks, unkeyed SHA-256 checksum | Detects accidental edits, truncation and malformed transport |
| Reuse on one target | Random envelope nonce; target stores only a SHA-256 nonce digest and expiry | Rejects a previously claimed nonce on that target; it is not a cross-device revocation service |
| Clipboard reader | Android sensitive clip description, five-minute expiry and conditional clear | Reduces accidental exposure and stale lifetime; it cannot hide plaintext from an app that can read the clipboard |
| Clipboard writer | Strict parser and server verification of the imported credential | A writer can still replace fields and recompute an unkeyed checksum; only Pixiv authentication decides which account is valid |

Base64 is encoding, not confidentiality. The checksum has no secret key, so a
malicious writer can recompute it. The nonce digest is also not an
authenticator. These controls must therefore be described as corruption and
target-local replay detection only; no UI or log may call the envelope
`encrypted`, `authenticated`, `E2E`, or `secure transfer` in that stronger
sense.

## Import trust order

1. The user explicitly requests a clipboard read from Login help.
2. The parser bounds and validates the canonical envelope, then verifies its
   checksum before returning expiry or clock-skew outcomes.
3. The target claims the nonce digest. Replayed envelopes stop before network
   verification and are cleared only when the clipboard still matches the
   original fingerprint.
4. The imported access token is sent once to the exact App API
   `/v1/user/detail` destination. If Pixiv reports an expired access token,
   the supplied refresh token may be exchanged once through the existing exact
   OAuth endpoint; no request body is replayed across routes.
5. Only the server response supplies account id, name and image metadata.
   Credential and metadata are committed through the existing account-store
   atomic boundary; failure must leave no new half-account.

## Android lifecycle

On Android API 33 and above the native channel sets
`ClipDescription.EXTRA_IS_SENSITIVE`. The channel schedules a five-minute
clear and compares the current clipboard fingerprint before clearing. A later
user copy is consequently preserved. Explicit import success or recognized
import failure uses the same conditional clear. Lower Android API levels retain
the bounded expiry and conditional clear but cannot receive the sensitive clip
metadata flag.

## Residual risk and future change

An evil clipboard reader can obtain the credential during the expiry window,
and an evil writer can create a different checksum-valid envelope. Password,
pairing, QR or public-key transfer would require a new version and an explicit
UX/security approval; putting a key beside ciphertext in this same clipboard
payload would not solve either threat.

Automated tests cover malformed, oversized, truncated, unknown-version,
expiry, clock-skew, accidental-tamper, checksum-recalculation, target-local
replay, conditional-clear and atomic-write boundaries. Device evidence is
reported separately: the verified target is a MuMu emulator on API 35, not a
physical device; API 36 and a second independent device remain unverified.
