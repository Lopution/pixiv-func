# Ugoira playback & GIF export research (beta56 verified)

Date: 2026-08-27 (task 08-26-ugoira-player-export)

Reference repo pinned at `c62b18ccb92a64fedde81c8d5a2bc95cfc8f5989`
(`/tmp/pixiv_func_ref`). Audit context: `../08-27-open-source-pixiv-app-plan-audit/research/source-evidence.md`.

## Visible behavior (beta56)

### Cover state (`lib/pages/illust/ugoira_viewer/ugoira_viewer.dart`)

- Inline in illust detail (NOT a separate route): `state.init == false` renders
  `PixivImageWidget(previewUrl, colorBlendMode srcOver, tint white24 /
  black45 dark)`, fit-width, height 180 placeholder spinner, `Hero('IllustHero-$id')`.
- Center overlay: loading ? `CircularProgressIndicator` :
  `Icon(Icons.play_circle_outline_outlined, size: 70)`.
- Tap cover → `controller.play()`; while loading it retries via
  `Future.delayed(333ms, _playRoutine)` (controller.dart `play/_playRoutine`).

### Player state (`lib/components/frame_gif/frame_gif.dart` + controller)

- When init: `VisibilityDetector(key GIF-$id)` wraps a `CustomPaint` drawing the
  current `ui.Image` fitted `BoxFit.fitWidth` inside full-width SizedBox,
  `repaint: indexValueNotifier`.
- Tap while playing → toggle pause; paused shows tint overlay
  (white24/black45) + play icon size 70. No separate progress bar anywhere.
- Offscreen (`visibleFraction == 0`) → `stop()`; visible again → `start()`
  preserving pause state (`playing` vs `isPause` are separate flags).
- App lifecycle: observer sets `isActivity` = resumed-only; ticking halts on
  background implicitly, resumes in place.

### Loading sequence (`ugoira_viewer/controller.dart`)

1. Toast `获取动图信息成功前`: `ApiClient.getUgoiraMetadata(id)` — delays copied
   from `ugoira_metadata.frames[].delay`; failures toast `获取动图信息失败`.
2. ZIP downloaded with a dedicated Dio client (`Referer: https://app-api.pixiv.net/`)
   from `toCurrentImageSource(zip_urls.medium)`; receiveTimeout 60s; whole body
   into `Uint8List` (**memory unbounded** — replaced).
3. `PlatformApi.unZipGif(bytes)` unzips ALL entries to bytes;
   `_generateImages()` decodes EVERY frame to `ui.Image` ahead of first paint
   (**memory unbounded** — replaced with disk-indexed bounded decode).
4. Render size derived from first frame: width = screen width, height scaled.

### Save flow (download mode)

- Detail controller `downloadGif()` → `UgoiraViewerController.save()`, page 0
  becomes `IllustSaveState.downloading`, result maps back via
  `IllustController.downloadComplete(0, saveResult)`.
- Native side (`PlatformApi.kt saveGifImage`): bytes from `externalCacheDir/$id.gif`,
  `androidndkgif GifEncoder ENCODING_TYPE_FAST` per-frame quantization, delay
  units = centiseconds tick, saved through gallery insert; boolean result.
- Toasts: `开始合成图片(N)` → success/failure.

## Metadata endpoint schema (re-verified 2026-08-27)

- `GET /v1/ugoira/metadata?illust_id=<id>` via ApiClient
  (`lib/app/api/api_client.dart` getUgoiraMetadata) →
  `{"ugoira_metadata": {"zip_urls": {"medium": "https://i.pximg.net/img-zip-ug...,", ...},
  "frames": [{"file": "000000.jpg", "delay": 80}, ...], "mime_type": "image/jpeg"}}`.
- Delay is integer milliseconds, strictly ordered by array position; filenames
  reference ZIP entries 1:1.

## Internal rewrites (visible behavior frozen)

| beta56 | Replica v1 |
|---|---|
| Whole ZIP in `Uint8List` | streamed into owned app-cache temp file |
| `unZipGif` all entries | SafeZip central-directory random access, typed limits first |
| All frames decoded ahead | header-checked bounded decode + LRU window with `ui.Image.dispose` |
| Recursive `Future.delayed` chaining (drift accumulates) | monotonic deadline/ticker timeline modulo loop duration |
| NDK GIF encoder (native FAST type) | Dart isolate encoder (RGB555 median-cut palette + LZW), frame-by-frame pulled from disk index |
| boolean saveGifImage | user-visible job snapshot, exactly-one terminal event, MediaStore pending commit/abort owned by the job |

Audit decisions adopted here (P0/P1): typed archive/frame/pixel limits before
pixel allocation; header-before-decode; owned temporary outputs cleaned by
owner; exactly-once group post-process; final progress/terminal bypasses
throttling. Rejected patterns (kept out): all-frames-in-memory, sync IO on
large blobs, silent corruption-to-default delay substitution.

## Risks / notes

- GIF delay granularity is centiseconds; Pixiv 80ms frames round-trip exactly,
  arbitrary ms values round half-up (documented; cannot exceed format limits).
- `ui.Image.toByteData(rawStraightRgba)` runs in engine; quantization/LZW moved
  off UI isolate per PRD risk note (one worker per export job, killed on cancel).
