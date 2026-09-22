# Codebase — Search (搜索首页 / 输入 / 结果 / 筛选 / 标签搜索)

Scope: `lib/features/search/search_page.dart` (569 lines, hosts `SearchHomePage` + `SearchInputPage`), `search_result_page.dart` (337), `search_filter_sheet.dart` (549), `tag_search_page.dart` (25), `search_text.dart` (8, label helper), `lib/core/search/*`, route wiring `lib/app/navigation/routes.dart`.

## SearchHomePage (搜索首页, `search_page.dart:22-172`)

- Scrollable: `CustomScrollView` `PageStorageKey('search-home')` (`:39`), `restorationId: 'search-home'` (`:40`), controller from `SmoothWheelScroll` (`:37-38`) — no explicit controller.
- Entries: `_SearchGuideBox` read-only `SearchBar` → `openSearchInput` (`:47`, `:179-198`); reverse-image `FilledButton` → `openReverseImageSearch` (`:55-62`); Spotlight `FilledButton.tonalIcon` (`:66-78`).
- Trending section: title + `SegmentedButton<SearchResultType>` for illust/novel (`:90-106`) bound to `trendingKindProvider` — an **in-memory** `NotifierProvider` defaulting to illust, rejecting `user` (`search_trending_controller.dart:11-23`). `trendingTagsProvider` is a deliberately non-autoDispose `FutureProvider` keyed on kind + account (`:32-52`).
- **Tag truncation confirmed**: grid `itemCount: tags.length >= 3 ? tags.length - (tags.length % 3) : tags.length` (`:153-155`) — silently drops the dangling partial row (1–2 tags). This is the "popular tags must not be discarded" target; fix is a presentation change (e.g. `Wrap`/chips or `SliverGrid` with `maxCrossAxisExtent`) not a fetch change. (Astra row 4 assigns the discard fix to W1, hierarchy to W2 — coordinate.)
- Tile tap → `openSearchResults` with `IllustSearchQuery`/`NovelSearchQuery` (`:222-227`); long-press opens representative work (`:228`, `:206-213`).
- Trending error → inline `FeedError(scrollable: false)` (`:120-128`); loading → sliver spinner; empty → text. Section header (incl. kind selector) stays visible in all states.

## SearchInputPage (`search_page.dart:292-515`, modal route `/search/input`)

- Route state: `q` and `type` query params (`routes.dart:1007-1023`); `initialKeyword`/`initialType` seed `TextEditingController` (`:327`) and `TabController` index (`:321-326`).
- Type tab changes → `widget.onTypeChanged` → `replaceSearchInput` → `context.replace` rewrites `q`/`type` in the URL (`:371-378`, routes.dart:1237-1247) — draft keyword + type ARE durable route state.
- Filters: button visible only for illust/novel (`:496-507`); `_editFilters` → `showSearchFilterSheet` → `settingsProvider.setSearchFilters` (`:405-413`) — the input page's filter set is the **persisted default** (`searchFiltersProvider`, `settings_controller.dart:309-315`), not per-query.
- Submit: `_submit` (`:380-388`) → `openSearchResults` (push `/search/results`); numeric-only keywords shortcut to the entity page (`routes.dart:1263-1274`).
- **Suggestion tap = fill + immediate submit**: `_selectSuggestion` (`:419-424`) writes the text field then calls `_submit()` — there is no "fill only" affordance (Astra row 5 / design §4.2 requirement "建议项区分填入与立即搜索" is unmet).
- Autocomplete: `searchAutocompleteProvider` — `autoDispose` notifier, 260 ms debounce, generation + CancelToken guards (`search_autocomplete_controller.dart:45-140`); panel states at `:517-568` (empty keyword hint / `FeedLoading` / `FeedError` / no-suggestions text / `ListView` — **no PageStorageKey/restorationId**, scroll resets; acceptable since it is a transient push route).
- Focus: post-frame + post-transition `requestFocus` (`:329-350`); back button is plain `Navigator.pop` labeled `searchCancel` (`:479-483`).
- The keyword draft survives process death via the `q` param, but the `TextEditingController` cursor/selection does not.

## SearchResultPage (`search_result_page.dart`)

- Route state: `/search/results` params → `_searchQuery` (`routes.dart:313-327`); `query` is the page's single identity (`:28`).
- AppBar title = `query.keyword` plain Text (`:66-71`) — **not editable**; only affordance is a filter `IconButton` when the query type supports filters (`:72-79`). No clear/edit-keyword action in the header (Astra row 5 partial).
- Filter edit: `_editFilters` (`:36-55`) → sheet → `replaceSearchResults(updated)` (`context.replace`, routes.dart:1282-1293) — query change = new route entry, keeping back-stack semantics sane.
- Feed: `searchFeedProvider(query)` family (`search_feed_controller.dart:194`; `feedKey = 'search:${query.cacheKey}'` `:21`); `cacheKey` covers type + trimmed keyword + all 13 filter fields (`search_models.dart:168-181`, `:447`, `:465`, `:503`).
- Scroll/restoration per query: `PageStorageKey(query.cacheKey)` + `restorationId: 'search-${query.cacheKey}'` in all three type feeds (`:160-163`, `:231-234`, `:288-291`).
- States: async loading/error → `FeedEmpty`/`FeedError` (`:81-89`); `showInitialError`/`showInitialSpinner` (`:91-105`); **empty result** → `FeedEmpty(searchNoResults)` whose only action is `onRefresh` of the same query (`:138-145`) — no "modify query" path (Astra row 5).
- `SmoothWheelScroll` with no explicit controller (`:157-164`) — same desktop `PrimaryScrollController` caveat.

## showSearchFilterSheet (`search_filter_sheet.dart`)

- Modal bottom sheet (`showAppBottomSheet`, `:10-20`) — route `extra`-free, returns `SearchFilters?` via `Navigator.pop`.
- Local draft `_filters = widget.initial` (`:37`) + 6 `TextEditingController`s for numeric bounds (`:39-56`); date pickers clear the duration preset (`:96-102`); `_invalidRange` gates the Apply button (`:74-78`, `:389-394`); Reset clears everything (`:135-143`).
- Commit semantics differ by caller: input page persists to settings (`search_page.dart:412`); result page replaces the route query (`search_result_page.dart:44-54`). Two destinations for the same control — a QueryContext doc must spell this out.
- Premium-only `popularDesc` flagged inline (`:163-165`); illust-only groups hidden for novel/user (`:27-30`).

## TagSearchPage (`tag_search_page.dart`)

- 25-line compatibility wrapper: builds `SearchResultPage` with `IllustSearchQuery(keyword, filters: default)` (`:14-23`); route `tag/:keyword` is a common branch route pushed on the *current* stack (routes.dart:604-615, `:1413-1418`) so it opens on top of whatever page the tag was tapped from. Fully inherits the result-page context (AppBar shows keyword, filter button present).
- The keyword is a durable path parameter; no additional `restorationId` beyond the page key.

## Route serialization gap (critical)

- `_searchFilters` parses only `target|sort|duration|start|end` (routes.dart:295-311); `_searchQueryParameters` writes only those five (`:335-346`).
- `SearchFilters` has 13 fields — `aiFilter`, `bookmarkMin/Max`, `ratio`, `contentType`, `widthMin/Max`, `heightMin/Max` are **dropped from the URL**. A restored/deep-linked `/search/results` silently loses them, and the decoded `cacheKey` differs from the original → PageStorage + provider-family identity also diverge after process death. Any QueryContext design must either complete the serialization or accept/declare the loss.
