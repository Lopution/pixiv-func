# Flutter Android API 36 工程基线设计

## Design objective

以 Flutter `3.47.0` stable 官方 Android Kotlin 模板为宿主层，只补齐平台工程和质量门禁，不重写当前 `lib/` UI shell。生成物必须能由 Flutter 工具重新识别和维护，不能退回旧版 Gradle 集成方式。

## Boundaries

```text
existing lib/ + pubspec.yaml
        │
        ├── Flutter 3.47.0 template generated android/
        │       ├── Kotlin DSL + Flutter Plugin DSL
        │       ├── AGP 9.1.0 / Gradle 9.3.1
        │       ├── API 36 compile + target
        │       └── built-in Kotlin enabled
        │
        └── repository quality files
                ├── .gitignore
                ├── analysis_options.yaml
                └── focused widget/startup smoke test
```

The existing `lib/` remains the source of truth for the current UI. Android generation happens from an isolated temporary Flutter project or an equivalent non-destructive template run; only reviewed generated platform/config files are merged into this repository. This avoids allowing a stock template `lib/main.dart` or example test to replace the existing shell.

## Android template strategy

1. Install or locate the exact Flutter `3.47.0` SDK and verify `flutter --version` before generation.
2. Generate a temporary Kotlin Android app with Flutter's own `flutter create` command, using project name `pixiv_func` and the requested organization seed.
3. Review the generated files, then merge the Android directory and root template files that do not conflict with the existing application.
4. Normalize the generated Android identifier to `io.github.lopution.pixivfunc` in the namespace, application ID, Kotlin source package path, and manifest references.
5. Keep `android/settings.gradle.kts` and the Flutter Gradle plugin loader from the template. Do not manually recreate the old `flutter.gradle` `apply from` flow.

## Kotlin and Gradle contract

Flutter `3.47.0` officially supplies the API 36, AGP 9.1.0, Gradle 9.3.1 and Kotlin 2.4.0 template versions. The generated app module should use the modern Kotlin DSL and Flutter plugin DSL. `android/gradle.properties` must explicitly enable `android.builtInKotlin=true`; the app module must not apply `org.jetbrains.kotlin.android`/`kotlin-android` as an application plugin. Java and Kotlin compiler targets remain JVM 17 as required by the template.

The generated project may contain the template's compatibility property for the new AGP DSL. It is retained or adjusted only after the actual Flutter 3.47.0 build confirms compatibility; no legacy Gradle script is introduced to bypass AGP 9.

## Package and display identity

The canonical identity is:

| Concern | Value |
|---|---|
| Dart package/project name | `pixiv_func` |
| Android namespace | `io.github.lopution.pixivfunc` |
| Android application ID | `io.github.lopution.pixivfunc` |
| Android display label | `Pixiv Func` |
| Android language | Kotlin |

The old package names `moe.xiaocao.pixiv` and `site.xiaocao.pixiv` are not introduced.

## Quality files and test shape

Use the Flutter 3.47.0 root `.gitignore` and `analysis_options.yaml` as the baseline, reviewing them against the existing Trellis layout. The test should exercise a real existing widget path, preferably the startup gate with deterministic settings, and should avoid depending on a live account, network, WebView, or Android device. It may use a small test-only provider override or direct widget construction if that is the least coupled way to make the smoke test deterministic.

## Preservation and rollback

- Before generation, record `git status --short` and the current HEAD.
- Generate outside the repository where possible; this makes rollback a matter of discarding the temporary output rather than restoring existing source.
- Merge only new/owned paths and review any `pubspec.yaml` or root config changes line by line.
- If Flutter generation modifies an existing file unexpectedly, stop and keep the generated copy out of the repository until the diff is understood.
- Do not use destructive Git restoration commands to remove unrelated user changes.

## Deferred compatibility surface

Deep links, predictive back, WebView, MediaStore, SEND intents, FileProvider, and API 36 edge-to-edge behavior are product requirements for later Android work. This scaffold establishes the host and target SDK only; adding those manifest/runtime contracts belongs to the feature tasks that need them.
