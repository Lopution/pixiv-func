# Risks — W2 discovery-query-context

## Route vs provider divergence
- `?mode=` on ranking is written via `context.replace` on every tab change (`ranking_page.dart:77` → routes.dart:1249-1255): heavy swipe sessions generate many route replaces; verify go_router doesn't leak entries or reset scroll on replace of the *current* location (it replaces, not pushes — but confirm restoration buckets survive).
- `/new` and `/recommended` gaining query params changes branch-root identity: a `context.replace` rebuild must not recreate the `TabController` mid-gesture — seed via `initialIndex` + guarded sync, not re-`initState`.
- **Search filter serialization gap** (routes.dart:295-347 vs `search_models.dart:120-160`): completing params changes URLs — old bookmarks/deep links must decode leniently (`firstWhere orElse` already does); new params must round-trip into `cacheKey` or PageStorage identity forks between pre/post-restore pages.

## Stale provider-family instances
- Families keyed by `NewFeedKey`/`SearchQuery`/`RankingMode` keep instances alive while watched; `PagedFeedController` already re-watches account+settings (`paged_feed_controller.dart:309-336`) — switching selector in route vs provider key must stay 1:1 or two identities fight (route says A, widget state says B).
- `trendingTagsProvider` is intentionally non-autoDispose (`search_trending_controller.dart:32-52`) — a QueryContext claim of "session scope" is correct, but document it.

## Filter/query changes invalidating pagination
- `SearchQuery.cacheKey` includes all 13 fields (`search_models.dart:168-181,465,503`): after completing route serialization, a restored URL yields the same cacheKey — good; but during transition, in-flight pushes that produced partial URLs will decode to *different* cacheKeys → duplicate provider entries for the "same" logical query. Acceptable (self-healing), note in PR.

## Restoration size limits
- Restoration buckets must stay small (see external-flutter.md): do NOT serialize result lists, image bytes, or WebView state; only scalars/ids. `q` strings are unbounded — trim/limit on write.

## Scroll-controller ownership
- `SmoothWheelScroll` desktop path owns a private controller (`smooth_wheel_scroll.dart:160-177`); `PrimaryScrollController.of` re-tap would silently no-op there → explicit per-key controllers are required (ranking/new already have them; recommended/search/search-home do not).
- Re-tap must target the **visible** scrollable only — Offstage-stacked bodies (new page `:149-161`) keep controllers alive for hidden keys; guard with `hasClients` + mounted checks.

## Gesture conflicts
- `RootSwipeSwitcher` hands edge drags to `BranchSlidePager` (`root_swipe_switcher.dart:330-350`); adding re-tap to `selectIndex` must not fire during `syncIndex`/drag settle (`_suppressGoBranch`, `branch_slide_stack.dart:156-172`) — emit only from the explicit same-index tap path.
- goBranch's stack pop + re-tap ordering: signal must arrive after the pop animation commits, else the revealed root scrolls before it's visible.

## Reverse-image lifecycle
- autoDispose family keyed on a page-local `Session` object (`reverse_image_controller.dart:128-133`) — any refactor that rebuilds `Session` identity resets the flow mid-search; keep `Session` in `initState`, never rebuild it in `build`.
- `webUpload` keeps the owned temp file until flow end (`:332-339`); a persistent header must not retain the file longer than the flow does.
- Engine persistence writes even when a switch fails (`_selectEngine` writes settings unconditionally, page `:126-131`) — check ordering if W1 changes cancel semantics.

## Ownership / overlap
- Astra row 4 (trending-tag discard) is assigned **W1 fixes discard, W2 converges hierarchy** — coordinate who edits `search_page.dart:153-155`; safest: W2 lands the keep-all presentation after W1's content fix merges.
- Row 21 cancel semantics is W1's; W2 adds the persistent header on top — sequence after W1 merge.
- W6 later consumes these pages for ranking variants — keep `IllustCard`/row untouched here.
- Shared-file conflicts with parallel leaves: `branch_slide_stack.dart`, `routes.dart`, `func_bottom_nav.dart`, `root_swipe_switcher.dart` are cross-cutting (W3's author page re-tap also needs the same signal) — land the re-tap channel once, document the contract, let W3 consume.
- l10n keys (new labels for filter summary / modify-query / persistent type selector) conflict across leaves — agree on key prefix early.
