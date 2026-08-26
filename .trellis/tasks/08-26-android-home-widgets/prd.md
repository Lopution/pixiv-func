# 复刻 Android Home Widgets

## Goal

在不复制明文凭据的前提下，复刻beta56推荐/刷新Android Home Widgets及后台更新、点击导航。

## Confirmed Facts

- beta56有RecommendAppWidget、RefreshAppWidget和WorkManager，每30分钟更新推荐图片。
- 旧native ApiClient读取Flutter SharedPreferences中的账号/设置；新CredentialStore秘密不可复制到普通prefs。
- Widgets在app进程未运行时仍需明确的无账号、锁定、token失效和网络失败状态。

## Dependencies

- 08-26-recommended-feed-paging、secure-account-store、pixiv-network-token-refresh与android-platform-parity完成。

## Requirements

- R1: 从beta56提取Widget尺寸、圆角、图片、progress、refresh按钮、更新周期和click行为；不新增复杂交互。
- R2: 设计后台认证方案，复用Android Keystore保护的current account credential或受控headless Flutter，不把token/cookie写普通SharedPreferences/RemoteViews。
- R3: Recommend Widget加载真实推荐封面，Refresh Widget/按钮触发唯一受限工作；点击作品通过pixivfunc deep link进入Detail。
- R4: WorkManager使用unique work、网络约束、系统调度和有界重试，不启动常驻timer/service。
- R5: 多Widget IDs、resize、删除最后Widget、app update/reboot和账号切换正确注册/取消/刷新。
- R6: 无账号、设备credential不可用、reauth、R18/AI block、网络失败显示安全明确状态，不保留上一账号敏感图。
- R7: 图片下载遵循strict TLS/allowlist与有界bitmap尺寸，缓存/临时文件按Widget lifecycle清理。
- R8: receiver/provider exported、PendingIntent mutability和intent参数符合当前Android安全要求。

## Acceptance Criteria

- [ ] 添加/resize/refresh/remove Widgets在API36真机工作，样式/点击符合beta56。
- [ ] 后台更新使用unique constrained work，无重复worker或每秒timer；移除最后Widget取消工作。
- [ ] 无账号/账号切换/reauth/锁定时不泄露旧账号图片或credential并提示打开app。
- [ ] 点击推荐安全进入正确Detail，恶意PendingIntent/deep link参数被拒绝。
- [ ] 普通prefs/log/RemoteViews不含token/cookie，网络/bitmap内存有界。
- [ ] analyze、全量test、各Android test/debug build和reboot/background真机验证通过。

## Out of Scope

- iOS widgets。
- 在Widget中直接收藏/登录。
- 常驻后台服务。

## Risks and Deferred Items

- flutter_secure_storage与native/headless访问兼容性需实测；无法安全获取凭据时Widget应要求打开app刷新，不能降级明文。

## Source Anchors

- beta56 android/appwidget/*、res/layout/recommend_app_widget.xml、refresh_app_widget.xml
- AccountStore/Network/Recommended/DeepLink contracts

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
