# Startup, transition and i18n runtime fixes

## Goal

修复冷启动欢迎页闪屏与 StartupGate 重定向死锁、返回动画降帧（Android SFR 帧率投票）、Hero 返回重影、信息流预加载回归、详情页 viewer 闪烁及多语言布局溢出

## Requirements

- 已登录冷启动不得闪过 Welcome：从 `/splash` 起步，等 settings/account 水合后再路由；`StartupGate` 在重定向 in-flight 期间不得卸载 Router（`MaterialApp.builder` 的 child 即 Router 本体），`/user-agreement` 纳入登录流程白名单。
- 任意二级页面静置后返回须满帧：对 Flutter SurfaceView 投 `Surface.setFrameRate(面板最高刷新率)`（API 30+，surface 重建自动重投，建议性投票、系统可忽略）。
- Hero 返回不得重影：`SnapshotWidget` 捕获推迟一帧，等 HeroController postFrame 占位替换落地。
- 信息流恢复有界预加载：分层图片缓存（`image_tier_cache.dart`）在屏幕外做受控的档位准备，而非完全禁用。
- 详情页加载未完成时打开 viewer 不得闪烁/切错档位：viewer 路由取 store 现值实体，URL 切换走 gapless。
- i18n 布局对任意语言一致：引导页注释固定槽位、标题 FittedBox、底栏 label 与 profile tabs 按最宽者统一缩放；不做 per-locale 布局分支。

## Acceptance Criteria

- [x] 真机验证：开启 SFR 后静置 3s+ 返回 `interval/nativeVsync p50=8.3ms`（120Hz），关闭投票复现 16.7ms——因果链闭合
- [x] 全新安装：首次登录、登录页进入用户协议均不再卡 `_StartupProgress`（死锁回归测试 `settings_pop_repro_test.dart`）
- [x] Hero 往返单图飞行，无滑出页残影
- [x] `flutter analyze` 零 issue；782 测试全过；`git diff --check` 干净
- [x] 俄/英长文本下底栏不截断、profile tabs 不拥挤、引导页锚点一致（`func_bottom_nav_test.dart` 覆盖统一缩放）

## Notes

- 探针与浮动调试面板已全部移除；`MainActivity.kt` 仅保留 SFR 正式实现。
- 投票取 `Display.supportedModes` 最高刷新率自适应；代价为前台静置时屏幕维持高刷（用户已确认接受）。
