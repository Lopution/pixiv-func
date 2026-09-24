# External: Reduced-Motion & Motion-Spec References

## 1. Flutter official reduced-motion contract

Source: `api.flutter.dev` `MediaQueryData.disableAnimations`, `SemanticsBinding.disableAnimations`, `AnimationBehavior`, `AccessibilityFeatures` docs; SDK 3.47.2 source (`/opt/flutter-3.47.2`).

- `MediaQuery.disableAnimations` = the platform asking to "disable or reduce animations as much as possible"; corresponds to **Android's "Remove animations"** a11y setting (also fires on Android animator-duration-scale=0).
- It is read from the engine via `SemanticsBinding.disableAnimations` — **overriding `MediaQueryData` in a `MediaQuery` widget does NOT affect framework animations** (`AnimationController`-driven). It only feeds custom code that reads `MediaQuery` itself. (Flutter docs, verbatim caveat.)
- Framework side: `AnimationController` defaults `animationBehavior: AnimationBehavior.normal` → under the flag, `forward()` duration scales ×0.05 (`animation_controller.dart:651`) and fling velocity ×200 (L785); `AnimationBehavior.preserve` (unbounded/physics controllers) is untouched. So platform-flag coverage is *framework-wide and automatic*; app-level gates are needed for the *in-app* setting and for choosing snap-vs-animate semantics rather than 5%.
- **iOS gap**: `AccessibilityFeatures.reduceMotion` exists separately and **does not set `disableAnimations`** (Flutter docs). An app-level gate that only reads `MediaQuery.disableAnimations` misses iOS "Reduce Motion" users. Reaching it: `View.of(context).platformDispatcher.accessibilityFeatures.reduceMotion` or `SemanticsBinding.instance.accessibilityFeatures.reduceMotion` (widget-testable via `accessibilityFeaturesTestValue = FakeAccessibilityFeatures(reduceMotion: true)`).
- Testing hooks: `tester.platformDispatcher.accessibilityFeaturesTestValue = FakeAccessibilityFeatures(disableAnimations: true)` drives MediaQuery + SemanticsBinding + AnimationController together; `debugSemanticsDisableAnimations` is a debug override on `SemanticsBinding`.
- Flutter guidance for custom explicit animations: check the flag and "reduce duration or skip non-essential animations" — the app's `MotionTokens.enabled`/`resolve` pair is exactly this pattern.

## 2. Material motion spec (m3.material.io/styles/motion)

- M3 Expressive moved to a spring-based `MotionScheme` (expressive/standard); the **easing+duration token system remains for transitions** and is what this codebase implements via `MotionTokens`.
- Duration scale (legacy tokens, still the reference grid): short1–4 = 50/100/150/200ms (small utility transitions), medium1–4 = 250/300/350/400ms (medium-area transitions), long1–4 = 450–600ms (large expressive transitions). Suggested pairs: emphasized 500ms on-screen, decelerate 400ms enter, accelerate 200ms exit; standard 300/250/200.
- Repo mapping sanity check: `MotionTokens` values (press 120, fast 180, medium 200, dialog 220, listEntrance 220, sheet 250, navIndicator/pageTransition 300, imageFade 500) sit inside the M3 short/medium band — consistent with spec, no drift.
- Motion roles for the acceptance matrix (design §4.10 wording aligns):
  - *State transition / spatial continuity* — Hero flight, route slide, drag-to-dismiss: collapsible to instant state change under reduced motion, but state must still land (already the `resolve` semantics).
  - *Feedback* — press scale, ink, snackbar in/out, refresh indicator, loading fades: "remove the flight, never the state it communicates" (motion_tokens.dart doc). Reduced motion may keep a fade while dropping translation/scale.
  - *Content behaviour* — Ugoira playback, scroll physics: NOT decoration; reduced motion must not gate them.
  - *Decorative* — entrance stagger, indicator elastic replay, landing ink: first candidates to collapse.

## 3. A11y / motion regression practice in larger Flutter projects

- Flutter renders to a canvas; TalkBack/VoiceOver read a **parallel semantics tree** — the only automatable surface is `tester.ensureSemantics()` + `tester.getSemantics(finder)` / `matchesSemantics` / `tester.semantics.find` (fine-grained node assertions preferred over `TestSemantics` whole-tree string compare — flutter/flutter#184367 direction).
- Automated a11y checks cover roughly a third of WCAG-relevant issues; screen-reader *paths* still need a human on TalkBack (Android) / Narrator (Windows) / VoiceOver (iOS). Design §7's "at least one screen-reader representative path verified or explicitly marked unverified" matches industry practice.
- Semantics-based substitutes usable in CI: focus order (`orderedEquals` on semantics traversal), label/role presence (`matchesSemantics(label:…, isButton:…)`), `ExcludeSemantics`/`BlockSemantics` correctness for offscreen branches — the repo already `ExcludeSemantics`s off-window branch pages (`branch_slide_stack.dart:433`).
- Golden tests: appropriate only for static visual regression (this repo's `golden_matrix_test.dart` = component states light/dark). Not suitable for asserting motion correctness — trajectory must be asserted via widget state/terminal-state tests (repo quality guide already mandates "assert the terminal state, not only the trajectory").

## 4. Implications for W10

1. Platform `disableAnimations` already collapses all `AnimationBehavior.normal` controllers — the app's gate mainly adds (a) the in-app `reduceMotion` source and (b) clean snap semantics instead of 5% slow-mo. Both halves already exist.
2. iOS `reduceMotion` is a genuine coverage hole: one-line widening of `MotionTokens.enabled` (OR the reduceMotion flag) + one widget test via `accessibilityFeaturesTestValue` closes it — candidate narrow fix.
3. Test strategy: app-gate assertions via `MediaQuery`/`MotionScope` injection; framework-collapse assertions via `accessibilityFeaturesTestValue`; screen-reader items must be marked manual/未验证 in this environment.
