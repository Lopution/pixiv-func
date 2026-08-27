# Ranking research checkpoint

Date: 2026-08-27

## Reference behavior

- Reference commit: `c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`.
- `lib/pages/ranking/ranking.dart` creates 11 tabs in the order `day`,
  `dayR18`, `dayMale`, `dayMaleR18`, `dayFemale`, `dayFemaleR18`, `week`,
  `weekR18`, `weekOriginal`, `weekRookie`, `month`.
- The reference uses one `RankingListSource` per mode and a two-column
  `SliverWaterfallFlowDelegateWithFixedCrossAxisCount` with main spacing 5
  and cross spacing 10.
- `lib/pages/ranking/content/source.dart` calls
  `getRankingPage(mode)` for the first page and `getNextPage` with the returned
  `next_url` for subsequent pages.

## Current API contract checkpoint

- The checked-out `pixiv_dart_api` source used by the reference maps the
  enhanced enum names to snake_case values (`day_r18`, `week_original`, and
  so on) and calls `/v1/illust/ranking` with `filter=for_android` and
  `mode=<value>`.
- The current implementation keeps this endpoint behind the centralized
  `NextPageParser` allowlist. Ranking cursors must retain the same mode,
  use the allowlisted host/path, use the Android filter when present, and
  carry a numeric offset.
- No live account/API response was used as a success substitute. R18 mode
  availability remains an explicit runtime error/empty state when the server
  rejects it.

## Implementation consequence

Ranking uses an explicit `RankingMode` mapping, an independent Riverpod
family controller per mode, the shared `IllustStore`/`BookmarkStore`, and a
page-owned `ScrollController` per tab. Only the selected tab body is built;
already loaded controllers and scroll positions remain available when the
user returns to a tab.
