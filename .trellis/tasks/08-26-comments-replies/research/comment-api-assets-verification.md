# Comments/replies research evidence

验证时间：2026-08-27（本地固定 beta56 reference 与依赖源码）

## API contract

Source repository: `/tmp/pixiv-func-reference.oOgoAc`, commit
`c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`.

The companion `pixiv_dart_api` source is checked at commit
`d57cff0f052af08551d01957905f5f99a592036d`.

- `GET /v3/illust/comments?illust_id=...` returns `{comments, next_url}`.
- `GET /v2/illust/comment/replies?comment_id=...` returns `{comments, next_url}`.
- `POST /v1/illust/comment/add` accepts `illust_id`, optional `comment`,
  `stamp_id`, and `parent_comment_id`.
- `POST /v1/illust/comment/delete` accepts `comment_id` and is intended for
  the current user's own comment.
- Comment response fields are `id`, `comment`, `date`, `user`,
  `has_replies`, and optional `stamp`. The reply endpoint does not provide a
  reliable direct parent field, so the active root comment ID is retained as
  local thread context.

## Beta56 UI behavior

The fixed reference uses an explicit reply icon, avatar-to-user navigation,
an inline emoji renderer, a translation overlay, an own-comment delete action,
and a separate replies page. It does not use long-press reply. The composer
uses 10 emoji columns and 5 stamp columns.

## Asset provenance

Copied without transformation from the fixed reference:

- `assets/emojis/`: 38 PNG files
- `assets/stamps/`: 40 JPG files
- content-only aggregate SHA-256 (sorted `sha256sum` values):
  `0a29f3933f8dc70499caa3463cd0628e56e4f6b6d838040a50364e46c520e514`

The asset directories are registered in the root `pubspec.yaml` and the
picker order is captured by `commentEmojiNames` and `commentStampIds`.
