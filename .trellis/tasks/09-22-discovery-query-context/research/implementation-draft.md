# Implementation Draft — W2 QueryContext

Maps `design.md` §4.2/§5.1 to concrete ownership. Not implementation — the contract W2 lands against existing code.

## 1. Per-page QueryContext ownership

### Recommended (`/recommended`)

| Field | Proposed owner | Layer | Today |
|---|---|---|---|
| content type (`illust/manga/novel/user`) | route `?type=` via `replaceRecommendedType` facade | route-durable | `_type` in `State` only (recommended_home_page.dart:47) |
| scroll | `PageStorageKey/restorationId 'recommended-<type>'` + **new explicit per-type `ScrollController`** | memory + Flutter restoration | keys exist (:305/:308); no controller |
| feed | `recommendedFeedProvider(type)` family | provider | already per-type |
| re-tap (branch) | pager re-tap signal → active type's controller `animateTo(0)` | new | absent |

Adding `?type=` mirrors ranking's `?mode=` precedent (routes.dart:977-980) and makes the selector survive process death; `TabController` seeded from the param, `_onTabChanged`/`_selectType` write `context.replace`.

### Ranking (`/ranking`, illust)

| Field | Owner | Layer |
|---|---|---|
| mode | route `?mode=` (existing) | route-durable |
| scroll | `ranking-<mode>` keys + existing per-mode controller map (`ranking_page.dart:43,80-82`) | memory + restoration |
| feed | `rankingFeedControllerProvider(mode)` | provider |
| re-tap (mode tab) | add `TabBar.onTap` → same-index → `_scrollControllerFor(mode).animateTo(0)` | new (TabBar currently has no onTap for same-index) |
| novel entry | keep AppBar icon → pushed `/novel-ranking` | unchanged |

### Novel ranking (`/…/novel-ranking`, pushed)

| Field | Owner | Layer | Change |
|---|---|---|---|
| mode | route `?mode=` (NEW — route today passes `const NovelRankingPage()`, routes.dart:549-553) | route-durable | add param + `replaceNovelRankingMode` facade |
| switching UX | adopt `RootSwipeSwitcher + TabSlideStack` + per-mode mounted bodies | shared-rule parity | today body is swapped via `ValueKey(mode)` (`novel_ranking_page.dart:90-95`), no swipe |
| scroll | existing `novel-ranking-<mode>` keys + controller map (:33,64-66) | keep |
| re-tap | same `onTap` → controller `animateTo(0)` | new |

Illust/novel ranking share one click/drag rule set: same scrollable TabBar chrome, same `onTap` semantics (different index → switch + route write; same index → scroll top), same swipe switching. The cross-type entry stays the AppBar action (W6 owns item variants, not this).

### New (`/new`)

| Field | Owner | Layer | Change |
|---|---|---|---|
| scope (`following/everyone/myPixiv`) | route `?scope=` (NEW) | route-durable | today `_selectedIndex` only (:37) |
| type (`illust/novel`) | route `?type=` (NEW) + **persistently visible selector** | route-durable | today hidden behind `_selectorExpanded` (:39,70-74) |
| selector presentation | dedicated always-visible control (segmented button row or filter chips under the AppBar) | — | removes "re-tap to expand" entirely |
| re-tap (scope tab) | same-index `onTap` → active `NewFeedKey`'s controller `animateTo(0)`; `_onTabTap` expand logic deleted (:70-74) | new behavior per design §5.1 |
| scroll | existing `new-<scope>-<type>` keys + per-body controllers (:230,:299) | keep |
| feed | `newFeedProvider(NewFeedKey)` family | keep |

### Search

| Field | Owner | Layer | Change |
|---|---|---|---|
| input `q`/`type` | route params (existing, routes.dart:1013-1020) | route-durable | keep |
| input draft text/selection | `TextEditingController` in page State | memory (+ optional `RestorableTextEditingController`) | keep |
| default filters (input page) | `searchFiltersProvider` → `SettingsRepository` | durable | keep; declare as "defaults" tier |
| result query (all 13 filter fields) | route params — **complete `_searchFilters`/`_searchQueryParameters`** (routes.dart:295-347) with `ai`, `bmin`, `bmax`, `ratio`, `ct`, `wmin`, `wmax`, `hmin`, `hmax` | route-durable | fixes the silent-drop gap |
| result header | keyword becomes an editable entry (tap → `openSearchInput(q,type)` prefilled; or inline field) + filter-summary row + clear | QueryContext visibility | today title is read-only Text (:66-71) |
| empty state | `FeedEmpty` gains a "modify search" action → push input prefilled / pop to edit | Astra row 5 | today only retries (:138-145) |
| suggestion tap | row tap = **fill** (write controller, no submit); trailing action = immediate search | §4.2 "区分填入与立即搜索" | today tap = fill+submit (:419-424) |
| trending kind | `trendingKindProvider` (memory) | declare memory-only | keep; illust default per session |
| popular tags | render **all** tags: `itemCount: tags.length` (drop the `%3` trim at :153-155); keep 3-col grid (short final row) or switch to `Wrap` chips | §4.2 + W1 coordinate | today drops 1-2 tags |
| scroll | `search-<cacheKey>` keys | keep; add explicit controllers for re-tap |

### Reverse image (`/reverse-image`, root push)

| Field | Owner | Layer |
|---|---|---|
| engine | `settingsProvider.selectReverseImageEngine` (durable) + session `initialEngine` | durable + memory — keep |
| input file, phase, results, webView/webUpload, engineFailures | `reverseImageSearchControllerProvider(session)` autoDispose family | **memory, declared** — temp file is session-owned; process-death recovery is out of scope for the image itself |
| `initialReference` | route `extra` | first-frame snapshot only (spec-conformant) |
| persistent task header | NEW: pinned summary (thumbnail + engine chip + phase) above phase bodies for ready/searching/failure/success-empty/webView states | W2 "图片、引擎和状态上下文" |
| in-flow cancel | `controller.cancel` → `ready`/`idle` (exists, :427-449); page-level back stays navigation — W1 fixes `_cancelAndPop` semantics (:117-120,:297); W2 consumes the corrected cancel and keeps the header | W1 dependency |

## 2. Re-tap-to-top implementation point

**Branch-level re-tap (bottom nav same-destination tap):**
- Publish from `BranchSlidePager.selectIndex` same-index path (`branch_slide_stack.dart:140-143`): after `goBranch(index)` (which pops the branch stack to root), emit a tick on a pager-owned `ValueNotifier<int>/Stream<int>` (e.g. `pager.reTapEvents`) — same shape as PixEz `topStore.setTop`.
- Root pages subscribe via `BranchSlideStack.maybeOf(context)` (post-frame listen in `initState`) and scroll their **active** scrollable's controller `animateTo(0, MotionTokens.gated)`. Because `goBranch` pops pushed routes first, the consumer must be the root page listening after pop completion — a notifier tick consumed post-frame covers this.
- Feeds lacking explicit controllers (recommended bodies, search home, search results) need per-key `ScrollController`s — required anyway on desktop because `SmoothWheelScroll` injects its own controller there (`smooth_wheel_scroll.dart:160-177`) so `PrimaryScrollController.of` cannot reach them.

**In-page selector re-tap (tab/chip same-index):**
- `TabBar.onTap` same-index → current mode/scope/type controller `animateTo(0)`. Pages with controller maps already support it; add `onTap` where missing (ranking). New page: re-tap goes to the **scope tab** same-index path; type chips' same-index tap also scrolls top (never toggles visibility).

**Rules (design §5.1):** pure scroll — never `refresh()`, never opens/closes UI; animated (`MotionTokens`-aware), reduced-motion → `jumpTo(0)`.

## 3. Restoration-level declarations (per §5.1)

| Surface | In-memory | Flutter restoration | Route-durable | Process-death |
|---|---|---|---|---|
| Recommended type | — | — | `?type=` (new) | type + scroll |
| Ranking mode | — | — | `?mode=` (exists) | mode + scroll |
| Novel-ranking mode | — | — | `?mode=` (new) | mode + scroll |
| New scope/type | — | — | `?scope=&type=` (new) | scope/type + scroll |
| Search input draft | cursor/selection | (optional restorable controller) | `q`/`type` | q/type |
| Search results query | — | — | full param set (extend) | query + scroll |
| Trending kind | session | — | — | resets to illust (declared) |
| Reverse-image session | input/phase/results/webView | — | — | engine only; flow resets (declared, temp-file bound) |
| Filter defaults, engine | — | — | — | SettingsRepository (exists) |

## 4. design.md §4.2 item mapping

| §4.2 item | Covered by |
|---|---|
| 发现页标签可读性、刷新反馈靠近当前内容 | remove `FittedBox(scaleDown)` (recommended :158-164, new :106-112); refresh error surfaces near header not only tail (recommended :264-282 pattern) |
| 排行类型/模式明确；插画/小说同一套切换规则；变体归 W6 | novel-ranking adopts RootSwipeSwitcher+TabSlideStack+`?mode=`; no new item components built |
| 新作范围/类型持续可见；重复点击=回顶 | always-visible type control; `_selectorExpanded` + `_onTabTap` removed; re-tap→top |
| 搜索共享可编辑查询头、筛选摘要、清除、空结果修改 | result header edit entry + filter summary chips + clear; empty state gains modify action |
| 建议项区分填入/立即搜索 | split tap vs trailing action (:419-424 today) |
| 反向搜图各阶段保留图片/引擎/状态上下文 | persistent task header across ready/searching/failure/empty/native/WebView results |

## 5. Leaf template prerequisites (implement.md §4)

- Copy W2 rows from traceability §4 (rows 1-5, 21) into the leaf.
- Owning files: the six feature files + `branch_slide_stack.dart` (re-tap signal) + `routes.dart` (new params) + `search_models.dart` (param names) + l10n keys. Shared owners consumed: `SmoothWheelScroll`, `RootSwipeSwitcher`, `feed_states` — extend, don't fork.
- Behavior changes needing PRD migration notes: new-page re-tap expand → scroll-top; suggestion tap → fill (was submit); novel-ranking swipe/tab semantics.
