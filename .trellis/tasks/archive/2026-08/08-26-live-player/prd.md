# 复刻 Live 播放器

## Goal

重新核验Pixiv Sketch Live当前可用链路，并在可用时复刻beta56播放器、清晰度、全屏和作者关注体验。

## Confirmed Facts

- beta56 Live为16:9，单击显示controls、双击play/pause、右上quality、底部elapsed/duration、全屏landscape、下方avatar/name/follow。
- 旧controller使用fixed-IP local live proxy和本地HLS转换；父PRD明确禁止恢复。
- 原始需求给出v1/live/list、Sketch detail/logs/websocket/HLS候选，但要求运行时重新验证。

## Dependencies

- 08-26-pixiv-network-token-refresh、user-profile-follow、android-platform-parity 与 P0 `08-26-restricted-compat-network` 完成。

## Requirements

- R1: 实现开始当天以真实请求核验Live list/detail/HLS/必要日志或WebSocket端点、认证和schema，记录版本/日期；不可用时明确blocker。
- R2: 不得恢复fixed-IP proxy、TLS bypass或非必要中间转发；HLS URL/redirect遵循严格host/TLS策略。
- R2a: Live 的 list/detail/HLS host 只有在当日 endpoint 证据通过并进入 shared `PixivDestinationRegistry` 后才能请求；播放器复用 `NetworkAccessPolicy`，不新增独立代理或 DNS/SNI fallback。
- R3: Live detail/list entity映射错误可观察；ended/private/region/auth/error状态明确。
- R4: 播放器默认16:9；single tap切换controls、double tap play/pause，手势去歧义且不误触owner区域。
- R5: controls右上quality，底部elapsed/duration与fullscreen；切quality尽量保留position/play state。
- R6: fullscreen进入landscape并在退出/路由dispose恢复原orientation/system UI；wakelock仅播放可见时持有。
- R7: 页面下方展示avatar、name和FollowStore按钮，点击进入共享User Profile。
- R8: 生命周期/音频焦点/耳机/后台/网络切换/缓冲/重试/结束释放player、socket和wakelock。
- R9: 不添加beta56没有的chat UI，也不发送Live bullet comments。
- R10: 采用两阶段 feasibility gate：在 endpoint/auth/schema/HLS 真实证据和 host policy 通过前，不添加 player/codec 依赖或产品页面实现；若外部链路不可用，只保存脱敏研究与 blocker，不留下 fixture 驱动的业务播放器。

## Acceptance Criteria

- [ ] 若endpoint可用，真实Live list→detail→HLS播放成功并记录受控证据；若不可用则任务保持blocker而非mock。
- [ ] 16:9、single/double tap、controls、quality、elapsed/duration和landscape fullscreen符合beta56。
- [ ] quality切换、buffer/error/retry/ended与background/foreground状态正确。
- [ ] 退出/结束后orientation/system UI/wakelock/player/socket全部恢复/释放。
- [ ] owner/follow与UserStore/FollowStore同步；无chat或fixed-IP路径。
- [ ] Live 的 endpoint/HLS/redirect route 与大陆 access policy 有独立记录；系统 proxy/VPN 关闭时失败分类可见，不能以 API/fixture 成功替代。
- [ ] dependency diff 证明 player 依赖只在 feasibility 通过后加入；mock/fixture 只能用于 mapper/controller 单测，不能计入真实 Live 可用验收。
- [ ] analyze、全量test、debug build和真实API36设备Live验证通过。

## Out of Scope

- Live chat/bullet comments。
- 录制/下载Live。
- fixed-IP/local proxy。

## Risks and Deferred Items

- Live服务可能已停用、需额外认证或禁止第三方播放；本次固定源码审查未找到可作为当前链路证据的完整开源实现，真实不可用即为外部blocker，但“未找到”本身也不等于服务已停用。

## Source Anchors

- beta56 lib/pages/live/live.dart、controller.dart、models/m3u8.dart、components/live_previewer
- 父PRD Live候选endpoint与current research

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
