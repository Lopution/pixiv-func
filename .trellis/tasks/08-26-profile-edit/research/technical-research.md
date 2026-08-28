# Profile edit capability research

Date: 2026-08-28 (Asia/Shanghai)

## Reference source

The task metadata names `c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989` as the
beta56 reference. It is not present in the local Git object database
(`git cat-file -t` returned no object), is not reachable from the local refs,
and the repository's GitHub API returned `422 No commit found`. Therefore the
field contract is limited to the fields already represented by the current
`UserEntity`; no unverified beta56 field was added.

## Official route checks

These checks used fresh requests without the user's account cookies or tokens:

| Request | Observation | Decision |
| --- | --- | --- |
| `GET https://www.pixiv.net/settings/profile` | HTTP 302 to an HTTP login return URL | Not an approved authenticated app route; the app's exact-HTTPS WebView policy must reject the downgraded redirect |
| `GET https://www.pixiv.net/setting_user.php` | HTTP 302 to an HTTP login return URL | Legacy URL is not used |
| `GET https://app-api.pixiv.net/v1/user/detail` | HTTP 400 for a request without the required user parameter/auth context | Read-only user detail is live; this does not establish an update endpoint |

The current repository's `PixivUserRepository` exposes only detail, works,
bookmarks and relationship reads. No official profile update contract was
found in the source, and no password/cookie DOM helper is carried forward.

## Implementation decision

`PixivProfileEditRepository` uses the authenticated official read-only detail
request to populate the form and returns an explicit unavailable capability for
submit. `ProfileEditController` nevertheless implements the complete safe
boundary: capability and dirty-field checks, bounded text validation, owned
image cleanup, typed confirmed/pending/field-error outcomes, ephemeral current
password handling, and account/credential/network revision fencing. The UI
shows unsupported fields and does not offer a fake success path.

This is a product capability blocker for real profile mutation, not an account
login blocker. The current MuMu session has a real signed-in account and has
already loaded the Home/profile read path. No profile value was changed because
the task did not specify a harmless test value or authorize a live account
mutation; a future controlled test-account run can replace the unavailable
adapter only after an official API/Web contract is confirmed.

## Network boundary

The read path is the existing `PixivHttpClient` supplied by
`pixivHttpClientProvider`, which consumes the app-scoped `NetworkAccessPolicy`.
Image preparation is app-private and performs no network operation. There is
no third-party reverse proxy, fixed-IP route, Host/SNI rewrite, certificate
relaxation, or password/cookie extraction.
