# Codebase: Motion Primitives Inventory & Deviation Points

Research base: `main@8067b2d` (branch `docs/09-22-ui-leaf-planning`, working tree of `/root/Pixiv-func`). All line numbers verified against this HEAD. W1–W9 leaf task dirs exist in `planning` state — nothing merged yet; the code below is the pre-roadmap baseline W10 will verify against.

## 1. Shared motion primitives (`lib/app/motion/`)

| File | Public API | Reduced-motion handling | Notes |
|---|---|---|---|
| `motion_tokens.dart` | `MotionTokens` (all durations/curves), `MotionScope` (InheritedWidget) | `enabled()` L82-86 = `MediaQuery.disableAnimations` OR `MotionScope.reduce`; `resolve()` L91-93 collapses duration to `Duration.zero` | Single choke point. `MotionScope` mounted in `app.dart:200-210` (`builder`, inside MaterialApp → covers Navigator incl. `useRootNavigator` dialogs) reading `settings.reduceMotion` |
| `page_transitions.dart` | `FuncRouteTransition` (L14), `FuncModalTransition` (L53), `RoutePopSnapshot` (L104) | Duration gated at call site `routes.dart:166,198` via `resolve()`; Hero flights ride the route animation → collapse automatically | `TickerMode` freeze + `SnapshotWidget` raster snapshot during transitions |
| `feed_entrance.dart` | `StaggeredEntrance` (L33) | Gate at L127-131: `!MotionTokens.enabled(context)` → `_markDone()` (static end state, counts as played) | Fling gate `_flingGateVelocity=1000` L62; viewport-exposure trigger; once-per-`played`-set keyed by entity `id` |
| `hero_transition.dart` | `illustHeroTag` L15, `illustHeroFlightShuttleBuilder` L53, `IllustHeroFlightChild` L27 | Implicit via route duration; no explicit gate needed | Progress-aware global clip via `HeroRectClip`; `_kHomeBottomNavHeight=80` fallback L40; reads `homeShellMetricsProvider` |
| `hero_rect_clip.dart` | `HeroRectClip` (SingleChildRenderObjectWidget) | n/a | `debugLastPaintClipRect` exposes render-time clip for tests |
| `drag_to_dismiss.dart` | `DragToDismiss` (L8) | **NOT gated** — return animation `duration: MotionTokens.fast` at L43 is raw; platform flag still auto-collapses the `AnimationController` (5%), in-app `reduceMotion` does not | Consumer: `image_viewer_page.dart:158` |
| `press_scale.dart` | `PressScale` (L25) | Gated: `MotionTokens.resolve` at L106 + `TickerMode` check L97-107 | Raw `Listener` for scroll-takeover release |
| `app_overlays.dart` | `showAppBottomSheet` L9, `showAppDialog` L36 | Gated: `MotionTokens.enabled` → `AnimationStyle.noAnimation` at L23-28 / L49-51 | The §5.5 mandated overlay entries |

## 2. Navigation / switching motion

| File | Mechanism | Reduced-motion handling |
|---|---|---|
| `root_swipe_switcher.dart` | `RootSwipeSwitcher` horizontal drag → tab strip first, edge hand-off to `BranchSlidePager`; `TabSlideStack` L371 slides bodies on `controller.animation` | `_landOn` L256-259 `animateTo(duration: enabled ? null : Duration.zero)`; `_settleTo` L262-270 snaps when `!enabled` |
| `branch_slide_stack.dart` | `BranchSlidePager` (L21, ChangeNotifier wrapping a real `TabController`); `BranchSlideStack` container L224; scroll-hide `_navVisibility` | `motionEnabled` callback L253 wired to `MotionTokens.enabled`; `endDrag`/`selectIndex`/`syncIndex` use `duration: _motionEnabled() ? null : Duration.zero` (L131,150,165); `_setNavHidden` L335-345 snaps `value` when disabled |
| `func_bottom_nav.dart` | `FuncBottomNav` elastic indicator + landing-ink replay; `FuncShellBottomNav` covered/scroll slide-outs | Gate at L114,139 (indicator+ink replay), L636,661 (visibility controllers snap to end value) |
| `pull_to_refresh.dart` (`lib/app/`) | `PullToRefresh` = EasyRefresh `BuilderHeader`+`MaterialHeader`, `triggerOffset:100`, `clamping:false` | Not gated — indicator is refresh *feedback*, not decoration (decision: keep) |
| `scroll_behavior.dart` (`lib/app/`) | `FuncScrollBehavior` — desktop drag devices + `BouncingScrollPhysics(AlwaysScrollable)` + no overscroll glow | n/a (physics, not animation) |
| `smooth_wheel_scroll.dart` | `SmoothWheelScroll` wheel tick smoothing | `_animationDuration = MotionTokens.wheelScroll` L79, used ungated L122-132 with `Curves.linear` — functional scroll, not decoration (decision: keep; platform flag does NOT collapse scroll-position animations anyway — `AnimationBehavior.preserve`) |

## 3. Settings plumbing (already shipped)

- `AppSettings.reduceMotion` field `app_settings.dart:255` (persisted, JSON round-trip tested).
- Toggle UI: `browse_settings_page.dart:349-356` (`SettingsControl`, label `reduceMotion` / hint `reduceMotionHint`, localized zh/en/ja/ru).
- Writer: `settings_controller.dart:120` `setReduceMotion`.
- Consumer: `app.dart:201` `MotionScope(reduce: settings.reduceMotion)`.

## 4. Feedback channel (#48 — must stay single-source)

- `app_snack_bar.dart`: `buildAppSnackBar` L21, `showAppSnackBar` L49 (branch-aware margin via `homeShellMetricsProvider`+`BranchRootScope`), `showAppSnackBarOn` L76, shared `appSnackBarAnimationStyle` L12-15 (medium in / fast out).
- All `showSnackBar` call sites go through these entries. Non-bypass uses: `home_page.dart:111` (exit hint, `showAppSnackBarOn`), `app.dart:87` (update prompt via `_messengerKey`, `showAppSnackBarOn`). `BranchRootScaffold` (`func_bottom_nav.dart:829`, mounted per branch at `routes.dart:791`) hosts the per-branch `ScaffoldMessenger`.
- **Gap**: `appSnackBarAnimationStyle` is passed unconditionally (L90). Platform `disableAnimations` still auto-collapses it (Snackbar's controller is `AnimationBehavior.normal` → 5% duration), but in-app `reduceMotion` does not. Whether to gate it is a §4.10 decision point — SnackBar is feedback, spec allows keeping it.
- **No `HapticFeedback` usage anywhere in `lib/`** — the §5.6 shared haptic owner does not exist yet (W4 creates it with download/save as first consumer). W10 acceptance = grep confirms a single owner file and no second `HapticFeedback.*` call family.

## 5. Deviation points (hardcoded or ungated animation)

| Location | Code | Issue |
|---|---|---|
| `lib/features/illust/detail/widgets/detail_image_pager.dart:65-69` | `_controller.animateToPage(next, duration: Duration(milliseconds:180), curve: Curves.easeOut)` | Hardcoded 180ms+easeOut, bypasses `MotionTokens` AND both reduced-motion sources (180ms ≈ `fast` but unlinked). Narrow-fix candidate. Owner: W4 page |
| `lib/features/novel/novel_page.dart:670-728` | `_ChromeBar` controller `duration: MotionTokens.fast` + `forward()/reverse()` | Uses token but never checks `MotionTokens.enabled` → in-app reduceMotion still animates chrome slide/fade. Owner: W5 |
| `lib/features/novel/novel_reader.dart:279-283, 363-367` | `_pageController.animateToPage(duration: MotionTokens.fast, curve: fastCurve)` | Token used, no `enabled`/`resolve` gate → in-app reduceMotion still animates page turns. Owner: W5 |
| `lib/features/new/new_page.dart:134-136` | `AnimatedSize(duration: MotionTokens.fast)` for `_selectorExpanded` | Ungated implicit animation; also `_onTabTap` L72 toggles `_selectorExpanded` — the re-tap→expand behaviour §5.1 replaces with re-tap→top (W2) |
| `lib/app/motion/drag_to_dismiss.dart:43` | `_returnAnimation` `duration: MotionTokens.fast` | Ungated (see §1) — cancel-return animates under in-app reduceMotion |
| `lib/app/widgets/app_snack_bar.dart:90` | `snackBarAnimationStyle: appSnackBarAnimationStyle` | Unconditional (see §4 — decision point) |
| `lib/app/pixiv_image.dart:46,89,647-648` | `fadeDuration=MotionTokens.imageFade`, `fadeOutDuration: MotionTokens.imageFadeOut` | Feeds OctoImage `fadeInDuration`/`fadeOutDuration` — NOT gated. Image cross-fade is load feedback + the fadeOut>fadeIn asymmetry is load-bearing against the white-flash regression; collapsing could regress. Keep-by-default, document in W10 |
| `lib/features/localnovel/local_novels_page.dart:112` | Raw `showDialog<bool>` | Bypasses `showAppDialog` → no `MotionTokens.dialog`, no reduced-motion gate. Owner: W6 (management page overlay migration); W10 verifies convergence |
| `lib/features/settings/pages/frame_probe_page.dart:44` | `Timer.periodic(Duration(milliseconds:500))` | Data poll cadence, not animation — excluded |
| `lib/features/illust/detail/ugoira_viewer.dart:399` | `Duration(milliseconds: frame.delayMs)` | Ugoira frame timing = content behaviour; spec: reduced motion must NOT remove it. L129-155 also gates playback start on route-animation completion (zero-duration route → immediate play: consistent) |
| `lib/features/home/home_page.dart:164-180` | `NavigationRail` extended transition | Framework-owned animation; platform flag collapses it, in-app setting doesn't (known framework limit) |

**Framework-owned residual**: `TabBar.onTap` → `TabController.animateTo` runs `kTabScrollDuration` internally; pages cannot inject a zero duration without replacing tap semantics (`controller.index = i; offset = 0`). Platform `disableAnimations` collapses it (AnimationBehavior.normal), in-app `reduceMotion` does not — W10 must decide accept-vs-shim per §4.10 "同级分类点击/拖动过程一致".

**Dead-ish surface**: `FuncSemanticTokens.motion{Short,Standard,Emphasized}` (`func_semantic_tokens.dart:136-140`) alias `MotionTokens.{fast,medium,pageTransition}` but have **zero consumers** — not a second constant set (values are references), but an unused parallel API. W10 "no second constant source" check should note it; deletion is optional narrow-fix.

## 6. Re-tap / QueryContext current state (baseline for §5.1 verification)

- `new_page.dart`: `_onTabTap` L72 toggles `_selectorExpanded` (Astra #3, confirmed) — W2 must repurpose to scroll-to-top.
- `recommended_home_page.dart:154`: segmented `onTap` → `onChanged(type)` only; no re-tap hook.
- `ranking_page.dart:104-126`: `TabBar` + `RootSwipeSwitcher` + `TabSlideStack` (already converged pattern); `novel_ranking_page.dart:77` still a plain `TabBar` with index-swap body (Astra #2).
- `BranchSlidePager.selectIndex` (`branch_slide_stack.dart:137-151`): same-index → `goBranch` (branch-root reset), NOT scroll-to-top — bottom-nav re-tap currently navigates. The §5.1 "re-tap = back to top, no refresh" contract has **no implementation yet** (W2/W3 scope); W10 regression-verifies it lands and carries no hidden expand entry.

## 7. Terminal-state animation summary for §4.10

- feed→detail→viewer spatial continuity: `illustHeroTag` scoping + `illustHeroFlightShuttleBuilder` + `DragToDismiss` + `FuncRouteTransition` — one chain, no parallel transitions found.
- Keyboard/form geometry: `home_page.dart:160` sets `resizeToAvoidBottomInset:false` on the shell Scaffold (overlay-not-compress contract, see component-guidelines "Common Mistakes"); leaf pages manage `viewInsets` padding themselves.
- Loading not blocking content: `StaggeredEntrance` triggers on viewport exposure not mount; `PixivImage.preload` on pointer-down must not block route push (contract in component-guidelines).
