# Risks: W10 Execution Constraints & Unverifiable Surface

Base: `main@8067b2d`, environment = WSL2 Ubuntu 24.04, Flutter 3.47.2 at `/opt/flutter-3.47.2`.

## 1. Environment limits (what cannot run here)

| Capability | Status | Evidence |
|---|---|---|
| `flutter test` / `analyze` / `dart format` | **works** | `flutter test --no-pub test/motion_test.dart` → 17/17 pass, 2026-09-22 |
| `flutter devices` | Linux desktop + Chrome only | no Android emulator/device attached |
| Android runtime (touch, haptics, TalkBack, predictive-back visuals, EasyRefresh feel) | **not executable** | no emulator image/device; building an APK is possible (`android/` + `/opt/android-sdk` exist) but running it is not |
| `flutter run -d linux` | **unproven** | device listed, but headless WSL — needs WSLg/display; do not claim desktop-verified without an actual run |
| Windows Narrator / iOS VoiceOver | **not executable** | no Windows host / no iOS toolchain |

**Marking rule** (design §7 + traceability §7): every acceptance row that needs a device/desktop run and cannot be executed gets `未验证` with the exact missing surface named (e.g. "TalkBack path — no Android device"), never inferred from widget tests.

## 2. Items whose acceptance inherently depends on missing surfaces

- §7 "至少一条屏幕阅读器代表路径 (TalkBack/Windows Narrator)" → **cannot verify**; substitute = `ensureSemantics`/`matchesSemantics` traversal assertions on the three priority pages (author header, viewer, comment input) + explicit 未验证 tag.
- §5.6 haptic levels (selection vs manage-mode vs save-success) → owner doesn't exist yet (W4); even post-W4, feel is unverifiable — only call-site single-source is grep-able.
- Gesture feel items (§4.10 同级拖动一致, drag-to-dismiss threshold, pull-to-refresh retract) → terminal-state widget tests cover correctness; jank/手感 stays 未验证.
- Process-death restoration (§5.1 恢复等级 third tier) → needs real app kill/relaunch — manual/emulator only.
- True 60fps performance of `RoutePopSnapshot`/Hero clip → frame-timing assertions don't exist in suite; `FrameProbe` is a manual tool. Performance stays 未验证.

## 3. Technical risks

- **Two flag systems**: platform `disableAnimations` (auto-collapses every `AnimationBehavior.normal` controller to 5%) vs in-app `MotionScope.reduce` (only affects `MotionTokens.enabled/resolve` consumers). A widget that uses a MotionTokens constant without the gate animates under in-app reduceMotion — the exact gap list is in codebase-motion-inventory §5. Verify per-consumer, not per-token.
- **iOS hole**: `MediaQuery.disableAnimations` does not include `AccessibilityFeatures.reduceMotion` (Flutter docs). `MotionTokens.enabled` currently misses iOS reduce-motion users → one-line widening is the highest-value narrow fix.
- **MediaQuery injection ≠ platform flag in tests**: `MediaQuery(disableAnimations:true)` feeds the app gate but NOT framework `AnimationController`s; use `platformDispatcher.accessibilityFeaturesTestValue = FakeAccessibilityFeatures(...)` when asserting framework-level collapse. Mixing them produces false-pass tests.
- **TabBar tap animation can't be zeroed without replacing tap semantics** (framework `animateTo` internal) — accept + document, or shim per page; don't half-fix one strip.
- **`FuncSemanticTokens.motion{Short,Standard,Emphasized}`** — unused alias ramp of MotionTokens; zero consumers. Flag in the "no second constant source" check; removal optional.
- **Golden tests are machine-stable but motion-blind** — never use `matchesGoldenFile` as motion evidence; assert controller/duration/widget-state instead (suite already does).

## 4. Dependency risks

- W10 is the last wave; every §4.10 verification assumes W1–W9 merged. If a leaf hasn't landed, the corresponding row stays `blocked`/`未验证`, not silently green. Re-baseline all file:line references at W10 start (`implement.md` Step A) — this research's line numbers pin `main@8067b2d` only.
- The re-tap→top contract (§5.1) has no implementation yet; if W2/W3 scope shifts, W10's check list changes.
- `local_novels_page.dart:112` raw `showDialog` is W6-owned; if W6 leaves it, W10 either narrow-fixes (one-line swap to `showAppDialog`) or records as known gap.

## 5. Decision points to surface at planning review

1. Widen `MotionTokens.enabled` to include iOS `reduceMotion`? (recommend yes — 1 line + test)
2. Gate `appSnackBarAnimationStyle` under in-app reduceMotion? (recommend keep — feedback channel; platform flag already collapses it)
3. Fix ungated MotionTokens consumers (`_ChromeBar`, `novel_reader` pager, `new_page` AnimatedSize, `DragToDismiss` return, `detail_image_pager` hardcode) — all in W4/W5/W2-owned files; narrow-fix within W10 or file bugs back to leaves? Per design §3 "W10 只做集成验收与证据驱动的窄修复" — these qualify as 窄修复 if small.
4. PixivImage fade + wheel-scroll smoothing deliberately stay ungated (functional/feedback) — confirm at review.
