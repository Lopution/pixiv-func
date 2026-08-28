# 实现反向图片搜索 — Design

## Objective

保留 beta56 粉色入口和结果体验，同时安全处理应用内选图及 Android SEND image，使用当时可用且合规的反向搜图服务。

## Architecture and Boundaries

- ImageInputSource 封装 picker/content URI，ImagePreprocessor 流式验证并输出临时受限文件。
- ReverseImageProvider 是可替换接口并先返回 typed capability。`StructuredProvider` 解析受控响应；`InteractiveWebProvider` 只在研究证明必要且产品差异获批后，把 FileProvider URI 交给同源真实页面的 file chooser，不把 challenge HTML 下载后重放。
- ResultMapper 产生 typed PixivHit/ExternalHit；PixivHit 通过 IllustRepository 补全，ExternalHit 经过 URL policy。
- Controller 管 consent/preprocess/upload/results/error/cancel 状态和临时资源生命周期。

## Data Flow

picker or SEND URI → validate/consent/preprocess owned temp → capability gate → structured upload/parse，或经批准的 interactive WebView handoff → Pixiv detail hydration/safe external link → all-terminal cleanup。

## Compatibility, Security, and Migration

- 保留 beta56入口、结果卡片和相似度信息；服务内部可替换。
- 如果 provider 需要用户 key，必须另行取得产品/凭据决策，不将默认密钥内置。
- Interactive WebView 是外部能力边界，不等同于 beta56 卡片验收；challenge、用户取消、页面 origin 变化与 file chooser 重试都必须成为 typed terminal/state。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚已独立提交的叶子任务。
