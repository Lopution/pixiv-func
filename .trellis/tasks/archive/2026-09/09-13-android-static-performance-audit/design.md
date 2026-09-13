# Technical design: Android static performance audit

## Scope and evidence model

The audit is read-only with respect to product source. Existing worktree edits remain in place and are treated as an explicit comparison state. Every finding uses one of these labels:

- **Code fact**: directly observable control flow, constant, platform branch, or third-party API behavior.
- **Strong static suspect**: a code fact with a plausible causal path to Android frame misses and a meaningful Windows/platform contrast.
- **Runtime required**: a hypothesis that needs a frame timeline, image-cache counters, or memory trace before it can be called a root cause.

No Android device is available. The report must never convert a strong static suspect into measured FPS, frame time, or confirmed device behavior.

## Investigation model

### Feed swipe path

Trace the shared path used by the feed pages:

`touch scroll -> CustomScrollView -> SliverMasonryGrid -> cached child build/layout -> IllustCard -> PixivImage -> image provider/ResizeImage -> Flutter image cache -> raster/clip`

The audit must inspect:

1. `lib/app/widgets/feed/feed_grid.dart`: `kFeedCacheExtent`, masonry layout, item extent propagation, and all callers.
2. `lib/app/widgets/feed/illust_card.dart`: Hero/clip structure, tap-time preloads, and decode width handoff.
3. `lib/app/pixiv_image.dart`: `decodeWidthFor`, `memCacheWidth`, fade behavior, tier upgrades, `_recordWhenDecoded`, transition history, and cache bookkeeping.
4. Every feed caller that passes `ScrollCacheExtent.pixels(kFeedCacheExtent)`.
5. `lib/main.dart` image-cache sizing and Android-only startup branches.
6. `lib/app/widgets/smooth_wheel_scroll.dart`: prove whether Android uses the desktop wheel animation path or the transparent pass-through path.

The primary causal question is whether high-DPR decoded pixel area and a broad masonry cache cause decode/layout/raster work to arrive during a finger fling. The report must compare the current worktree implementation with `git show HEAD:<file>` where the behavior differs.

### Hero path

Trace:

`IllustCard Hero -> route CustomTransitionPage/SlideTransition -> illustHeroFlightShuttleBuilder -> HeroRectClip -> endpoint image/clip`

Inspect per-frame work in `lib/app/motion/hero_transition.dart` and `hero_rect_clip.dart`, including endpoint measurement, render-tree walks, dynamic clip rectangles, nested rounded clips, `RepaintBoundary`, and any fallback that rebuilds sliver header content. Distinguish first-frame setup from work repeated on every animation tick. Check whether a destination route animation runs concurrently with the Hero overlay.

### Platform comparison

Use pinned source trees/commits, not current branch heads, for these comparisons:

| Client | Platform | Comparison focus |
| --- | --- | --- |
| `niuhuan/pansy` | Flutter/Rust | waterfall feed and custom image provider; establish that simpler code does not imply target-size decode |
| `yangyuehan058/pixview` | Flutter/Android | `CachedNetworkImage`, masonry cards, and absence/presence of explicit decode sizing |
| `mikaelzero/Piko` | Flutter | grid/keep-alive and animated placeholder cost |
| `mixelka75/pixiv-mix-app` | Compose/Kotlin | visible-window prefetch, platform-specific high-resolution radius, stable lazy grid/list, bounded Coil memory cache |
| `ultranity/Pix-EzViewer` | native Android | Glide `override(cellWidth)`, fixed RecyclerView sizing, and pausing image requests while scrolling |

The comparison is design evidence only. It cannot prove that another client reaches 120 Hz on the same hardware or workload.

## Risk ranking to validate in the report

1. **P0 candidate: decode-size regression/ambiguity.** The worktree `PixivImage.decodeWidthFor` should be compared with `HEAD`; the current implementation/comment must be checked for a physical-width cap. Android's commonly higher DPR can multiply decoded pixel area, while Windows often uses a lower DPR. This is a conditional platform explanation, not a device measurement.
2. **P0 candidate: broad cache prebuild.** `kFeedCacheExtent = 800` is applied across shared feeds and feeds a variable-height `SliverMasonryGrid`; entering the cache can build/layout more cards and enqueue more image work during a fling. The report must account for child delegate repaint boundaries and avoid claiming that cache extent alone proves jank.
3. **P1 candidate: decoded-image retention.** A fixed 256 MiB global Flutter image cache may reduce re-decode but can increase Android memory pressure when combined with high-DPR bitmaps. Compare with the platform-aware cache policy in PixMix and mark the memory/GC effect runtime-required.
4. **P1 candidate: Hero/routing composition.** A Hero shuttle, dynamic `HeroRectClip`, nested rounded clips, and a concurrent route slide can increase per-frame compositing. Render-tree measurement and header fallback work should be separated into setup versus tick cost.
5. **P1/P2 candidate: bookkeeping/retention.** `_recordWhenDecoded`, transition history, `_completedDecodes`, and Offstage-loaded recommendation tabs add work or retain state. The report must call these secondary risks unless code evidence shows they execute on the hot path at scale.

Explicit non-candidates: Android's confirmed high-refresh setting, `SmoothWheelScroll`'s non-desktop pass-through branch, and startup-only `PipelineWarmup` do not explain recurring finger-scroll jank by themselves.

## Verification and limitations

Run only deterministic, non-device checks available in the repository: `flutter analyze --no-pub`, focused existing tests for Hero/scroll/image behavior, and `git diff --check`. Record the empty ADB/device state only as a constraint if useful; do not attempt to manufacture a runtime trace. Any recommendation involving pause/resume, cache tuning, or Hero simplification belongs in a later approved implementation task.

## Deliverables

- `research/static-audit.md`: evidence-backed findings and comparison matrix.
- `prd.md`: scope, constraints, acceptance criteria.
- `implement.md`: reproducible audit checklist and quality gates.

No product source file is owned by this task.
