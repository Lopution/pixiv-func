# Flutter 3.47.0 Android 模板核验

## Sources checked

- Flutter `3.47.0` stable tag: https://github.com/flutter/flutter/tree/3.47.0
- Android template: https://github.com/flutter/flutter/tree/3.47.0/packages/flutter_tools/templates/app/android.tmpl
- Kotlin Android template: https://github.com/flutter/flutter/tree/3.47.0/packages/flutter_tools/templates/app/android-kotlin.tmpl
- Flutter Android Gradle constants: https://github.com/flutter/flutter/blob/3.47.0/packages/flutter_tools/lib/src/android/gradle_utils.dart
- Built-in Kotlin migration: https://docs.flutter.dev/release/breaking-changes/migrate-to-built-in-kotlin/for-app-developers

## Findings

- The `3.47.0` tag exists in the official Flutter repository.
- Flutter 3.47.0's Android Gradle constants are compile SDK 36, target SDK 36, AGP 9.1.0, Gradle 9.3.1, Kotlin Gradle plugin 2.4.0, and Java 17 minimum.
- The official app template uses `settings.gradle.kts`, the declarative Flutter plugin loader, and the `dev.flutter.flutter-gradle-plugin`; it does not use the legacy `apply from: flutter.gradle` flow.
- The Flutter 3.47 migration documentation states that built-in Kotlin can be enabled with `android.builtInKotlin=true` after the app has removed the application-level Kotlin Gradle Plugin application. The generated app module should therefore be checked for the absence of `org.jetbrains.kotlin.android`/`kotlin-android` application plugin usage.
- The template's generated compatibility properties may contain `android.newDsl=false`; this must be validated against the actual 3.47 build rather than replaced with handwritten legacy Gradle configuration.

## Build validation

- With `android.builtInKotlin=true` and no Kotlin plugin declaration in the root settings, AGP 9.1.0 exposes its bundled Kotlin `2.2.10`, which Flutter 3.47.0 rejects because its minimum supported version is `2.2.20`.
- Keeping the official template's root-only declaration `id("org.jetbrains.kotlin.android") version "2.4.0" apply false` supplies the version metadata required by Flutter without applying KGP in `android/app`.
- With that root declaration, `android.builtInKotlin=true`, and the template's `android.newDsl=false` compatibility flag, `flutter build apk --debug` completed successfully. The produced APK reports compile/target SDK 36 and the requested package identity.

## Implementation implication

Use the exact Flutter 3.47.0 SDK's `flutter create` output as the source of the Android scaffold, then make only the requested package identity and built-in Kotlin adjustments. Do not hand-copy an older Gradle layout or assume an unverified newer SDK is equivalent.
