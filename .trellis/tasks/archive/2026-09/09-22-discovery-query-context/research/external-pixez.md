# External — PixEz (Flutter, github.com/Notsfsssf/pixez)

Source: PixEz `master` via jsDelivr, files fetched to `/tmp/dl_*.dart` / `/tmp/pixez_*.dart` on 2026-09-22. Line numbers are from those copies; repo paths noted per section.

## Re-tap-to-top broadcast (`lib/store/top_store.dart`)

```dart
// top_store.dart:15-18,32-35
_TopStoreBase() {
  _streamController = StreamController();
  topStream = ObservableStream(_streamController.stream.asBroadcastStream());
}
setTop(String name) {
  LPrinter.d(name);
  _streamController.add(name);
}
```

- `NavigationBar.onDestinationSelected` / `NavigationRail` emit `"${index + 1}00"` **only on same-index re-tap** (`android_hello_page.dart:193-201`, `:231-239`):
  ```dart
  if (this.index == index) {
    topStore.setTop("${index + 1}00");
  }
  ```
- Pages subscribe to `topStream` with their own code:
  - New page (`new_illust_page.dart:48-51`): `if (event == "301") { _scrollController.position.jumpTo(0); }`
  - Rank page (`rank_page.dart:73-77`): `"200"` → re-emits mode-specific `201 + index` so the active ranking sub-page scrolls:
    ```dart
    if (event == "200") { topStore.setTop((201 + index).toString()); }
    ```
  - Search guide (`search_page.dart:69-73`): `"400"` → `_openSearch()` pushes the suggestion page (re-tap = open input, PixEz's search tab is a guide page).
- Lighting list (`lighting_page.dart:88-96`): after a forced refresh with content, `jumpTo(0.0)` — refresh implies top in PixEz's list stores.

**Contract shape**: a string-code broadcast stream decouples the nav bar from page controllers; each page maps its own code → `jumpTo(0)`. Pixiv-func equivalent: a `reTapProvider`/`Stream`-backed notifier the pager publishes and page bodies consume, routed by branch index (+ in-page tab index for ranking).

## Search filter persistence (`lib/page/search/result_illust_list.dart`)

```dart
// :121-141 — restore when the "remember" setting is on
final recordRememberCurrentSelectionKey =
    'illust_search_result_record_remember_current_selection';
recordRememberCurrentSelection =
    Prefer.getBool(recordRememberCurrentSelectionKey) ?? false;
if (recordRememberCurrentSelection) {
  searchTarget = Prefer.getString(searchTargetKey) ?? search_target[0];
  selectSort = Prefer.getString(searchSortKey) ?? "date_desc";
  searchAIType = Prefer.getInt(searchAIKey) ?? 0;
  ugoiraFilter = UgoiraFilter.values[Prefer.getInt(ugoiraFilterKey) ?? ...];
}
// :148-158 — persist on change
await Prefer.setString(searchTargetKey, searchTarget);
await Prefer.setString(searchSortKey, selectSort);
```

- Opt-in flag `illust_search_result_record_remember_current_selection`; when on, last-used target/sort/AI/ugoira filters persist via `Prefer` (shared_preferences wrapper) and seed the next search.
- Pixiv-func equivalent exists: `searchFiltersProvider`/`setSearchFilters` are always-persisted (input page), while result-page edits replace the route — two different durability levels today (see codebase-search.md).

## Search guide history presentation (`lib/page/search/search_page.dart`)

```dart
// :226-260 — history chips
if (targetTags.length > 20) {
  final resultTags = targetTags.sublist(0, 12);
  ... for (var f in _tagExpand ? targetTags : resultTags) buildActionChip(...)
  ActionChip(icon: expand_more/expand_less) → _tagExpand toggle
}
```

- History shows 12 of N when N>20, with an explicit expand chip that reveals **all** — nothing is dropped. Contrast: pixiv-func drops `tags.length % 3` trending tags silently (`search_page.dart:153-155`). For the W2 "keep all popular tags" requirement, PixEz's pattern = Wrap + collapse toggle that never discards items.

## Suggestion store (`lib/store/suggestion_store.dart:24-33`)

- MobX store; `fetch(query)` writes `autoWords`; errors swallowed (`catch (e) {}`). No debounce in the store (caller debounces). Pixiv-func's `searchAutocompleteController` already does more (debounce, cancel, stale-guard).

## Takeaways for W2

1. Re-tap-to-top needs a broadcast channel owned by the shell, codes per branch (+ sub-tab), page bodies subscribe and call their own scroll controller — pages already keep per-mode/per-key controllers in ranking/new.
2. Filter durability is an explicit opt-in in PixEz; pixiv-func persists defaults unconditionally — decide which level the QueryContext contract claims.
3. Never truncate popular tags silently; collapsed presentation must retain the full list (expand affordance or wrap layout).
