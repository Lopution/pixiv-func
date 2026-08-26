# Illust detail & viewer research (beta56 verified)

Date: 2026-08-26 (task 08-26-illust-detail-viewer)

## beta56 detail page (`lib/pages/illust/illust.dart`, controller.dart)

- App bar: `illust.title` (bold). Actions (when download mode): Download All icon
  with badge showing remaining count (`none`/`error` states; shows 全部 when all
  pending); plus bookmark switch (bookmark mutation = task 9, deferred here).
- Images: single page → one `PixivImage` (fit width, Hero `IllustHero-$id` on
  index 0); multi-page → vertical SliverList of pages; ugoira → UgoiraViewer
  (out of scope: we show cover + typed route to player, R7).
- Tap image → `ImageScalePage(initialPage: index)`. Long-press → toggle
  download mode (page tint white24 in light mode).
- Download mode per-page top-right button: none/error → download icon (tap
  downloads/retries), downloading → spinner, exist → check (primary color).
  States keyed by page index (`IllustSaveState`).
- Info block order: avatar(48)+name(16 bold)+account(12)+follow → upload date,
  views, bookmarks → resolution `WxH`, `ID: n`, share → summary expandable →
  tag chips wrap (`#name translatedName`, size 14, radius 10, surface color).
- Tag tap → SearchIllustResultPage(keyword); tag long-press → block mode
  (blocked badge icon on chip top-right, primary color when blocked). No
  copy-tag behavior anywhere.
- Related/comment tabs exist in beta56 but belong to later tasks (comments task;
  related feed is parent-scope) — deferred, not silently omitted: detail page
  renders without tabs in this task.

## beta56 viewer (`scale/scale.dart`)

- Horizontal PageView (`ExtendedImageGesturePageView`), title `n/total`.
- URLs: `scaleQuality` setting ? original : large; single page uses
  `metaSinglePage.originalImageUrl` else `imageUrls.large`; multi uses
  `metaPages[i]`.
- GestureConfig: minScale 0.9, maxScale 6.0, initialScale 0.95, speed 1.0,
  gaplessPlayback, headers `Referer: https://app-api.pixiv.net`.

## Detail API

- `GET /v1/illust/detail?illust_id=<id>` → `{illust: {...}}`; same illust JSON
  shape as feed items plus `meta_pages[i].image_urls.{original,...}` and
  `meta_single_page.original_image_url`; `visible:false` marks
  deleted/restricted works (R1). Uses shared PixivHttpClient (auth + refresh).

## Decisions

- Modern viewer: `PageView` + per-page `InteractiveViewer`
  (minScale 0.9 / maxScale 6.0, panEnabled), page swiping disabled while a
  page is zoomed past 1.0 (gesture boundary per PRD risk note); page index
  restored from constructor arg and reported back via title.
- Download states map to DownloadManager tasks keyed (illustId, pageIndex):
  no task→none, queued/running→downloading, failed→error(retry via
  manager.retry), canceled→none, succeeded→exist. Beta56's
  `imageIsExist` pre-check for previously saved files is deferred (needs
  MediaStore query surface; noted as platform gap, states start at none).
- Blocked tags persist in `SharedPreferencesAsync` key `blocked_tags`
  (StringList of tag names); service exposed as riverpod Notifier for chip UI.
- Minimal tag result page: `/v1/search/illust?search_target=
  partial_match_for_tags&word=<tag>` via shared PagedFeedController + existing
  NextPageParser allowlist entry; placed under `lib/features/search/` so the
  later Search task reuses the controller/page.
- Toast platform bridge does not exist yet in this replica; beta56 toasts for
  download start/finish/block-tag are represented visually (mode overlay,
  per-page state icons, block badge). Toast bridge itself stays with the
  platform-parity gap list, not faked here.
- Replica route rhythm (R8): custom PageRouteBuilder sliding in from the
  right (iOS-style), used for detail/viewer/tag-result pushes; Hero tag
  `IllustHero-<id>` on the detail first image.

## Deferred to their own tasks (explicit)

- Bookmark button/mutations → 08-26-bookmark-state-sync.
- Related works + comments tabs → comments task / parent scope.
- Ugoira playback/export → 08-26-ugoira-player-export (badge + typed route only).
- History recording → 08-26-history-persistence.
- Real-device gesture/back verification → recorded as unverified (no device).
