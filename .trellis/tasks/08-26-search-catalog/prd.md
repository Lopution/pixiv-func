# 复刻 Search 与结果页

## Goal

复刻 beta56 Search 首页、输入页和 Illust/Manga、Novel、User 三类结果，并用可取消 debounce 和共享实体状态实现稳定搜索。

## Confirmed Facts

- beta56 Search guide 顶部 fake search box、粉色 reverse image 按钮及两列 trending tags；tag tap 搜索，long press 打开代表作品。
- 输入页有 Illust & Manga、Novel、User tabs；数字在 Illust/User 路由对应 ID，文本走 keyword/autocomplete。
- 旧 autocomplete 使用延迟 Future 和 CancelToken，现代实现要求 cancellable debounce。

## Dependencies

- 08-26-bookmark-state-sync、08-26-novel-reader 与 08-26-user-profile-follow 完成。
- Reverse image 作为独立子任务后接入。

## Requirements

- R1: Search 首页保持 fake search box、粉色反向搜图入口和两列 trending tags；tag tap 搜索、long press 进入代表作品。
- R2: 输入页保持三个 tabs、焦点/取消按钮和原版视觉；Illust 数字输入走 illust ID，User 数字输入走 user ID，Novel 数字行为按 beta56/当前 API明确。
- R3: 实现 cancellable debounce：新输入取消旧 timer/request，旧响应不能覆盖新 query，页面 dispose 后不更新状态。
- R4: 分别实现 Illust/Manga、Novel、User keyword 结果，复用实体 stores、分页状态、错误/空/取消和 scroll/query 恢复。
- R5: 复刻 beta56 Illust/Novel filters 的可见字段和默认值；只发送当前 API 支持的 typed 参数，未知值不拼接。
- R6: 搜索历史/建议如原版存在则按参考实现；不得将 private query 写日志或跨账号泄漏。
- R7: tag/detail 最小搜索路由迁移到本任务共用实现，不产生重复页面。

## Acceptance Criteria

- [ ] Search guide、两列 trending、tap/long-press 和输入页三 tabs 与 beta56 行为一致。
- [ ] 数字/文本在三个 tabs 路由正确；autocomplete debounce/取消/旧响应测试通过。
- [ ] 三类结果真实加载、分页、刷新、错误/空和返回恢复，作品/Novel/User 进入共享页面。
- [ ] filters 默认值、序列化和 reset 与参考行为一致，恶意/未知参数不进入请求。
- [ ] 快速输入、切 tab、返回、账号切换和离线场景无旧结果闪回或状态串扰。
- [ ] analyze、全量 test、debug build、Widget 和真实 API 搜索验证通过。

## Out of Scope

- 反向图片服务上传/解析。
- 新增高级搜索语法。
- 保存原版没有的跨设备搜索历史。

## Risks and Deferred Items

- Search endpoint/filter 参数可能变化；typed mapper 与实时 research 必须明确不支持项，不能静默丢条件。

## Source Anchors

- beta56 lib/pages/search_guide/*、lib/pages/search/search.dart、controller.dart、result/*
- 父 PRD Search 行为与 shared stores/paging

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
