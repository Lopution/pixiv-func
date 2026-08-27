# 复刻 Novel 阅读器

## Goal

在第一条插画链完成后，用当前 Novel API 重建 beta56 的水平分页阅读体验，并在旋转/字体变化后稳定恢复阅读位置。

## Confirmed Facts

- beta56 有 novel page、previewer 和 viewer，但正文依赖旧 HTML/布局方式。
- 父 PRD明确要求优先 v2/novel/detail，水平 PageView、左右各 30% 点击翻页、底部阅读百分比。
- 该功能按产品顺序在 Login→Recommended→Detail→Bookmark gate 之后开始。

## Dependencies

- 08-26-bookmark-state-sync milestone gate 通过。
- 08-26-pixiv-network-token-refresh 完成；User/Profile 基础可后续增强作者入口。

## Requirements

- R1: 开始实现时核验当前 v2/novel/detail 与相关 series/marker endpoint；禁止恢复旧 HTML scraping 作为默认。
- R2: 定义 NovelEntity、正文 block/paragraph 模型和 mapper，保留标题、作者、series、tags、caption、字数及可选 inline 标记。
- R3: Reader 使用水平 PageView；左侧 30% 点击 previous、右侧 30% 点击 next，中间区域不误翻页，底部显示阅读百分比。
- R4: 布局按 viewport、字体、字号、行距和正文版本缓存；后台/旋转/字体缩放后通过 stable text anchor 恢复接近原位置。
- R5: 分页计算可取消且不阻塞 UI；长文本、空段落、emoji/ruby/换行和不可识别标记有明确降级。
- R6: 阅读设置与全局 theme/language 协调，Dark/Light/System 不造成不可读文本或页码跳变。
- R7: 错误、删除/受限、网络重试、series 上下篇和历史/marker 集成边界可观察；不伪造保存/分享。

## Acceptance Criteria

- [ ] 真实 Novel detail 可加载并水平阅读，左右 30% 点击区域和滑动分页均符合行为。
- [ ] 底部百分比单调且首尾正确；旋转、字体缩放和主题切换后 stable anchor 保持在同一段附近。
- [ ] 布局缓存 key 正确失效，长文本分页在测试阈值内完成且不会持续阻塞主 isolate。
- [ ] API/解析/删除/受限错误明确，未知 markup 不崩溃且有可观察降级。
- [ ] 不包含 HTML scraping 默认路径，也不显示未实现的 save/share 成功。
- [ ] 单元、Widget、性能、analyze、全量 test、debug build 和真机阅读验证通过。

## Out of Scope

- Novel save/share。
- 发布 Novel。
- 完整 Search Novel 首页。
- Comments；除非原版 Novel detail 明确共用基础评论入口。

## Risks and Deferred Items

- 当前 API schema 与富文本标记可能变化；正文 mapper 和布局 cache 必须版本化，不能把解析失败静默成空正文。

## Source Anchors

- beta56 lib/pages/novel/novel.dart、controller.dart、lib/components/novel_viewer/*、novel_previewer
- 父 PRD R8 与当前 Pixiv API research

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
