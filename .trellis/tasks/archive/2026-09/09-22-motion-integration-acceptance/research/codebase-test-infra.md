# Codebase: Test Infrastructure & Verification Tooling

Base: `main@8067b2d`. Verified runnable: `/opt/flutter-3.47.2/bin/flutter test --no-pub test/motion_test.dart` → 17/17 pass (2026-09-22, this environment). `pubspec.lock` + `.dart_tool/package_config.json` present — deps resolved.

## 1. Toolchain

- SDK: `/opt/flutter-3.47.2/bin/flutter` (Flutter 3.47.2, rev d3b14c8769). `/opt/flutter-3.47.0` is rollback residue only.
- Gates (per `quality-guidelines.md` + `implement.md` Step D):
  - `flutter analyze --no-pub` — must be zero issues;
  - `dart format --output=none --set-exit-if-changed lib test` — CI runs before analyze;
  - `flutter test --no-pub` — full suite or focused files;
  - `python3 ./.trellis/scripts/task.py validate <task>`; `git diff --check`.
- Devices visible to `flutter devices`: **Linux desktop (linux-x64) + Chrome only** — WSL2, no Android emulator/device, no Windows host. `flutter run -d linux` needs a display server (WSLg) — unverified here; treat device/runtime checks as "not executable in this env" unless a desktop run is proven.

## 2. Test suite layout (138 test files)

- `test/` flat files + `test/architecture/` (layering rules), `test/goldens/` (10 PNG masters), `test/failures/` (golden failure artifacts), `test/helpers/` (5 helper files), `test/fixtures/` (saucenao payloads).
- Motion-relevant existing tests:
  - `motion_test.dart` (452 L) — gate matrix: `MotionScope(reduce:)` + `MediaQuery(disableAnimations:true)` injection helper `_wrap`; `MotionTokens.resolve` collapse ×4; `StaggeredEntrance` exposure/fling/once/TickerMode-frozen; `PressScale` scale/duration/scroll-takeover/frozen; `reduceMotion` JSON round-trip + corrupt fallback.
  - `func_bottom_nav_test.dart` (327 L) — pumps the **real home shell** via `createPixivRouter` + `ProviderScope` overrides + `memoryPreferences()`; asserts indicator/landing-ink/scroll-hide through the real `BranchSlideStack`.
  - `root_swipe_switcher_test.dart` (357 L) — drag hand-off strip↔branch.
  - `hero_transition_test.dart` (625 L) — flight clip assertions via `HeroRectClip.debugLastPaintClipRect` (render-time, not widget prop).
  - `app_snack_bar_test.dart` (155 L), `pull_to_refresh_test.dart` (298 L), `novel_reader_chrome_test.dart` (270 L), `navigation_restoration_test.dart` (313 L), `home_page_test.dart` (297 L), `shared_component_semantics_test.dart` (123 L), `responsive_layout_test.dart` (76 L).
  - `golden_matrix_test.dart` — light/dark × feed-empty/error/loading + author_summary + tag_chips + home_bar, `tester.view.physicalSize` + `matchesGoldenFile('goldens/…png')`.
- `test/architecture/layering_test.dart` — import-graph rules R1–R5 on `lib/` (allow-lists currently empty). **Pattern is directly reusable** for W10's "no second source" checks (e.g. a test that fails on `SnackBar(`/raw `showDialog`/`HapticFeedback` outside owner files).

## 3. Test-writing constraints already documented (quality-guidelines.md)

- `material_ui` shadows `flutter/material` types — pump `material_ui`'s `MaterialApp`, assert on `material_ui` `SnackBar`/`SwitchListTile` or app wrappers (`SettingsControl`); `showAppSnackBar` resolves `material_ui`'s `ScaffoldMessenger.maybeOf` — a plain Flutter `MaterialApp` silently drops it.
- dart:io loopback ~20% drop under full-suite parallelism → `tolerant()` 3-attempt wrapper for socket tests.
- `HistoryDatabase` needs injected `databasePath` per test (shared FFI path locks).
- autoDispose `.future` needs a held listener; `ref.read` before `watch` on autoDispose notifier = permanent spinner (reference fix in `reverse_image_search_page.dart` `listenManual`).
- Non-ASCII `http.Response` bodies → `http.Response.bytes(utf8.encode(...), charset utf-8)`.
- `InAppWebView`/`WebViewPlatform` need platform fakes on Linux test host (`reverse_image_search_page_test.dart`, `login_navigation_test.dart` precedents).
- Regression tests must be proven against unfixed code; assert terminal state not trajectory; delete tests that froze wrong behaviour together with the guard.

## 4. Reduced-motion test mechanics (verified against SDK + suite)

Two distinct layers, both reachable in widget tests:

1. **App gate** (`MotionTokens.enabled`): reads `MediaQuery.maybeOf(context)?.disableAnimations` + `MotionScope`. Test override = wrap subtree in `MediaQuery(data: MediaQueryData(disableAnimations: true))` or `MotionScope(reduce: true)` — existing `motion_test.dart` `_wrap` pattern. **Caveat (Flutter docs)**: MediaQuery override does NOT collapse `AnimationController`-driven framework animations — it only feeds code that reads `MediaQuery.disableAnimations` itself.
2. **Platform flag**: `tester.platformDispatcher.accessibilityFeaturesTestValue = FakeAccessibilityFeatures(disableAnimations: true)` (flutter_test `window.dart:759` in SDK 3.47.2; also `debugSemanticsDisableAnimations` debug override on `SemanticsBinding`). This drives BOTH `MediaQuery.disableAnimations` AND `SemanticsBinding.disableAnimations` → every `AnimationBehavior.normal` controller auto-scales duration to 5% (`animation_controller.dart:651`), fling velocity ×200 (L785). Use this for "framework path also collapses" assertions; use the MediaQuery injection for app-gate-only assertions.

Not covered by `disableAnimations` on the platform side: iOS "Reduce Motion" is `AccessibilityFeatures.reduceMotion` — a **separate flag that does not set `disableAnimations`** (Flutter docs). `MotionTokens.enabled` does not read it → iOS reduced-motion users currently get full animation unless they also flip the in-app toggle. W10 narrow-fix candidate: OR `View.of(context).platformDispatcher.accessibilityFeatures.reduceMotion` (or `SemanticsBinding.instance.accessibilityFeatures.reduceMotion`) into `MotionTokens.enabled`.

## 5. What is automatable vs manual

Automatable now:
- Gate wiring: every new `MotionTokens.enabled/resolve` call site assertable via `pumpWidget` + duration/opacity inspection (motion_test precedent).
- Route transition zero-duration: pump router (`createPixivRouter` + overrides per `func_bottom_nav_test`), `MediaQuery(disableAnimations:true)`, push route, assert `ModalRoute.transitionDuration == Duration.zero` or settled-in-one-pump.
- SnackBar single-channel: static grep + optionally a layering-style test scanning `lib/` for `SnackBar(`/`showSnackBar(` outside `app_snack_bar.dart`.
- Haptic single-source (post-W4): same grep/test scan for `HapticFeedback.` outside the owner file.
- re-tap→top (post-W2/W3): widget test tapping current tab → assert `ScrollController.position.pixels == 0` and no refresh call.
- Semantics: `tester.ensureSemantics()` + `getSemantics`/`matchesSemantics` exist in suite already (`shared_component_semantics_test.dart`).

Manual/unavailable here: TalkBack, Windows Narrator, real touch/haptic feel, Android predictive-back visuals, true 60fps jank assessment, iOS reduceMotion path. Mark "未验证" per design §7 — Widget test evidence must not be extrapolated to device pass.
