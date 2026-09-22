# External — Pixiv Shaft (Android, github.com/CeuiLiSA/Pixiv-Shaft)

Source: Shaft `master` via jsDelivr on 2026-09-22 (`/tmp/shaft_*.java|kt`, tree listing `/tmp/shaft_tree.json`). Repo paths under `app/src/main/java/`.

## Shared query model across input and results (old UI, `ceui.lisa.*`)

`SearchActivity` (`activities/SearchActivity.java`):

```java
// :61-66 — one activity-scoped ViewModel is THE query owner
searchModel = new ViewModelProvider(this).get(SearchModel.class);
searchModel.getKeyword().setValue(keyWord);
searchModel.getIsNovel().setValue(index == 1);
searchModel.getIsPremium().setValue(isPremium);
```

- `SearchModel` (`viewmodel/SearchModel.java:5-77`) holds `keyword`, `starSize` (bookmark count), `searchType`, `sortType`, `lastSortType`, `startDate`, `endDate`, `nowGo`, `isNovel`, `isPremium`, `r18Restriction` — all `MutableLiveData`, **survives rotation** (ViewModel), not process death.
- The three result fragments (`FragmentSearchIllust/Novel/User`) and the filter sheet (`FragmentFilter`) all resolve the **same** activity-scoped model: `FragmentFilter.java:36` `new ViewModelProvider(requireActivity()).get(SearchModel.class)`, then `.observe` start/end date (`:43-49`) and `setValue` back on every control (`:80,:102,:128-129,:143-145,:155,:174`).
- Submit is a broadcast trigger: `searchModel.getNowGo().setValue("search_now")` (`FragmentFilter.java:226`; `SearchActivity.java:256`) — every result fragment re-queries with the current model values.
- The activity AppBar search box writes `searchModel.getKeyword().setValue(...)` (`SearchActivity.java:205`) — the header is editable and is the single source for all three result tabs. This is the "shared editable query header" reference for W2's search contract.

## Filter persistence (`utils/Settings.java`, `fragments/FragmentFilter.java`)

`Settings` is the JSON-serialized SharedPreferences aggregate behind `Shaft.sSettings`:

```java
// Settings.java:122,128
private String searchDefaultSortType = ""; // 搜索结果默认排序方式
private String searchFilter = "";          // last star-size filter value
```

`FragmentFilter` restores both into the sheet:

```java
// :88-91 — star-size spinner preselects the persisted value
if (PixivSearchParamUtil.ALL_SIZE_VALUE[i].equals(Shaft.sSettings.getSearchFilter())) { ... }
// :111 — sort spinner defaults to the persisted default sort
baseBind.sortTypeSpinner.setSelection(
    PixivSearchParamUtil.getSortTypeIndex(Shaft.sSettings.getSearchDefaultSortType()));
```

So Shaft persists a **default sort** + **last star-size filter** durably, while the whole working query (keyword, dates, target, sort) lives in the activity ViewModel. Two durability tiers — durable defaults vs per-session working copy — matching pixiv-func's `searchFiltersProvider` (durable defaults) vs route query (per-page working copy).

## New UI (`ceui.pixiv.ui.search.*`)

- `SearchViewModel` (`search/SearchViewModel.kt`): `tagList` (multi-tag query, joined by spaces at `:120`), `inputDraft`, `illustSelectedRadioTabIndex`/`novelSelectedRadioTabIndex`; debounced suggestion pipeline (`debounce(500)`, `:44-60`); `triggerAllRefreshEvent()` posts refresh events to all three result fragments (`:92-97`).
- `buildSearchConfig` (`:99-127`): maps the radio tab index → `SortType` (popular-preview/date_desc/date_asc/popular_desc), derives `usersYori` + `search_target` — the config object is rebuilt at submit time from ViewModel state.
- `SearchViewPagerFragment` (`search/SearchViewPagerFragment.kt`): `navArgs` carry `keyword` + `landingIndex` (`:28-35`); `commitEditingTag` appends the draft to `tagList` and fires the refresh event (`:109-125`) — "add token then refresh-all" rather than per-field route state.
- History: Room — `generalDao().getAllByRecordTypeLiveData(RecordType.VIEW_TAG_HISTORY)` (`SearchViewModel.kt:37-40`), `SearchDao`/`SearchEntity` under `ceui.lisa.database`.

## Takeaways for W2

1. One owner for the working query (activity-scoped `SearchModel` / `SearchViewModel`) shared by input + all result tabs; result pages never hold private copies — pixiv-func should keep `SearchQuery` in the route and let the result header edit → `replaceSearchResults`.
2. Durability tiers are explicit: durable **defaults** (`searchDefaultSortType`, `searchFilter`) vs session working state (ViewModel) vs URL-ish args (navArgs `keyword`). Mirror: settings = defaults, route params = working query, PageStorage = scroll.
3. `nowGo`/refresh-event broadcast = the pattern also usable for re-tap-to-top (a "refresh-all" style trigger distinct from a scroll trigger).
4. Shaft does NOT serialize full filter state into a shareable URL; pixiv-func already exceeds it (route params) but drops 8 filter fields — see codebase-search.md gap note.
