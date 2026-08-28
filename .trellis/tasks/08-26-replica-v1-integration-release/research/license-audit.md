# Replica v1 license and provenance audit

Audit date: 2026-08-29 (Asia/Shanghai). This is a repository and dependency
evidence record, not a substitute for legal review of a distribution.

## Project provenance

- `LICENSE` is the complete GNU Affero General Public License, version 3,
  matching the original Pixiv Func reference license.
- `README.md` declares `SPDX-License-Identifier: AGPL-3.0-only` and identifies
  the project as a modernized replica. `NOTICE` attributes git-xiaocao (小草)
  and pins the beta56 behavior reference to
  `svenfuss/pixiv_func_mobile@c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`.
- The five fixed-commit audit samples and their GPL/MIT licenses are recorded
  in `08-27-open-source-pixiv-app-plan-audit/research/source-selection.md`.
  Their code, assets, secrets, client identity, fixed IPs and TLS-bypass
  techniques were used as research references only; no third-party source was
  copied into this implementation.

## Locked Dart dependencies

`pubspec.lock` contains 133 hosted packages. A local package-cache audit found
a `LICENSE*` file for all 133 locked hosted packages, including transitive
packages. The direct runtime dependencies are:

| Package family | Locked versions | Observed license evidence |
|---|---|---|
| `meta`, `crypto`, `http`, `path`, `visibility_detector` | `1.19.0`, `3.0.7`, `1.6.0`, `1.9.1`, `0.4.0+2` | Dart project-author BSD-style license files |
| `cupertino_icons`, `flutter_riverpod`, `cached_network_image`, `flutter_cache_manager`, `flutter_staggered_grid_view` | `1.0.9`, `3.4.2`, `3.4.1`, `3.4.2`, `0.7.0` | MIT license files |
| `go_router`, `shared_preferences`, `webview_flutter`, `path_provider` | `17.5.0`, `2.5.5`, `4.14.1`, `2.1.6` | Flutter Authors BSD-style license files |
| `flutter_secure_storage` | `10.3.1` | BSD 3-Clause license file |
| `sqflite`, `sqflite_common_ffi` | `2.4.3`, `2.4.2+1` | BSD 2-Clause license files |
| `archive`, `image` | `3.6.1`, `4.3.0` | MIT `LICENSE` plus `LICENSE-other.md` files |

The SDK packages (`flutter`, `flutter_localizations`, `flutter_test`) are
provided by the pinned Flutter SDK rather than the hosted cache. Dev-only
packages were included in the 133-package audit. A release process still needs
to publish the complete third-party license texts alongside the distributed
artifact; this audit does not silently replace that obligation with package
names.

## Bundled assets

- `assets/icon.ttf` is byte-identical to the fixed beta56 reference asset at
  the pinned commit. Current SHA-256:
  `b1277d76c133e157bd819b55477e9400880c84474d9cddffe25ecce3e7199c05`.
- The 38 emoji PNGs and 40 stamp JPGs are byte-identical to the corresponding
  directories in the same fixed reference checkout. A reproducible aggregate
  over the sorted per-file content hashes is
  `118183fa6fdfadd125c4ece564bdacd91cfb24407a0c712e26c2f90bffd470c8`.
- The older archived comments evidence records a different aggregate value;
  because that value cannot be reproduced from the current/reference bytes, it
  is not used as this acceptance record's hash. The byte-for-byte comparison
  and pinned source commit are the authoritative evidence here.

No new remote asset, private key, account credential, cookie, or updater
signing material is stored in the repository.

## Release boundary

The current Android release variants build with the local debug signing
configuration in `android/app/build.gradle.kts`; no production keystore is in
the repository. Therefore the APKs are build artifacts, not distributable
production-signed releases. A production release must inject an external
signing configuration and matching updater signer/public-key material before
the release gate can be closed.
