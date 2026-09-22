# Codebase — Navigation shell, routes, scroll infrastructure

Scope: `lib/app/navigation/routes.dart`, `lib/app/widgets/branch_slide_stack.dart`, `lib/app/widgets/func_bottom_nav.dart`, `lib/app/widgets/root_swipe_switcher.dart`, `lib/app/widgets/smooth_wheel_scroll.dart`.

## Router / restoration skeleton

- `GoRouter` has `restorationScopeId: 'router'` (`routes.dart:824`); the shell `StatefulNavigationShell` uses `restorationScopeId: 'home-shell'` (`:943`).
- Branch restoration ids: `recommended` (`:971`), `ranking` (`:985`), `new` (`:995`), `search` (`:1005`), `settings` (`:1043`).
- `_page`/`_modalPage` set `restorationId: state.pageKey.value` when inside a `RestorationScope` (`:169-171`, `:201-203`) — every pushed route gets a Flutter restoration bucket keyed by its unique page key.

## Branch switching and the current same-tap semantics

- `FuncShellBottomNav` renders M3 `NavigationBar`; a slot tap calls `onSelected(i)` (`func_bottom_nav.dart:378`), which is `_pager.selectIndex` (`branch_slide_stack.dart:467-474`).
- `BranchSlidePager.selectIndex` (`branch_slide_stack.dart:137-151`):
  - same index → `shell.goBranch(index)` and return. GoRouter's `goBranch` default pops the branch's pushed stack to its root — **same-tap today means "return to branch root", never scroll-to-top**.
  - different index → `tab.animateTo(index)` (strip slides; `_onTabChanged` at `:61-68` forwards real index changes to `goBranch` unless `_suppressGoBranch` from `syncIndex` `:156-172`).
- `syncIndex` exists for external index moves (deep link, restoration, rail tap) — slides without re-entering `goBranch` so a pushed route is not dropped (`:153-155`).
- Rail variant shares the same `onSelected` path (`func_bottom_nav.dart:751`).

## In-page horizontal gestures

- `RootSwipeSwitcher` (`root_swipe_switcher.dart:17-46`): optional `tabController` + `onPrepareAdjacent`; committed drags (`distanceFraction` 0.25, `:42`; `minFlingVelocity` 400, `:37`) move the page's TabController; when the drag runs off the tab strip's edge (or there is no strip) `_releaseBranch` forwards to `BranchSlidePager.endDrag` (`:330-339`), else `_stepBranch` → `shell.goBranch(next)` (`:344-350`). Ranking/Recommended/New wire their tab controllers through it; SearchHomePage passes none.

## Scroll controller ownership (re-tap-to-top relevant)

- `SmoothWheelScroll` (`smooth_wheel_scroll.dart:143-188`):
  - non-desktop: `widget.controller ?? PrimaryScrollController.maybeOf(context) ?? _controller` (`:150-157`) — touch feeds with no explicit controller hang off the ambient `PrimaryScrollController`.
  - desktop: always uses its private `_controller` (`:160-177`) for wheel smoothing — `PrimaryScrollController` lookups inside the subtree still see the ambient one, which is **not** attached to the scrollable. Re-tap-to-top via `PrimaryScrollController.of` would silently no-op on desktop for feeds without explicit controllers.
- Explicit controllers exist today in: `RankingPage._scrollControllers` per mode (`ranking_page.dart:43,80-82`), `NovelRankingPage` (`novel_ranking_page.dart:33,64-66`), `_NewFeedBody` (`new_page.dart:230,235,299`). Recommended feeds, SearchHomePage and all search-result feeds have **none**.
- PageStorage keys: `recommended-<type>`, `ranking-<mode>`, `novel-ranking-<mode>`, `new-<scope>-<type>`, `search-home`, `search-<query.cacheKey>` — see per-page notes.
- No `RestorationMixin`/`RestorableProperty` usage anywhere in `lib/features/` (verified by grep) — all restoration today is `restorationId` on scrollables + route query params.

## Search route encoding (see codebase-search.md for the field-level gap)

- Parse: `_searchType`/`_searchEnum`/`_searchDate`/`_searchFilters`/`_searchQuery` (`routes.dart:277-327`); serialize `_searchQueryParameters` (`:329-347`).
- Facades: `openSearchInput` push (`:1225-1235`), `replaceSearchInput` replace (`:1237-1247`), `replaceRankingMode` replace (`:1249-1255`), `openSearchResults` push (`:1257-1280`), `replaceSearchResults` replace (`:1282-1293`), `openReverseImageSearch` root push with `extra` (`:1301-1306`), `openNovelRanking` branch push (`:1189-1190`), `openTagSearch` (`:1413+`).
- Ranking branch `homeBuilder` reads `?mode` → `RankingPage(initialMode:, onModeChanged:)` (`:977-980`); novel-ranking route passes `const NovelRankingPage()` — **no mode param** (`:549-553`).

## Feed state primitives

- `PagedFeedController.build` (`paged_feed_controller.dart:309-360`) watches the account boundary (`accountStoreProvider` current id) and the filter-settings tuple, so every family member re-builds on account change — no stale-account feeds.
- Shared states `FeedLoading`/`FeedEmpty`/`FeedError`/`FeedTail` (`feed_states.dart:79,106,164,8`); `FeedError.scrollable=false` exists for non-scrollable embedding (`:200-209`).
