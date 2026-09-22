# Codebase — Recommended (推荐页)

Scope: `lib/features/home/recommended/recommended_home_page.dart` (430 lines), `recommended_illust_page.dart` (137 lines, standalone dead page — no references outside itself; verified by grep), `lib/core/illust/recommended_feed_controller.dart`, `lib/core/illust/recommended_repository.dart`.

## Selector / tab switching

- `RecommendedContentType` enum: `{illust, manga, novel, user}` — `recommended_repository.dart:17`.
- `_RecommendedHomePageState` owns view-local state: `_type` (`recommended_home_page.dart:47`), `_loaded` set (`:48`), `_entrancePlayed` per-type set (`:49`), and a `TabController` of length 4 created at `:54-55`.
- Tab/swipe changes flow through `_onTabChanged` (`:66-72`) → `setState` updates `_type` + adds to `_loaded`; bar taps go through `TabBar.onTap` → `_selectType` (`:74-80`, wired at `:154`).
- The selector is a `TabBar` inside the AppBar title (`:94-100`, `:146-168`): `isScrollable: false`, each label wrapped in `FittedBox(fit: BoxFit.scaleDown)` at `:158-164` — the Astra-confirmed unbounded-shrink issue (traceability row 1).
- `RootSwipeSwitcher` wraps the body (`:102-110`) with `onPrepareAdjacent` pre-warming neighbors; `TabSlideStack` (`:111-124`) mounts only loaded types, keyed `ValueKey(type)` (`:117`).

## Query / filter state and ownership

- The type is the only selector; it lives only in widget `State` (`_type`, `:47`). **Not in the route** — `/recommended` has no query params (routes.dart:963-972).
- Feed data: `recommendedFeedProvider` — `AsyncNotifierProvider.family<PagedFeedState, (record with type)>` (`recommended_feed_controller.dart:126`). Each type has an independent provider-family instance → independent cursor/error/refresh lifecycle. Account + filter-boundary invalidation is inherited from `PagedFeedController.build` (`paged_feed_controller.dart:309-336`: watches `accountStoreProvider` id and the R18/AI/muted settings tuple).

## Scroll retention and restoration

- Each type body is a `CustomScrollView` with `PageStorageKey('recommended-${type.name}')` (`:305`) and `restorationId: 'recommended-${type.name}'` (`:308`) — in-process tab switching keeps scroll via PageStorage; process-death covers scroll offset via Flutter restoration.
- **No explicit `ScrollController`** — `SmoothWheelScroll` (`:302`) passes `widget.controller ?? PrimaryScrollController.maybeOf(context) ?? _controller` on touch platforms (`smooth_wheel_scroll.dart:150-157`), but always its private `_controller` on desktop (`:169-174`). Re-tap-to-top cannot rely on `PrimaryScrollController.of` for these feeds on desktop.
- `_type` itself is **not** restored: no `RestorationMixin`, no route param — process death returns to `illust`.

## Loading / error / empty

- `_RecommendedFeedView` (`:181-241`): `loading` → `FeedLoading`; `error` → `FeedError` + `retryInitial`; `feed.showInitialError` → `FeedError`; `showInitialSpinner` → `FeedLoading`; `isEmptyAndReady` → `FeedEmpty` + refresh (`:205-224`).
- Refresh error renders as an extra `FeedTail` sliver above the normal tail (`:264-282`) — error appears at list tail, not near the tab bar (Astra row 1: "刷新错误仍进入列表尾部结构").

## Re-tap behavior

- None. `TabBar.onTap` → `_selectType` early-returns on same type (`:74-75`); bottom-nav same-branch tap calls `goBranch` which pops the branch stack to root (see codebase-navigation.md). No scroll-to-top exists.

## Context visibility

- The tab bar stays mounted in all feed states (it is in the AppBar, not the body) — good precedent for the QueryContext contract.
