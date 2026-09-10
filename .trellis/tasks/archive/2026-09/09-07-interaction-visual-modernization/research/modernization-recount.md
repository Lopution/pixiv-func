# F0 modernization recount

Date: 2026-09-09

Code HEAD: `973ca71` (`task/09-07-interaction-visual-modernization`)

## Source baseline

The F0 recount at the implementation branch is unchanged from the planning snapshot:

| Item | Result |
|---|---:|
| Files importing `package:flutter/material.dart` or `package:flutter/cupertino.dart` | 84 |
| Files importing `package:flutter/cupertino.dart` | 4 |
| Direct `Navigator` push matches under `lib/features` and `lib/app` | 23 |
| `ReplicaPageRoute` references under `lib` | 27 |
| `PageStorageKey` references under `lib` | 7 |
| Golden image files | 1 (`test/goldens/home_bar.png`) |

The resolved versions before F1 are `go_router 17.5.0` and
`cached_network_image 3.4.1`.

## Candidate dependency resolution

`flutter pub outdated` and the dry-run command below both resolved the planned
candidate set on Flutter 3.47.2:

```bash
flutter pub add --dry-run \
  go_router@^18.0.1 cached_network_image@^4.0.0 \
  material_ui@^1.2.0 cupertino_ui@^1.0.2
```

Candidate versions:

| Package | Candidate |
|---|---:|
| `material_ui` | 1.2.0 |
| `cupertino_ui` | 1.0.2 |
| `go_router` | 18.0.1 |
| `cached_network_image` | 4.0.0 |

The dry run reported six dependency changes including the four direct
candidate packages and their compatible image/router transitive packages. It
did not modify `pubspec.yaml` or `pubspec.lock`.

The migration is a Dart analyzer fix, not a standalone executable. The correct
dry run is:

```bash
dart fix --dry-run --code=migrate_design_widgets
```

It currently proposes 114 `migrate_design_widgets` fixes in 109 files. The
dry-run also reports one `missing_dependency` fix for `pubspec.yaml`; it does
not modify the repository. F1 can apply the migration after the candidate
dependencies are made direct and then review the generated import changes.

## Legacy dependency scan

The resolved package source scan still finds SDK Material imports in these UI
dependencies. Their app entry points remain behind the single app-level bridge
until those packages publish the extracted-package imports:

| Resolved package | Current app surface |
|---|---|
| `easy_refresh 3.5.1` | `lib/app/pull_to_refresh.dart` and feed refresh surfaces |
| `flutter_staggered_grid_view 0.7.0` | `lib/app/widgets/feed/feed_grid.dart` and feed pages |
| `octo_image 2.1.0` (via image loading) | `lib/app/pixiv_image.dart` and image/card surfaces |
| `webview_flutter 4.14.1` | `lib/features/login/login_webview_page.dart` |

Other matches belong to Flutter SDK/localization internals or package
platform/test implementations rather than app-owned widget trees. The bridge
is therefore kept at `MaterialApp.builder`; no per-feature compatibility
wrapper is added.

## APK baseline attempt

The planned pre-migration command was run:

```bash
flutter build apk --release --flavor fdroid \
  --split-per-abi --target-platform android-arm64,android-arm \
  --obfuscate --split-debug-info=build/symbols/fdroid
```

The build reached Flutter/Gradle compilation but failed while `sqlite3 3.5.2`
tried to download its Android arm64 native asset from GitHub:

```text
Connection closed before full header was received
https://github.com/simolus3/sqlite3.dart/releases/download/sqlite3-3.5.2/libsqlite3.arm64.android.so
```

No new split APK was produced, so this branch does not claim a fresh size
measurement. The last verified B reference remains:

| ABI | Measured APK bytes | Current default threshold |
|---|---:|---:|
| `arm64-v8a` | 27,435,321 | 28,435,321 |
| `armeabi-v7a` | 23,226,795 | 24,226,795 |

Those values remain reference data only until the build can be repeated after
the GitHub native-asset download is available. The arm64 hard cap remains
32,000,000 bytes.

## Network evidence for the blocked build

The Dart/Flutter process had no `HTTP_PROXY`/`HTTPS_PROXY` environment
variable. Git has a separate stale proxy configuration at
`http://127.0.0.1:7897`, but no process was listening on that port. A direct
`curl` probe to the native-asset URL timed out after 20 seconds. This is an
environment download blocker; it is not a dependency-resolution or F1 code
failure.
