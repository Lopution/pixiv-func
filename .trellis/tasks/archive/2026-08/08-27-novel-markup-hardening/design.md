# Novel typed markup 与长文预算 — Design

## Typed representation

将 API/正文标记解析为不可变 token：`Text`、`NewPage`、`Chapter`、`Ruby(base, annotation)`、`Jump(target)`、`PixivImage(id)`、`UploadedImage(id)`、`Unknown(rawName, attributes, rawText)`。token 保留原始属性，解析器不执行 HTML、脚本或任意 URI。

## Pipeline and budget

`source -> scanner -> typed tokens -> bounded chunks -> layout pages -> generation-gated reader commit`。scanner 逐段消费输入并设置 token/count/UTF-16/bitmap budget；layout 每个 chunk 可 yield/cancel，进度和超限作为显式状态返回。图片 token 只产生受 allowlist 约束的 load request，不在 parser 中下载。

## Generation and errors

reader 章节/页面切换生成新的 `NovelLayoutContext`；parse/layout 完成后必须校验 generation、chapter id 和 lifecycle。未知或损坏标记保留为 fallback token 并记录诊断；非法 jump/image 标识返回 unsupported/invalid 状态，不导航任意 host。

## Test and rollback

fixtures 覆盖每个 token、Unicode、嵌套/损坏属性和超长正文；测试页边界、取消和迟到结果。新 parser 可由 feature flag/adapter 接入现有 reader，出现 beta56 差异时回退到旧渲染路径，但不删除未知 token 证据。
