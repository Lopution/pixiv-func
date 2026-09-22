# Codebase — New (新作页)

Scope: `lib/features/new/new_page.dart` (392 lines), `lib/core/new/new_feed_models.dart`, `lib/core/new/new_feed_controller.dart`.

## Selector / tab switching

- Two-dimensional selector: `NewFeedScope {following, everyone, myPixiv}` (`new_feed_models.dart:5`) × `NewFeedType {illust, novel}` (`:8`); combined key `NewFeedKey` (`:11`).
- View-local state in `_NewPageState`: `_selectedIndex` (`new_page.dart:37`), `_type` (`:38`), `_selectorExpanded` (`:39`), `_loadedKeys` set (`:34-36`, seeded with `following+illust`), `TabController` length 3 (`:46`).
- Scope tabs: `TabBar` in AppBar (`:94-116`), `isScrollable: false`, `FittedBox(scaleDown)` labels (`:106-112`) — same unbounded-shrink issue as Recommended.
- **Re-tap currently expands, not scrolls**: `TabBar.onTap: _onTabTap` (`:102`) → `:70-74` — same-index tap toggles `_selectorExpanded`, exposing `_NewTypeSelector` (an inline `ChoiceChip` row `:179-216`). This is exactly the behavior W2 must replace with scroll-to-top (design §4.2, §5.1; implement §6 W2 gate "重复点击当前分类/标签固定为回到顶部，不再承载展开入口").
- Type selector is therefore **not persistently visible** — hidden until re-tap; collapses on scope change (`:63-67`) and after selection (`:79`). Astra row 3: "当前'范围＋类型'并非持续可见".
- `RootSwipeSwitcher` (`:121-131`) + `TabSlideStack` (`:142-165`); each scope slot holds an `Offstage` stack of `_NewFeedBody` keyed `ValueKey(key)` (`:149-161`) — visited (scope,type) bodies stay alive.

## Provider ownership

- `newFeedProvider` — `AsyncNotifierProvider.family<PagedFeedState, NewFeedKey>` (`new_feed_controller.dart:76-81`); `feedKey` string `'new:<scope>:<type>'` (`:23`). Watched at `new_page.dart:246`.
- Selector state (`_selectedIndex`, `_type`, `_selectorExpanded`) is page-local `State` only — **no route params** on `/new` (routes.dart:987-996), no restoration registration.

## Scroll retention and restoration

- Each `_NewFeedBody` owns an explicit `ScrollController` (`:230`, `:235`, passed to `SmoothWheelScroll` at `:299`) — per-key controller, good hook for re-tap-to-top.
- `PageStorageKey('new-${scope.name}-${type.name}')` (`:302-304`) + `restorationId` (`:308-309`) — scroll survives widget rebuilds and process death.
- Selected scope/type do NOT restore (in-memory only).

## Loading / error / empty

- `_NewFeedBody.build` (`:245-317`): loading → `FeedEmpty(icon, newLoading)`; async error → `FeedError` + `invalidate`; `showInitialError` → `FeedError` + `retryInitial`; `showInitialSpinner` → `FeedEmpty`; `isEmptyAndReady` → `FeedEmpty` + refresh (`:275-283`); load-more error → `FeedTail`.
- Scope tabs stay visible in all states (AppBar); the **type** selector is absent unless expanded — the current context is partially invisible.

## Gaps vs W2 contract

- Re-tap = expand selector → must become pure scroll-to-top; type must gain a persistent visible home (chips row or equivalent).
- Scope/type are not durable; a `?scope=&type=` route pair (like ranking's `?mode=`) is the QueryContext-compatible fix.
