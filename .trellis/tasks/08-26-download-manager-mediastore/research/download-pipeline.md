# Download pipeline & MediaStore research

Date: 2026-08-26 (task 08-26-download-manager-mediastore)

## beta56 semantics (source verified: `/tmp/pixiv-func-reference.oOgoAc`)

- `lib/app/downloader/downloader.dart`:
  - GET with headers `{Referer: https://app-api.pixiv.net/}` only (plus Dio defaults).
  - Filename = URL last path segment; dedupe by filename; a **failed** task may be
    re-submitted (dedupe skips only non-failed).
  - `maxDownloadCount` from settings caps concurrent tasks (original default 3;
    mutex-based rough limiter, one lock per finished task).
  - Progress = received/total; UI list updated via isolate port messages.
  - Toast on task start, save success, save failure; `onComplete(index, ok)` for detail page.
  - Original transport defects we intentionally do NOT replicate:
    `badCertificateCallback = true` (TLS bypass), full-bytes responses
    (`ResponseType.bytes`) crossing isolate via `compute`.
- `lib/models/download_task.dart`: `DownloadState {idle, downloading, failed, complete}`
  + progress double; UI task list is filename-keyed.
- `moe.xiaocao.pixiv.util.Image.kt saveImage`:
  - API >= 29: `MediaStore.Images.Media.EXTERNAL_CONTENT_URI` insert with
    `DISPLAY_NAME`, MIME from extension, `RELATIVE_PATH = Pictures/PixivFunc`;
    write via `openOutputStream`; on any exception `contentResolver.delete(uri)`.
    Note original never sets `IS_PENDING`; we set `IS_PENDING=1` on insert and
    clear it on finalize, which is the scoped-storage-correct variant of the
    same visible behavior (file appears in Pictures/PixivFunc only on success).
  - API < 29: direct file under `Pictures/PixivFunc` + `MediaScannerConnection.scanFile`.
- Missing MIME / unknown extension in original → null MIME column; we reject
  extensions outside the image/zip allowlist instead (R4 安全规范化).

## Platform contract decisions

- Channel `pixivfunc/mediastore`:
  - `begin(displayName, mimeType) -> {id}` — API>=29 inserts pending item in
    `Pictures/PixivFunc`; API<29 returns error `unsupported` (see below).
  - `write(id, bytes)` — appends chunk to the pending item (chunk = network
    chunk, bounded memory; MethodChannel Uint8List per-chunk is not a
    full-file isolate transfer).
  - `finalize(id) -> uriString` — clears `IS_PENDING` (MediaScanner for <29 kept
    in Kotlin for symmetry but unreachable while unsupported).
  - `abort(id)` — deletes pending row/file; idempotent, never throws through cleanup.
- API < 29: legacy path needs `WRITE_EXTERNAL_STORAGE`, which the design
  explicitly forbids ("不保留旧 broad storage permission"). targetSdk 36 devices
  are API 33+; on API 24–28 the channel fails fast with `unsupported` and the
  task ends `failed` with that reason. Recorded as explicit degradation, not a
  silent no-op.
- Device verification (real MediaStore behavior, pending cleanup, OEM quirks)
  **not performed** in this environment (no device/emulator); recorded as the
  same class of deferred acceptance as OAuth login.

## Transport decisions

- `dart:io HttpClient` with `followRedirects = false`; redirect hops resolved
  manually (max 5), each `Location` host validated against the allowlist
  (`i.pximg.net`, `s.pximg.net`) — R7: no auto-follow to foreign hosts, no
  certificate bypass anywhere.
- Shared single `HttpClient` instance = shared keep-alive connection pool (R1).
- Streaming: `response.stream` chunks flow directly into the pending sink
  (R2); no `readAsBytes`/full-file `Uint8List`.
- TLS failures (`TlsException`/`HandshakeException`) surface as task failure.

## Dart-side type mapping

- Extensions allowlist: jpg/jpeg/png/gif/webp/zip (zip reserved for ugoira export).
- MIME: jpg/jpeg→image/jpeg, png→image/png, gif→image/gif, webp→image/webp, zip→application/zip.
- Filename rules: non-empty, ≤255 chars, charset `[A-Za-z0-9._-]`, no `/`, `\`,
  `..`, control chars, must keep its extension (traversal-safe, R4).
