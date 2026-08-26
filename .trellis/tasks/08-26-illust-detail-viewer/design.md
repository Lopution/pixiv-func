# 复刻作品详情与全屏查看器 — Design

## Objective

从推荐卡片进入真实作品详情，完整呈现单图/多页信息、下载模式和 beta56 全屏查看手势。

## Architecture and Boundaries

- IllustDetailRepository 通过 PixivHttpClient 获取并映射 detail；DetailController 以 ID watch IllustStore，并管理 fetch/download-mode/page state。
- Viewer 接收作品 ID + initial page，通过 store 读取 URL；ViewerController 管 PageController、每页 zoom state 和 lifecycle。
- TagActionService 提供 typed search/block command；最小 tag result 与后续 SearchCatalog 共用 repository contract。
- Download UI 只观察 DownloadManager task IDs；不在页面内下载或持有文件 bytes。

## Data Flow

card ID → detail route/controller → store snapshot + detail fetch/upsert → media carousel；tap → viewer(ID,index)；long press → download mode → typed requests → DownloadManager。

## Compatibility, Security, and Migration

- 后续 BookmarkStore、Comments、Ugoira 通过稳定 slot/route 接入，不改变详情基本布局。
- 原版可见交互保留，旧 extended_image/GetX 实现不迁移。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

