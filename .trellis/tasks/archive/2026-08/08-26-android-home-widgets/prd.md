# 复刻 Android Home Widgets

## Goal

在不复制明文凭据的前提下，复刻beta56推荐/刷新Android Home Widgets及后台更新、点击导航。

## Confirmed Facts

- beta56有RecommendAppWidget、RefreshAppWidget和WorkManager，每30分钟更新推荐图片。
- 旧native ApiClient读取Flutter SharedPreferences中的账号/设置；新CredentialStore秘密不可复制到普通prefs。
- Widgets在app进程未运行时仍需明确的无账号、锁定、token失效和网络失败状态。

## Dependencies

- 08-26-recommended-feed-paging、secure-account-store、pixiv-network-token-refresh、android-platform-parity 与 P0 `08-26-restricted-compat-network` 完成。

## Requirements

- R1: 从beta56提取Widget尺寸、圆角、图片、progress、refresh按钮、更新周期和click行为；不新增复杂交互。
- R2: 优先由 Flutter 生成 versioned、account-revision keyed 的无秘密WidgetSnapshot，只含作品ID/标题/作者/受控图片引用/生成时间；native不得读取Account/Credential数据库或复制token/cookie到SharedPreferences、WorkData、RemoteViews。
- R3: Recommend Widget加载真实推荐封面，Refresh Widget/按钮触发唯一受限工作；点击作品通过pixivfunc deep link进入Detail。
- R4: 若beta56周期网络刷新需要app进程外认证，只能在实测证明可用时由受控headless Flutter复用同一AccountStore/PixivHttpClient/TokenRefreshGate；不得另写native refresh栈。WorkManager使用unique work、网络约束、系统调度和有界重试，不启动常驻timer/service。
- R4a: headless Flutter 只有在可复用 shared `NetworkAccessPolicy` 并通过系统 proxy/VPN off 的 API 36 冷启网络验证后，才能使用大陆兼容 route；不能在 native worker 内另建 DoH、固定 IP 或代理。
- R5: 多Widget IDs、resize、删除最后Widget、app update/reboot和账号切换正确注册/取消/刷新。
- R6: account invalid/switch/reauth 必须立即清除旧账号snapshot；瞬时网络或单张图片失败可保留同账号last-good并安排有界重试，二者不得混为同一fallback。
- R7: 图片下载遵循strict TLS/allowlist，按Widget options计算尺寸并同时限制单图pixels、总bitmap bytes和RemoteViews IPC预算；缓存/临时文件按Widget lifecycle清理。
- R8: receiver/provider exported、PendingIntent mutability和intent参数符合当前Android安全要求。
- R9: one-shot/periodic work 名称包含Widget family/account revision并采用明确KEEP/UPDATE/REPLACE策略；resize事件去抖，最后一个Widget删除时取消工作。每个slot的PendingIntent使用唯一data identity，避免跨Widget实例串作品。

## Acceptance Criteria

- [ ] 添加/resize/refresh/remove Widgets在API36真机工作，样式/点击符合beta56。
- [ ] 后台更新使用unique constrained work，无重复worker或每秒timer；移除最后Widget取消工作。
- [ ] 无账号/账号切换/reauth/锁定时不泄露旧账号图片或credential并提示打开app。
- [ ] 点击推荐安全进入正确Detail，恶意PendingIntent/deep link参数被拒绝。
- [ ] 普通prefs/log/RemoteViews不含token/cookie，网络/bitmap内存有界。
- [ ] WidgetSnapshot schema/version/age/account revision、corrupt/oversize和原子替换测试通过；account switch清旧图，瞬时失败保留同账号last-good。
- [ ] RemoteViews在最大支持尺寸下不超过定义的pixel/IPC budget，resize风暴不反复取消在途worker，slot PendingIntent不碰撞。
- [ ] Widget 后台刷新消费与前台相同的 direct-first/route 记忆；账号变化不会复用旧 candidate 或把 last-good 推给另一账号。
- [ ] analyze、全量test、各Android test/debug build和reboot/background真机验证通过。

## Out of Scope

- iOS widgets。
- 在Widget中直接收藏/登录。
- 常驻后台服务。

## Risks and Deferred Items

- headless Flutter、secure storage plugin和WorkManager在进程冷启/OEM后台限制下需实测；无法复用同一认证栈时，周期网络刷新成为blocker或需产品批准改为“打开app刷新”，不能降级明文或默默减少beta56行为。

## Source Anchors

- beta56 android/appwidget/*、res/layout/recommend_app_widget.xml、refresh_app_widget.xml
- AccountStore/Network/Recommended/DeepLink contracts

## Open Questions

当前没有阻塞性产品决策。若 API36/OEM 实测证明无法在不复制credential栈的情况下保持beta56周期后台刷新，必须回到planning确认“打开app刷新”的可见差异或保持blocker，不能自行选择降级。
