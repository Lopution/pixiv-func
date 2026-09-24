# Implementation Draft: W10 Motion Integration & Final Acceptance

Scope rule (design §3/§4.10/implement.md §6 W10): verify, not rebuild. Narrow fixes only for integration-exposed problems; no new design system, no parallel entries, no new public motion widgets unless a §4.10 item cannot close without one.

## 1. Reduced-motion integration — where the platform flag is consumed

**Single choke point already exists**: `MotionTokens.enabled(BuildContext)` (`lib/app/motion/motion_tokens.dart:82-86`) = `MediaQuery.disableAnimations` OR `MotionScope.reduce`; `MotionTokens.resolve(context, base)` (L91-93) collapses a duration to zero. `MotionScope` is mounted in `MaterialApp.builder` (`app.dart:200-210`) from `AppSettings.reduceMotion`, so every route/sheet/dialog context sees it.

Correct consumption patterns (all already present — enforce, don't re-invent):

| Pattern | API | Reference site |
|---|---|---|
| Duration parameter | `MotionTokens.resolve(context, MotionTokens.x)` | `routes.dart:166,198`; `press_scale.dart:106` |
| Branch on gate | `MotionTokens.enabled(context) ? animate : snapToEndValue` | `func_bottom_nav.dart:114,139,636,661`; `branch_slide_stack.dart:253,336`; `root_swipe_switcher.dart:258,266`; `feed_entrance.dart:128` |
| Overlay presentation | `AnimationStyle.noAnimation` when `!enabled` | `app_overlays.dart:23,49` |
| Controller-in-callback (non-BuildContext) | inject `bool Function()` reading `MotionTokens.enabled` | `BranchSlidePager.motionEnabled` (`branch_slide_stack.dart:26,43`) |

**Narrow-fix candidates for the ungated consumers** (all file:line verified, details in codebase-motion-inventory.md §5):
- `detail_image_pager.dart:67` hardcoded 180ms+easeOut → `MotionTokens.resolve(context, MotionTokens.fast)` + `fastCurve`.
- `novel_page.dart` `_ChromeBar` (L670-728), `novel_reader.dart` (L279,363), `new_page.dart` `AnimatedSize` (L134), `drag_to_dismiss.dart:43` — token used but gate missing → wrap with `resolve`/`enabled`.
- `MotionTokens.enabled` iOS hole → OR `View.of(context).platformDispatcher.accessibilityFeatures.reduceMotion` (external-reduced-motion.md §1). One line + one test.
- `appSnackBarAnimationStyle` unconditional (`app_snack_bar.dart:90`) — decide: keep (feedback channel, platform flag already auto-collapses it) or gate via `AnimationStyle.noAnimation` when `!enabled`. Recommend keep + document; gating is a 3-line change if reviewers disagree.
- `TabBar` tap animation (framework-owned `kTabScrollDuration`): in-app setting cannot collapse it without replacing tap semantics (`controller.index = i`). Platform flag covers it. Recommend: document as known limitation rather than shimming every TabBar.
- Keep ungated on purpose: `PixivImage` fade durations (load feedback + white-flash asymmetry contract), `SmoothWheelScroll` wheel duration (functional scroll), `PullToRefresh` indicator (feedback), Ugoira `frame.delayMs` (content), `NavigationRail` extended transition (framework).

## 2. Per-contract regression checks (design §5)

| Contract | Check method | Automatable? |
|---|---|---|
| §5.6 SnackBar single channel | (a) grep gate: `rg "SnackBar\(|showSnackBar\(|ScaffoldMessenger\." lib/` — only `app_snack_bar.dart` + `home_page.dart:111`/`app.dart:87` (both route through `showAppSnackBarOn`) allowed; (b) narrow-fix candidate: layering-style test asserting no raw `SnackBar(`/`showSnackBar(` outside `app_snack_bar.dart`; (c) widget test `app_snack_bar_test.dart` extension: assert margin lift on branch root + `appSnackBarAnimationStyle` attached | yes |
| §5.6 Haptic single source | Pre-W4: `rg "HapticFeedback" lib/` → zero hits today. Post-W4: grep must show exactly one owner file + whitelisted call sites (W4 download/save; W6 manage-mode/destructive; W7 send-terminal). Any other `HapticFeedback.*` = fail | yes |
| §5.7 ActionTerminology | (a) l10n audit: `rg` the arb keys for 应用/保存/删除/移除/解除屏蔽/取消/重试/重新登录/重新打开/返回/查看 and eyeball pairing vs §5.7 table; (b) focused widget tests where leaves froze wording (e.g. download "查看"); (c) manual pass over changed pages' strings — mark which files were checked | partial (grep + spot tests; full pass = manual) |
| §5.2 ObjectPresentation variants | `rg "NovelCard|NovelRow|IllustCard"` call-site census post-W6: ranking/new/search/history etc. consume shared component + named variants; assert no feature-local card classes (R5 layering test pattern extensible) | partial (census automatable, visual parity manual) |
| §5.1 QueryContext + re-tap→top | Per leaf PRD declarations: check PageStorageKey/`restorationId`/`restorationScopeId` presence per page family (component-guidelines Route Restoration Contract); re-tap: widget test — tap current tab → `ScrollController` position 0, no refresh call, no expand toggle (replaces `new_page.dart` `_selectorExpanded` path); branch-root re-tap: assert `goBranch`-same-index semantics unchanged + scroll-top hook where W2/W3 add it | mostly yes (widget tests); process-death restoration = manual/emulator only |
| §5.3 FormState / §5.4 BackAndCancel | Not re-verified wholesale — leaves own them. W10 spot-checks: cancel-vs-leave on reverse image search (W1), dirty-confirm on profile edit, `PopScope` chrome-vs-page order on novel reader | partial |
| §5.5 overlays/breakpoints | Post-migration grep: `showDialog|showModalBottomSheet` outside `app_overlays.dart` should be empty (today: `local_novels_page.dart:112` — verify W6 migrated it); `AppBreakpoints`/`two_pane` consumers compile-locked | yes |

## 3. §4.10 line-item execution plan

| Item | How to verify | Auto/Manual |
|---|---|---|
| 同级分类点击/拖动一致 | `root_swipe_switcher_test` + `func_bottom_nav_test` already cover hand-off/indicator; W10 adds: run ranking + new + recommended tap AND drag paths under default + reduced motion; assert `TabSlideStack`/`BranchSlidePager` used everywhere (grep `novel_ranking_page` converged post-W2); spot-check no page reintroduces index-swap body | mostly auto; drag feel = manual |
| 作者头部仅改布局 | W3 owns; W10: `user_profile_test`/`author` tests + manual sweep that expanded/collapsed expose same actions (follow/share/more/stats reachable) | auto + manual |
| 查询信息输入/结果连续 | W2 owns; W10: `navigation_restoration_test`-style check — search input → result → back restores keyword/filters/type; assert AppBar shows editable context (post-W2) | auto |
| feed→detail→viewer 单一空间连续性 | `hero_transition_test` covers clip/flight; W10: (a) assert card→detail uses `illustHeroTag` scope pairing, viewer uses `DragToDismiss`, route uses `FuncRouteTransition` — grep each consumer hits the shared owner; (b) no second Hero tag family (`rg "Hero\("` census) | auto |
| 表单/键盘动画不动主控件 | Contract = shell `resizeToAvoidBottomInset:false` (`home_page.dart:160`) + leaf `viewInsets` padding. W10: widget test pumping a focused field in a sheet → assert primary action doesn't translate; manual on desktop keyboard | partial |
| loading 动效不延迟内容 | Assert entrance is exposure-triggered not gating content (already `feed_entrance` design + tests); assert `PixivImage.preload` not awaited before push (component-guidelines contract — grep `onTapDown` preload vs `onTap` push order); cold-route first frame shows snapshot via `initialEntity` | auto |
| reduced motion 只去装饰 | (a) matrix test: for each gated consumer assert state lands instantly (already per-widget in `motion_test`/`func_bottom_nav_test`); (b) negative assertion: Ugoira playback, scroll physics, SnackBar visibility still function under `reduce:true`; (c) apply §1 narrow fixes for the ungated list; (d) iOS hole fix | mostly auto |
| #48 SnackBar + MotionTokens 单一来源 | §2 table rows; plus `rg "Duration\(milliseconds" lib/` census stays at today's non-motion set (frame_probe poll, ugoira frame delay, wheel floor) + MotionTokens file; `FuncSemanticTokens.motion*` note (unused alias — keep or remove, decide) | auto |

## 4. Final acceptance matrix execution plan (design §7)

- **Automated layer** (runnable here): focused `flutter test --no-pub` on motion/nav/snackbar/reader files; new gate tests per §1 fixes; `analyze --no-pub`; `dart format` check; `git diff --check`; `task.py validate`. Full suite run once for regression.
- **Width matrix** (320/390/600/840/1200 + landscape): widget tests with `tester.view.physicalSize` (golden_matrix precedent) for breakpoint behaviour; real-window resize = manual/desktop.
- **Text scale** 1.0/1.3x: `MediaQuery(textScaler:)` injection in widget tests on key pages; long-translation = manual (ru locale spot check possible via `locale:` in pump).
- **Input**: touch = widget tests; system back = `PopScope` tests exist; keyboard Tab/Enter/Escape = `Focus`+`KeyEvent` widget tests (detail_image_pager precedent); mouse wheel/drag = `PointerScrollEvent` simulation + manual desktop.
- **A11y**: semantics-tree assertions via `ensureSemantics`/`matchesSemantics` on author header, viewer, comment input (design §7 priority pages); TalkBack/Narrator representative path = **mark 未验证 in this env**.
- **Feedback**: haptic on/off — unit-test the owner wrapper's gating if W4 builds one; device feel = manual.
- **Motion**: default vs reduced — `MotionScope`/`accessibilityFeaturesTestValue` injection, automatable for all gated widgets.
- **States**: loading/content/refresh-error/load-more-error/empty/busy/failure-retry — existing feed-state tests + per-leaf tests; W10 re-runs focused files, does not duplicate.
- **Matrix ledger**: final record must split implemented / unit-widget-tested / desktop-tested / device-tested / 未验证 per implement.md §7 — maintain a per-item table in the leaf's check output.

## 5. W10 gate checklist mapping (implement.md §6)

- "逐项关闭追踪矩阵" → build the per-item evidence table; no vibe-check closes.
- "默认/reduced、触摸/键鼠、中文/长翻译代表路径" → §4 rows; each row gets test id or 未验证 tag.
- "TalkBack/Narrator 已验证或标注未验证" → cannot run here; write explicit 未验证 + semantics-tree substitute evidence.
- "无新平行组件/冲突手势/过时文案/绕过 SnackBar/第二触觉来源" → §2 grep gates + optional layering-style static test.
- "只修窄问题" → §1 list is the entire sanctioned fix surface; anything bigger spawns a new task.
