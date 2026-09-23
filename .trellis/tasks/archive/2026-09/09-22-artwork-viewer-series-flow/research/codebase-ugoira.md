# 代码调研：Ugoira 管线 — `lib/features/illust/detail/ugoira_viewer.dart` + `lib/core/ugoira/*`

对应设计 §4.4：保留媒体特有播放能力；保存能力与“准备中/导出中/失败”状态放进一致入口。行号已核对。

## 1. 组件定位

- 类注释 L29-30：**inline beta56 兼容的 Ugoira 呈现面**——封面、播放按钮、暂停遮罩留在详情页原位；ZIP/解码/导出资源全部由该 widget 持有并随 dispose 释放。
- `onLongPress`（L40/79）透传给内部 `GestureDetector`（L195）——详情页把它接到 `_toggleDownloadMode`，与静态图长按同语义。

## 2. 内部资源与生命周期

- 字段：`_asset`（L100，`UgoiraAsset`）、`_cache`（L101，`UgoiraFrameCache<ui.Image>`）、`_scheduler`（L102，`UgoiraScheduler`）、`_exportJob`（L113，`UgoiraExportJob`）、`_playRequested`（L117）、`_disposed`（L121）。
- `_performLoad`（L369-）创建 `UgoiraFrameCache(dispose: (image) => image.dispose())`（L391-394）与 `UgoiraScheduler`（L396）。
- `_load`（L365-366）用 `_loadFuture` 去重，防止并发重复下载/解码。
- 播放门槛 `_startPlaybackIfReady`（L448-453）：需 `_playRequested`、未 dispose、scheduler 就绪。
- 可见性/生命周期：`VisibilityDetector`（L180）控制离屏暂停；`didChangeAppLifecycleState`（L160-162）在 resumed 时恢复——前后台/滚动离屏都不持续解码。
- dispose 链：注释与代码（L387-424 区间）按序取消 load、scheduler、cache（dispose 每张 ui.Image）、asset、export job。

## 3. 数据/解码管线（`lib/core/ugoira/`）

1. `UgoiraRepository.fetchMetadata` → `GET /v1/ugoira/metadata`，校验帧数/延迟合法性；
2. ZIP 经 `pixivMediaTransportProvider`（download_providers.dart L23）流式写入 app 私有临时文件；
3. `SafeZipIndex` 打开归档（防 zip-slip/恶意条目）；
4. `UgoiraAsset` 按帧读字节并解码；
5. `UgoiraFrameCache<ui.Image>`：`ugoira_cache.dart` L1-14 —— **字节预算 LRU，eviction/clear 一律回调 `dispose`**，L34-69 的 put/evict/clear 都走 `_dispose`，杜绝 GPU 图片泄漏；
6. `UgoiraScheduler` 驱动帧时序、播放/暂停、可见性挂起；
7. `UgoiraExportJob`（`ugoira_export.dart`）在 isolate 编码 GIF，进度经 snapshot 流回 UI；恢复记录由 `ugoiraRecoveryStoreProvider`（download_providers.dart L65，`DownloadRecoveryStore`）持久化。

## 4. 保存/导出入口现状

- 唯一用户面出口：overlay 上的 `IconButton(tooltip: ugoiraSaveGif, onPressed: _export)`（L230-243）。
- 进行中态：`snapshot.status == running || finalizing` 时把图标换成 22px `CircularProgressIndicator`（L234-241）；终态经 `showAppSnackBar` 报成功/取消/失败。
- `_export` 流程（summary 记录 + 代码结构核对）：必要时先 `_load`；要求可用登录态/下载提交上下文；创建 `UgoiraExportJob` 并监听进度。
- **缺口**：导出入口只在播放 overlay 内；下载模式（downloadMode）下 ugoira 复用同一 overlay 参数但导出动作与“批量下载页”语义不统一（ugoira 没有 page 概念，downloadAllPages 不适用）；导出失败无重试按钮级入口（只能再点同一图标）。

## 5. 与 §4.4 对齐点

- “准备中/导出中/失败”三态已在 `_exportJob.snapshot.status` 上具备，但目前只反映为 icon↔spinner 切换——需要与下载一致的可见状态文案/重试语义（设计 §5.2：状态可见可恢复）。
- Ugoira 在 `DetailImagePager` 中 `count=1`（detail_image_pager.dart L75），在窄屏是纵向 sliver 单块——**两种布局下导出/长按语义已一致**，可作为“对象稳定识别、呈现变体显式化”的样板。
- 查看器（ImageViewerPage）按 URL 渲染，ugoira 不会进查看器——若 W4 要给 ugoira 提供查看器级 chrome（缩放不适用），需在实现稿中显式排除或提供播放型变体。
