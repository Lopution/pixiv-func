# Reverse-image provider feasibility

Observation date: 2026-08-28 (Asia/Shanghai). This is a capability and
contract audit only; no user image, account cookie, API key, or credential was
sent to a provider.

## Candidate matrix

| Candidate | Observed entry point | Capability conclusion | Product consequence |
|---|---|---|---|
| SauceNAO | Official API documentation URL below returned HTTP 403 with a Cloudflare managed challenge from the research environment. The public home page was reachable and exposed a Terms link, but the API contract, quota and privacy text could not be inspected without completing the challenge. | `unavailable` for this app build; no blind HTML scraping or challenge replay. | Keep the provider visibly unavailable until its API contract and credentials are reviewed. |
| TinEye | Official OpenAPI document below returned HTTP 200 and describes `POST /search/` multipart upload, an `X-API-KEY` security scheme, supported image formats and server-side resizing guidance. The official Legal link points to `/terms`; that page was challenge-gated in this environment. | `structuredApi` conditionally available only with a product-owned key, approved data terms and a server-side secret boundary. It is not eligible as a bundled client provider. | Do not ship a key, send images, or add a direct third-party route in this leaf. The client reports the capability blocker explicitly. |
| Google Lens | Official Lens upload endpoint redirected to the real Lens page (`HTTP 302`, no image was uploaded). No stable public developer result contract was found at this entry point. Google Cloud Vision Web Detection is a separate structured API, whose official docs require a Google Cloud project with billing enabled and authenticated API requests. | `interactiveWebView` for the consumer page, but not a beta56 result-card provider; `structuredApi` is a separately credentialed Cloud product. | Do not silently hand off a local image to a browser or parse Lens HTML. Keep both paths out of the shipped provider until privacy, result mapping and credentials are approved. |

## Official sources and observed evidence

- SauceNAO API entry: <https://saucenao.com/user.php?page=search-api> — observed
  `HTTP 403`, `cf-mitigated: challenge`; no API request was made.
- SauceNAO home/terms navigation: <https://saucenao.com/> — observed home page
  and a Terms link; the linked terms content was not treated as verified when
  the API page was challenge-gated.
- TinEye API documentation: <https://services.tineye.com/developers/tineyeapi/>
  — official documentation identifies the REST server and links to its
  OpenAPI document.
- TinEye OpenAPI contract:
  <https://api.tineye.com/rest/docs/openapi.json> — observed `POST /search/`,
  `X-API-KEY`, multipart input, and JPEG/PNG/WebP/GIF/BMP/AVIF/TIFF format
  notes. The document does not grant this project credentials or permission to
  ship a client-side key.
- TinEye legal link: <https://tineye.com/terms> — official link was reachable
  only as a challenge-gated page from this environment, so no legal claim is
  inferred from it.
- Google Lens upload entry: <https://lens.google.com/upload> — observed
  redirect to the real Lens page; no upload was attempted.
- Google Cloud Vision Web Detection:
  <https://docs.cloud.google.com/vision/docs/detecting-web> — official docs
  describe web references, matching pages and visually similar images, and
  require an authenticated Cloud Vision request with billing enabled.

## Implementation decision

The feature implements the common picker/SEND input, bounded validation,
preview/privacy/cleanup state machine, typed provider capabilities and a
fail-visible unavailable provider. A provider adapter can be added later only
when its structured contract, ToS/privacy review and secret ownership are
confirmed. The unavailable result is an explicit terminal state, not a mock
success or a swallowed network failure.
