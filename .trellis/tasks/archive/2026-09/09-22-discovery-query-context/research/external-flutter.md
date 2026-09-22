# External — Flutter framework: PageStorage vs restoration

Sources: api.flutter.dev (`PageStorage`, `RestorationMixin`, `restorationId`), docs.flutter.dev/platform-integration/android/restore-state-android. Verified 2026-09-22.

## PageStorage — in-memory, per-session

- "Establish a subtree in which widgets can opt into persisting states after being destroyed … when a page is switched out, its widget is destroyed and its state is lost. By adding a `PageStorage` at the root and adding a `PageStorageKey` to each page, some of the page's state (e.g. the scroll position of a Scrollable) will be stored automatically in its closest ancestor PageStorage, and restored when it's switched back." (PageStorage class docs)
- "Usually you don't need to explicitly use a PageStorage, since it's already included in routes." — each `ModalRoute` carries a bucket.
- `Scrollable` writes through `ScrollController.keepScrollOffset` (default true); multiple scrollables under one bucket need unique `PageStorageKey`s.
- Lifetime: the app session / owning route. Does NOT survive process death.

## RestorationScope / restorationId / RestorationMixin — process-death tier

- Setting `restorationScopeId` on `MaterialApp.router` injects `RootRestorationScope` and enables state restoration; each nested scope creates a `RestorationBucket`; `restorationId` stores into the surrounding bucket (docs: "Restore state on Android").
- Built-in widgets (`ScrollView`, `TextField`, `TabBar` via `RestorableScrollOffset`/`RestorableTextEditingController` etc.) self-restore when given a `restorationId`.
- Custom `State` needs `RestorationMixin` + `RestorableProperty` members + `registerForRestoration` in `restoreState`; `restorationId` must be unique within the scope (debug assert).
- The framework serializes buckets as state changes so data is ready when the OS kills the app; **bucket payloads must stay small** (platform serialization limits; Android Binder transaction bounds) — restore identifiers and small scalars, not entity lists.
- Restoration is best-effort: iOS doesn't guarantee it; data can be absent → always provide defaults.

## Decision rule used in this codebase's spec

`.trellis/spec/frontend/component-guidelines.md` "Route Restoration Contract" (:138-148):
- `MaterialApp.router`/`GoRouter`/shell/branches keep stable `restorationScopeId`s; feed scrollables keep stable `PageStorageKey` + `restorationId` — "the key preserves in-process tab/mode switching and the restoration ID covers the Flutter restoration bucket".
- "Ranking mode, search input/filter, and viewer page are durable path/query values and are updated through the route facade. An entity or input passed as route `extra` accelerates the first frame but is not the restoration source."
- `.trellis/spec/frontend/state-management.md` :45-46 — same rule.

## Layered model to apply in W2

| Layer | Mechanism | Survives | W2 use |
|---|---|---|---|
| In-memory | PageStorage + provider families | tab/mode switches, page rebuilds | scroll, feed state, drafts on pushed routes |
| Flutter restoration | `restorationId` / `RestorationMixin` | OS recreation (best-effort, small data) | scroll offsets; optionally selector index |
| Route durable | typed `path`/`query` params | back stack, deep link, process death | ranking `mode`, search `q/type/target/sort/…`, proposed `new` scope/type |
| Durable store | `SettingsRepository` etc. | restarts, account lifetime | search filter *defaults*, reverse-image engine |
