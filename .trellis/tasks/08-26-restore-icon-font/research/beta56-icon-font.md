# beta56 icon font provenance

## Source

- Repo: `svenfuss/pixiv_func_mobile`
- Commit: `c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989` (`1.0.0-beta56+62`)
- File: `assets/icon.ttf`, fetched from
  `https://raw.githubusercontent.com/svenfuss/pixiv_func_mobile/c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989/assets/icon.ttf`

## Verification (2026-08-26)

- `file icon.ttf`: TrueType Font data, 11 tables, icomoon
- SHA-256: `b1277d76c133e157bd819b55477e9400880c84474d9cddffe25ecce3e7199c05`
- fontTools cmap check: glyphs present at exactly `0xe900`–`0xe90d` (plus
  control/space/'0' internals), matching all 14 `AppIcons` codepoints in
  `lib/app/icons/app_icons.dart`.
- Family name declared in pubspec as `iconFont` per beta56 `pubspec.yaml`.
