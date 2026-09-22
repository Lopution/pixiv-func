# Codebase — Reverse Image Search (以图搜图)

Scope: `lib/features/search/reverse_image_search_page.dart` (862 lines), `lib/core/reverse_image/reverse_image_controller.dart` (573), route `'/reverse-image'` (`routes.dart:906-925`).

## Route and entry

- `openReverseImageSearch` pushes `/reverse-image` on the **root** navigator with `extra: initialReference` (`routes.dart:1301-1306`). The page reads `state.extra is ReverseImageInputReference` (`:912-916`) — per spec, `extra` is a first-frame snapshot only; nothing else is in the URL.
- Because it sits on the root stack (not the search branch), its `_commonBranchRoutes` let results push illust/user pages without stacking a second shell (`routes.dart:918-925`).
- Page-level `restorationId` = pageKey via `_page` (`routes.dart:169-171`), but **no scrollable-level key/id and no restorable session** — see below.

## Session and provider ownership

- `_ReverseImageSearchPageState` creates `ReverseImageSearchSession` in `initState` (`reverse_image_search_page.dart:66-82`): platform picker (Android `MethodChannel` / desktop), `providers` map (4 engines over `thirdPartyHttpClientProvider`), `initialEngine` from **`reverseImageEngineProvider`** (persisted setting, `settings_controller.dart:298-304`), `uploadArmer`.
- `reverseImageSearchControllerProvider` = `NotifierProvider.autoDispose.family<…, ReverseImageFlowState, ReverseImageSearchSession>` (`reverse_image_controller.dart:128-133`). Because the family key is a per-page `Session` object, **each page instance gets a fresh controller**; state dies with the page (autoDispose → `close()` → owned temp file released, `:152-156`, `:451-457`). No process-death recovery by design today — the picked image is a session-owned temp copy.
- Engine switching: `_selectEngine` writes **both** the flow (`_controller.selectEngine`) and durable settings (`settingsProvider.selectReverseImageEngine`) (`page:126-131`; controller `:255-286` — with a held image, ready/failure/success-with-webUpload returns to `ready` keeping `input` + `engineFailures`; headless success is terminal and not switchable, `:251-254`).

## State machine (`reverse_image_controller.dart:18-27`, `:45-86`)

`ReverseImageFlowStatus`: `idle → picking → preparing → ready → searching → success | failure | canceled`. `ReverseImageFlowState` carries `engine`, `input` (info: path/size/dims), `results`, `webView` (SauceNAO-style HTML result), `webUpload` (Cloudflare-fronted upload page: Ascii2D/TinEye), `failure`, `engineFailures` (per-engine failure memory, `:82-85`).

- `pick` (`:158-176`), `prepare` (`:178-249`, generation-guarded, owns temp file via `OwnedReverseImageInput`), `search` (`:288-425`, CancelToken; success → hits/webView/webUpload; failure keeps input for retry or engine switch `:336-339`), `cancel` (`:427-449` → `canceled`, releases input), `close` (`:451-457`).

## Phase rendering (`reverse_image_search_page.dart`)

- `_body` switch (`:177-202`): idle/canceled → `_idle`; picking/preparing/searching → `_progress`; ready → `_ready`; failure → `_failure`; success → `_uploadWebView` | `_resultWebView` | `_results`.
- `_idle` (`:232-255`): icon, intro, engine chips, privacy card, pick button.
- `_progress` (`:285-304`): spinner + label + **"cancel" button = `_cancelAndPop` → `controller.cancel()` + `Navigator.pop`** (`:117-120`) — cancel leaves the page entirely; Astra row 21 (W1 fixes the cancel outcome; W2 keeps context).
- `_ready` (`:306-368`): preview card (`_AdaptiveImagePreview` adapts to EXIF aspect, `:482+`), dims/size, reselect + in-flow cancel (`:335-347`), engine chips, unsupported-engine hint (`:351-358`), search button gated on `spec.supportsInput`.
- `_failure` (`:370-426`): error icon + mapped message (challenge/unavailable/dailyLimit/rateLimited[wait]/unsupportedInput), engine chips when input held (`:405-408`), same-engine retry (`:411-415`), pick-new (`:417-421`). **The image preview is not shown in failure** — context (which image, which engine) is text-only.
- `_results` (`:428-463`): **empty result = bare `Center(Text)`** (`:429-431`) — no image thumbnail, engine name, or re-search affordance; list rows show similarity%, title/Pixiv-id, open via `openIllust` or external launcher (`:456-458`).
- WebView results: `_resultWebView` chooses desktop `_ControlledSauceNaoInAppWebView` vs `_ControlledSauceNaoWebView` (`:204-218`); `_uploadWebView` uses `InAppWebView` everywhere because only its ChromeClient sees the armed file-chooser slot (`:220-230`); navigation policy `ReverseImageNavigationAction` handled at `:620-660` (navigate/openIllust/openUser/openExternal/reject).

## Restoration / persistence summary

- Durable: selected engine (`reverseImageEngineProvider`).
- In-memory only: input file, results, webView/webUpload objects, per-engine failures, flow phase. Page pop or process death resets to `idle` (new session, new autoDispose family member, temp file gone).
- Astra row 21 status "部分": failure already has switch/retry entries; missing persistent image+engine task header and cancel-stays-in-page semantics.
