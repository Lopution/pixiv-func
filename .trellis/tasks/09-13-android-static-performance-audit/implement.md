# Implementation plan: Android static performance audit

This task produces a read-only report. Do not edit Dart, Kotlin, Gradle, generated localization, test snapshots, or dependency files.

## Checklist

- [x] Record branch, worktree status, Flutter SDK command path, and the existing user changes that overlap the audit.
- [x] Capture exact current and `HEAD` snippets for `PixivImage.decodeWidthFor`, `kFeedCacheExtent`, feed callers, global image-cache sizing, and Android/desktop scroll branches.
- [x] Trace the feed image/decode path and Hero route path; annotate each operation as setup-only, per-build, per-scroll-cache child, or per-animation-tick.
- [x] Inspect pinned source commits for the comparison clients and record only source-backed differences with direct tree links.
- [x] Write `research/static-audit.md` with a P0/P1/P2 table, causal chain, Windows contrast, confidence labels, and runtime evidence still required.
- [x] Run `/opt/flutter-3.47.2/bin/flutter analyze --no-pub` (or the repository's configured equivalent) and the focused existing tests relevant to Hero, scrolling, and image variants.
- [x] Run `git diff --check`; record all checks and any unavailable device/profile checks in the report.
- [x] Re-read the report for unsupported claims, especially exact FPS, Android DPR, memory pressure, and other clients' smoothness.

## Safe execution rules

- Keep all pre-existing worktree modifications and untracked files intact.
- Do not run ADB, install APKs, connect a device, or alter refresh-rate settings.
- Planning approval was obtained before `task.py start`; keep any later product implementation in a separate approved task.
- If later implementation is approved, split it into a separate task/phase; do not smuggle fixes into this audit.

## Quality gates

The report is acceptable only when every high-priority finding has a file/symbol anchor, a mechanism, a Windows/platform explanation, an evidence label, and an explicit runtime limitation. Static analysis and tests may pass without proving device smoothness; the report must preserve that distinction.
