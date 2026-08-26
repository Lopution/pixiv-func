# Flutter Android API 36 工程基线执行计划

## Preconditions and review gates

- [ ] Keep the task in `planning` until this plan is reviewed and the user explicitly approves starting implementation.
- [ ] Confirm the current task pointer is `flutter-android-scaffold`; do not start the unrelated automatic `00-bootstrap-guidelines` task.
- [ ] Record `git status --short --branch` and `git log -1 --oneline` before any generated files are merged.

## Ordered implementation checklist

1. **Prepare the toolchain**
   - Locate or install Flutter `3.47.0` from the official stable distribution without adding the SDK to the repository.
   - Verify Java 17 and Android SDK/API 36 availability with `flutter doctor` and the Android tooling checks.
   - Record exact tool versions before generation.

2. **Generate the platform scaffold**
   - Use `flutter create` with the Android Kotlin template in an isolated temporary directory or a non-destructive repository invocation.
   - Retain the official Kotlin DSL, Flutter plugin loader, wrapper, resources, and embedding v2 setup.
   - Merge only reviewed generated platform/config files; never overwrite the existing `lib/` shell with template code.

3. **Normalize Android identity and build settings**
   - Update the generated namespace, application ID, source package path, and label to the values in `prd.md`.
   - Verify compile/target API 36, AGP 9.1.0, Gradle 9.3.1, Java/Kotlin JVM 17, and `android.builtInKotlin=true`.
   - Remove any app-level KGP application or legacy `apply from` usage if generated output contains it; do not add a compatibility bypass.

4. **Add the repository quality baseline**
   - Add/review root `.gitignore` and `analysis_options.yaml` while preserving `.trellis/.gitignore` and generated Trellis files.
   - Add a deterministic non-placeholder widget/startup smoke test for the current shell.
   - Keep icon font migration and all Pixiv business implementation out of this task.

5. **Run the quality gate**
   - Run `flutter pub get`.
   - Run `flutter analyze` and fix only errors/warnings caused by the generated baseline or current Flutter API compatibility.
   - Run `flutter test`.
   - Run `flutter build apk --debug`.
   - Run `git diff --check` and search the generated Android files for forbidden legacy Gradle/KGP/ABI patterns.

6. **Review and finish the task**
   - Use `trellis-check` after implementation to verify scope, generated-file integrity, tests, and cross-layer preservation.
   - If a project-wide convention is learned, update the relevant `.trellis/spec/` file through `trellis-update-spec`; do not fill unrelated backend/frontend placeholders speculatively.
   - Only after the quality gate and review pass, follow Trellis Phase 3.4 for a scoped commit. Do not push unless separately requested.

## Validation commands

```bash
flutter --version
flutter doctor -v
flutter pub get
flutter analyze
flutter test
flutter build apk --debug
git diff --check
rg -n 'apply from:|flutter\.gradle|kotlin-android|org\.jetbrains\.kotlin\.android|splits|abi' android
```

## Risk and rollback points

- If Flutter `3.47.0` cannot be installed or Android API 36 cannot be resolved, stop at the precondition and report the real error; do not substitute an unverified SDK or claim build success.
- If `flutter create` proposes changes to existing Dart files or dependencies, preserve the original and merge only the requested generated paths after review.
- If AGP 9/built-in Kotlin fails because a dependency still applies legacy KGP, isolate that dependency failure and do not globally disable TLS, downgrade the target, or add old Gradle wiring.
- If a test requires platform plugins unavailable in the host test runner, replace it with a deterministic existing-widget test rather than a tautological empty test.
