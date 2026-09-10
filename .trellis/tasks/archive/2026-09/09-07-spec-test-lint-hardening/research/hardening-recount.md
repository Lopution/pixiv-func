# E0 hardening recount

Date: 2026-09-10

Code HEAD: `6d7ef46` (`task/09-07-interaction-visual-modernization`), fast-forwarded into
`task/09-07-spec-test-lint-hardening` before activation. The branch contains the
archived F implementation and its tests; no F commit was rewritten.

## Baseline commands and results

The counts below were obtained from the E worktree after the fast-forward:

```bash
find lib/core -mindepth 1 -maxdepth 1 -type d | wc -l
find test -type f -name '*_test.dart' | wc -l
rg -n '^\s*test(?:Widgets)?\(' test --glob '*_test.dart' | wc -l
rg -n '^\s*group\(' test --glob '*_test.dart' | wc -l
```

| Item | Result |
|---|---:|
| Direct `lib/core/<domain>/` directories | 23 |
| Test files | 79 |
| Test case declarations | 713 |
| `group` declarations | 66 |

The dependency cache was initialized with `flutter pub get --enforce-lockfile`.
`flutter analyze --no-pub` then reported `No issues found!` on Flutter 3.47.2.

## Spec surface

`rg -n '\(To be filled by (the )?team\)' .trellis/spec/frontend .trellis/spec/backend`
found 38 placeholder lines. Five belong to `frontend/hook-guidelines.md`, which
remains intentionally unfilled because the app has no Flutter Hooks contract.
The remaining 33 lines are in eight active guideline files and are E1 work:

- `frontend/directory-structure.md`
- `frontend/quality-guidelines.md`
- `frontend/state-management.md`
- `frontend/type-safety.md`
- `backend/database-guidelines.md`
- `backend/directory-structure.md`
- `backend/error-handling.md`
- `backend/logging-guidelines.md`

The canonical release document is `backend/release-artifacts.md`. No spec file
uses the retired release-pipeline name; eight stale references remain in
the child/parent planning documents and are handled by E1.

## Core library discoverability

All 23 selected existing entry files from `design.md` lack an anonymous
`library;` directive at the file header. E1 will add one responsibility/owner
summary and a link to the relevant spec section to each entry, without adding a
barrel file.

## Test duplication and brittle assertions

```bash
rg -l 'class [A-Za-z0-9_]+ implements CredentialStore' test --glob '*.dart'
rg -l 'class [A-Za-z0-9_]+ implements AccountMetadataRepository' test --glob '*.dart'
rg -n 'InMemorySharedPreferencesAsync' test --glob '*.dart'
```

Both account-double searches return the same 22 test files. The specialized
failure, counting, transfer, and lifecycle doubles will stay local; E2 will
share only the equivalent credential/metadata implementations and their common
provider overrides. `InMemorySharedPreferencesAsync` has three matches, all in
`test/helpers/test_preferences.dart`; direct construction outside that helper
is already zero.

`rg -n 'runtimeType\.toString\(\)' test` returns two assertions:
`PersonAvatar` in `illust_detail_page_test.dart` and `ApiParseError` in
`related_illust_repository_test.dart`. `test/zz_diag_tabbar_geometry_test.dart`
still exists and is the one-off diagnostic to remove in E2. `.gitignore` already
contains `test/failures/`.

## Semantics and static contracts

There are currently zero `Semantics(`, `SemanticsTester`, or
`find.bySemanticsLabel` matches in `lib/` and `test/`. E2 adds the small shared
component coverage described by the design.

The following six test files covering the five requested static contracts are
present at the E0 baseline:

- `test/pixiv_image_variants_test.dart` — feed/avatar `memCacheWidth` and
  unrestricted viewer decode;
- `test/startup_gate_test.dart` — settings/account startup boundary;
- `test/history_persistence_test.dart` — `EXPLAIN QUERY PLAN` checks without
  `SCAN TABLE`;
- `test/download_sink_test.dart` and `test/download_manager_test.dart` — sink
  lifecycle and bounded chunk streaming;
- `test/architecture/layering_test.dart` — import direction and shared-widget
  naming rules.

F's final release artifact measurements remain the source of truth:

| ABI | Measured APK bytes | Default threshold |
|---|---:|---:|
| `arm64-v8a` | 27,981,038 | 28,981,038 |
| `armeabi-v7a` | 23,805,284 | 24,805,284 |

The arm64 hard cap remains 32,000,000 bytes. These values match
`.trellis/spec/backend/release-artifacts.md` and the workflow defaults.

## E0 alignment decisions

- `test/helpers/test_preferences.dart` is the existing preferences owner; no
  second preferences helper is planned.
- `backend/release-artifacts.md` is the only release spec path.
- The E design and implementation plan now use `6d7ef46`, 79 test files, and
  713 test cases as the post-F baseline.
