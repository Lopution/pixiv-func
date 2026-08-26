# Recommended illust API schema verification (2026-08-26)

## Source

- `shenshengkafei/pixiv_dart_api` HEAD: `lib/model/illust.dart`,
  `lib/model/image_urls.dart`, `lib/model/user.dart`, `lib/model/tag.dart`,
  `lib/vo/illust_page_result.dart`, `lib/model/profile.g.dart`
  (ProfileImageUrls{medium}).
- beta56 `lib/pages/recommended/illust/source.dart` (pagination via
  `next_url`), `lib/components/illust_previewer/illust_previewer.dart`
  (card badges/layout).

## Verified response contract

- Endpoint: `GET https://app-api.pixiv.net/v1/illust/recommended`
  with `content_type` (illust|manga), `include_ranking_illusts`, `filter`,
  `min_bookmark_id_for_recent_illust`, `offset` query params.
- Response: `{"illusts": [...], "next_url": "...|null"}`.
- Illust JSON keys: `id, title, type ("illust"|"manga"|"ugoira"),
  image_urls {square_medium, medium, large, original?}, caption, restrict,
  user {id, name, account, profile_image_urls {medium}, is_followed?},
  tags [{name, translated_name?}], tools, create_date, page_count, width,
  height, sanity_level, x_restrict, meta_single_page
  {original_image_url?}, meta_pages [...], total_view, total_bookmarks,
  is_bookmarked, visible, is_muted, illust_ai_type, total_comments?}.

## Card semantics (beta56 IllustPreviewer)

- R-18 badge top-left when `x_restrict == 1` (illust.isR18).
- Ugoira "gif" badge bottom-left when `type == "ugoira"`.
- Page count badge top-right when `page_count > 1`.
- AI badge bottom-right when `illust_ai_type == 2`.
- Title 14 bold ellipsis; user name 12 ellipsis; aspect ratio
  `width/height` from the illust dimensions; bookmark switch button on the
  trailing side.
