# Codebase — Ranking (插画排行 + 小说排行)

Scope: `lib/features/ranking/ranking_page.dart` (252 lines), `novel_ranking_page.dart` (205 lines), `lib/core/illust/ranking_feed_controller.dart`, `lib/core/novel/novel_ranking_feed_controller.dart`, route wiring in `lib/app/navigation/routes.dart`.

## Illust ranking (`ranking_page.dart`)

### Selector / switching
- `RankingMode` (11 values) rendered as a **scrollable** `TabBar` in the AppBar title (`:104-115`); labels resolved via `l10nLookupFor` — no `FittedBox` shrink.
- `TabController` seeded from `widget.initialMode` (`:50-56`); `_selectedIndex` mirrors it (`:45`).
- `_handleTabChanged` (`:70-78`) fires on tap **and** swipe settle: updates `_selectedIndex`, adds to `_loadedModes`, and calls `widget.onModeChanged` → `replaceRankingMode(context, mode)` (routes.dart:1249-1255 → `context.replace('/ranking?mode=…')`). Mode is **durable in the URL**.
- Novel-ranking entry is an AppBar action icon (`menu_book_outlined`, `:97-102`) → `openNovelRanking` pushes a page on the branch stack (routes.dart:1189-1190).
- `RootSwipeSwitcher` + `TabSlideStack` (`:117-139`) — horizontal swipe moves between modes; off-edge drags hand off to the branch pager (`root_swipe_switcher.dart:330-350`).
- Per-mode `ScrollController` map (`:43`, `:80-82`) — kept for the page lifetime; **this is the natural re-tap-to-top hook** for the visible mode.
- `TabBar` has no `onTap` for same-index taps — tapping the current mode does nothing today.

### Provider ownership
- `rankingFeedControllerProvider` — `AsyncNotifierProvider.family<PagedFeedState, RankingMode>` (`ranking_feed_controller.dart:71-76`); per-mode cursor validation via `RankingRepository.validateModeCursor` (`:60-68`).

### Scroll / restoration
- Per-mode `CustomScrollView`: `PageStorageKey('ranking-${mode.name}')` (`:212`), `restorationId: 'ranking-${mode.name}'` (`:216`), explicit `scrollController` passed into `SmoothWheelScroll` (`:208-209`) — controller survives type switches (map keyed by mode), so PageStorage covers widget destruction; restorationId covers process death.
- The mode itself restores from the URL (`?mode=`) — survives process death.
- Branch `restorationScopeId: 'ranking'` (routes.dart:985).

### Loading / error / empty
- `_RankingModeBody` (`:145-252`): loading → `FeedLoading`; async error → `FeedError` + `retryInitial`; `showInitialError` → `FeedError`; `isEmptyAndReady` → `ReplicaEmptyState` + refresh retry (`:183-191`); load-more failure → `FeedTail` (`:233-242`). Mode tabs stay visible throughout (AppBar).

## Novel ranking (`novel_ranking_page.dart`)

### Selector / switching — divergent from illust ranking
- `NovelRankingMode` (9 values), same scrollable TabBar chrome (`:77-88`).
- **Body is replaced, not slid**: `body: _NovelRankingModeBody(key: ValueKey(mode))` (`:90-95`) — no `TabSlideStack`, no `RootSwipeSwitcher`; horizontal swipe does NOT switch modes. Traceability row 2 confirmed.
- `_handleTabChanged` (`:59-62`) updates `_selectedIndex` only — **no route write**: `/novel-ranking` is a pushed route `const NovelRankingPage()` (routes.dart:549-553) with no `mode` query param, so the selected mode is in-memory only and resets to `day` on rebuild/process death.
- `_scrollControllers` map (`:33`, `:64-66`) — kept per mode even though only one body mounts at a time; `_entrancePlayed` set (`:34`) prevents entrance-animation replay.

### Provider / scroll / restoration
- `novelRankingFeedProvider` — `AsyncNotifierProvider.family<PagedFeedState, NovelRankingMode>` (`novel_ranking_feed_controller.dart:65-70`).
- `PageStorageKey('novel-ranking-${mode.name}')` (`:168`) + `restorationId` (`:172`); explicit `scrollController` into `SmoothWheelScroll` (`:164-165`).
- Feed states mirror illust ranking (`:116-203`); uses `NovelRow` + `StaggeredEntrance` keyed by entity id (`:178-183`).

## Gaps vs W2 contract

- Illust ranking: mode durable ✓, re-tap-to-top ✗ (TabBar has no same-index `onTap`), novel entry is an icon — content-type switch is one-way navigation, not a shared selector.
- Novel ranking: mode NOT in route, no swipe switching, body replaced not slid — "插画/小说共享同一套点击/拖动切换规则" requires converging both.
