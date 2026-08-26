# 实现反向图片搜索

## Goal

保留 beta56 粉色入口和结果体验，同时安全处理应用内选图及 Android SEND image，使用当时可用且合规的反向搜图服务。

## Confirmed Facts

- beta56 使用 gallery pick + 全量 readAsBytes，并解析 SauceNAO HTML；该实现易耗内存、受第三方结构/限流影响。
- Android platform task 已提供 ACTION_SEND image/*、content URI grant 和 typed input。
- SearchCatalog 提供结果路由，IllustStore/Detail 可展示 Pixiv 命中。

## Dependencies

- 08-26-search-catalog 与 08-26-android-platform-parity 完成。
- 08-26-illust-detail-viewer 完成。

## Requirements

- R1: 支持 Search 首页选图和 Android SEND image 两个入口，统一为受控 ImageInput，不复制处理链。
- R2: 验证 MIME、可读权限、实际图片格式、尺寸/像素/文件大小上限；使用 stream/临时文件和必要的受限 resize，不长期保留原图。
- R3: 开始实现时核验可用服务、官方 API/ToS/限流/隐私和凭据需求；不得盲目恢复脆弱 HTML scraping 或把 API key 写入仓库。
- R4: 上传显示明确隐私提示、进度、取消、429/限流、网络/解析错误和重试；取消后关闭流并清理临时文件。
- R5: 结果按相似度排序、Pixiv ID 去重；Pixiv 命中通过 API补全 IllustStore 并进入 Detail，非 Pixiv source 使用安全外部 URL 打开。
- R6: 第三方响应和外部 URL严格解析/allowlist，禁止执行 HTML/JS 或自动打开未知 scheme。
- R7: 敏感图片不进入日志、缓存快照、task artifacts 或崩溃报告；生命周期结束按策略删除临时数据。

## Acceptance Criteria

- [ ] 应用选图和 SEND content URI 均进入同一 preview/upload/result flow；取消和权限丢失明确失败。
- [ ] 大图处理内存有界，尺寸/格式/大小限制可测试，临时文件在 success/failure/cancel 后清理。
- [ ] 服务成功结果排序/去重正确，Pixiv 命中进入共享 Detail 且 bookmark/entity 状态一致。
- [ ] 429、服务变更、malformed response、未知 URL 和网络/TLS 失败不崩溃、不显示空成功。
- [ ] 源码/日志/配置不含第三方私密 API key或用户图片 bytes；隐私提示可见。
- [ ] analyze、全量 test、debug build、app picker 与 Android SEND 真机验证通过；真实服务测试记录日期/限制。

## Out of Scope

- 自建反向图片搜索服务。
- 绕过第三方限流/验证码。
- 后台持久上传或图片云同步。

## Risks and Deferred Items

- 第三方服务可能要求 API key或禁止自动化；若无合规可用通道，本任务应标记 blocker并保留选择/错误 UX，不伪造结果。

## Source Anchors

- beta56 lib/pages/search/result/image/*、lib/pages/image_selector/image_selector.dart
- Android SEND adapter、SearchCatalog routes、IllustStore/Detail

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
