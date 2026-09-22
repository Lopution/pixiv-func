# W8 codebase 调研：路由 / provider / 基础设施 / 诊断生命周期

> 基线：`main@8067b2d`。本文件覆盖 §4.8 迁移所需的共享 owner、路由结构、诊断探针生命周期与尚未确定的接缝。

## 1. 路由结构（`lib/app/navigation/routes.dart`）

- `/settings` 是第五个 home branch：`StatefulShellBranch` :1036-1048，`navigatorKey` :810-811，`restorationScopeId: 'settings'` :1043，分支 observer :819。
- 子路由全表 `_settingsSubRoutes` :626-755：
  - `account` :631、`theme` :636、`language` :641
  - `translate` :646 + `credentials/:provider` :650-661（`baidu` bool 由 path param 派生 :657）
  - `network` :664 + `probe` :668-672 + `advanced` :673-682
  - `frame-probe` :685-690（每个 build 都注册，入口门控在根页）
  - `browse` :692、`download` :697 + `destination` :700-706
  - `history` :709 + `view` :712-718（`HistoryPage` 挂设置分支下；`openHistory` :1390 直达 `/settings/history/view`）
  - `muted` :721、`backup` :726、`tasks` :731、`about` :736 + `licenses` :739-752
- 门面方法（feature 间唯一合法通道）：`openSettingsPage`（settings_helpers:53）、`openHistory`、`openWatchLater`、`openWatchlist`、`openLocalNovels`、`openMe`、`openLogin`、`openTagSearch`、`openUser`、`openIllust`。
- 迁移含义：新增/重排页面须改 `_settingsSubRoutes`；跨页摘要不影响路由。分支 observer 已就位，页面级 `RouteAware` 可用（如用于诊断页返回时刷新录制指示）。

## 2. Provider / 状态 owner 地图

| 关注点 | owner | 说明 |
|---|---|---|
| 设置聚合 | `settingsProvider` → `SettingsController`（`core/settings/settings_controller.dart:26-164`） | AsyncNotifier；`_writeTail` 串行队列；写失败抛 `SettingsWriteException` 且保留旧值；`replaceAll` :146 供备份导入 |
| 细粒度选择器 | 同文件 :178-367 | themeMode/previewQuality/viewQuality/detailQuality/networkMode/downloadDestination/downloadCaption/namingRule/localHistory/pixivHistory/translation/maxDownloadCount/dohEnabled/dohEndpoints/reverseImageEngine/searchFilters/imageMirrorAllowlist/imageMirror/echFrontHost/autoImageSourceWinner |
| 账号 | `accountStoreProvider`（`core/auth/account_store.dart`） | `switchAccount`/`removeAccount`/`reload`；`AccountStatus.failure` 分支 |
| 服务端显示偏好 | `serverDisplaySettingsProvider`（`core/settings/server_display_settings.dart:126,180`） | 乐观更新+失败回滚（页面注释 :176-178） |
| 屏蔽 | `muteStoreProvider`（`core/mute/mute_store.dart:29-`） | `toggleTag/toggleUser/toggleWork`，per-key `pending` 集合 :130-194，`ensureHydrated` |
| 网络策略 | `networkAccessPolicyProvider`/`pixivNetworkFactoryProvider`（`core/network/compat/network_providers.dart`） | `effectiveRouteSnapshot()`、`clientFor(purpose, route, host)`、`revision`、`mode` |
| 凭据 | `translationCredentialStoreProvider`（`core/comments/translation_credentials.dart:75-81`） | read/write/delete × baidu/llm；**无 configured 状态查询口**——根页/翻译页摘要若要"已配置"需新增轻量读法（建议 `hasBaidu/hasLlm` 或 read 后判空，不读 secret） |
| 备份 | `backupServiceProvider`（`core/backup/backup_service.dart:244-257`）、`backupFilePickerProvider`（页面内 :14-23，可注入） | `export()` 返回 `({uri,fileName})?`；`apply(envelope, strategy)` 需可用账号（:142-147 抛 accountRequired） |
| 更新 | `updateServiceProvider`（`core/updater/update_providers.dart:30`） | FutureProvider；`UpdateService.capability()/check()/apply()/cancel()` |
| 开发者解锁 | `developerOptionsProvider`（`core/settings/shared_preferences.dart:18-`） | Notifier<bool>；about 7-tap `unlock()` 持久化 |
| SAF | `safTreePickerProvider`/`safDocumentSinkFactoryProvider`（`core/platform/saf_tree.dart:191-`） | `pickTree()` 只回 URI 字符串；**无 displayName 查询 API**——"人类可读名"需 URI 解码（`Uri.parse(treeUri).pathSegments.last` 解 `%3A`/`%2F`，如 `primary:Download/pixiv`）或新增平台通道 |
| 下载 | `downloadManagerProvider`（`core/download/download_providers.dart`） | 单例 manager，`tasks`/`groups`/`changes` 流；W6 拥有任务页 UI |
| 帧探针 | `FrameProbe.instance`（`core/debug/frame_probe.dart:17-105`） | 全局单例：start 挂 `addTimingsCallback`、stop 摘回调、report() 生成百分位报告；**生命周期本就独立于页面**，只是 `FrameProbePage.dispose` 主动 stop |

## 3. 弹层 / 断点 / 反馈通道现状

- 弹层：settings 内 dialog 全部经 `showAppDialog`（settings_page:213、account:151、backup:99、about:347）；**无 `showAppBottomSheet` 使用、无裸 showDialog/showModalBottomSheet**。W8 的弹层迁移=把"确认层级/语义顺序"按 §5.3/§5.4 对齐，不是入口替换。
- 断点：`AppBreakpoints`（`lib/app/layout/app_breakpoints.dart:5-28`）compact=medium=600、expanded=1200；`useNavigationRail`/`useExtendedRail`/`useTwoPaneDetail`。设置 feature 零引用——宽度接入从 0 开始。
- 限宽先例：`welcome_page.dart:26-28` `ConstrainedBox(maxWidth:520)`（W9 文件）；**form/settings 限宽无现存常量**——§5.5 说"内容最大宽度按任务角色确定"，本包须给设置选栏宽并声明（候选 560/600/640/720；见 implementation-draft）。
- 反馈：`showAppSnackBar`（`app/widgets/app_snack_bar.dart`）唯一出口，settings 已全部使用，无平行通道。
- 触觉：§5.6 owner 归 W4（下载/保存首个消费者）；W8 若需（如危险确认）只能消费已合入 owner——**依赖检查点：W4 可能未先合入**，本包不应自造触觉封装。
- `material_ui: ^1.2.0`（pubspec:40）遮蔽 material 组件；测试注意见其 spec 条目。

## 4. 诊断/探针生命周期详录

### 4.1 网络分层探测 `network_probe_page.dart`

- 状态：`_finished`/`_errors`/`_running` 全在 page State（:81-83）；**离开页面结果即丢**（无持久化——spec 亦要求"probe results are never saved"，app_settings.dart:115 注释）。
- 并发：`Future.wait` 全 host 同时跑（:111-119），单 host 失败入 `_errors` 不阻断。
- 报告模型已结构化（`network_probe.dart`）：`NetworkProbeReport{host,purpose,steps,conclusion,firstError,dnsDisagrees,totalDuration,environment}` :610-651；`toCopyableText()` :654-674 输出 env header+结论+逐层行。
- 结论枚举 8 值 :547-571（allReachable/dnsPolluted/sniBlocked/echAvailable/noSniAvailable/ipBlackholed/appLayer/inconclusive）——"先摘要后细节"所需数据已齐备，纯呈现层改造。
- 现状呈现顺序：run 按钮→每 host card(host+badge+DNS 提示+全部 step 行+copy)。目标：总体结论/建议动作在上（如"建议模式：ECH"/"全部可达"），per-host 明细折叠/次级。
- 无取消：probe future 不支持 cancel（`NetworkProbe.run` 无 cancelSignal 参数）；running 中离开页面 future 继续至完成（mounted 守卫丢弃 setState）——可接受的现状，或登记为限制。

### 4.2 帧探针 `frame_probe_page.dart` + `core/debug/frame_probe.dart`

- `FrameProbe.instance` 单例；`start()` 清旧帧并挂 `addTimingsCallback`（:28-33），`stop()` 摘回调（:35-39），`_frames` 保留→`report()` 可随时生成（:50-92）。
- **缺陷定位**：`_FrameProbePageState.dispose` :27-31 调 `stop()`——页面是录制生命周期的实际 owner，离开即断；注释自述用途"record timings while scrolling a feed"（:10-12）却离不开控制页。
- 分离方案方向（不改变 core 语义）：页面 dispose 不 stop；页面上显示"录制中 N frames"持续态；重进页面时 `_recording` 仍 true、计数继续；report 在 stop 后由 `report()` 生成。可选增强：录制中在设置根页 tile/全局有可见指示（SnackBar/角标），或 stop 动作留在原页 + "正在录制"横幅。安全约束：`release` build 中实例存在但入口隐藏；离开忘关→回调常驻开销小（只 append list），但应在 spec/PRD 写明"录制不自动停"语义与内存增长边界（`_frames` 无上限——长录制会积累 FrameTiming 对象，可考虑 cap 或文档化）。
- `FrameProbe` 无帧数上限：`_frames` List 无界（:22）——若改为离开继续录，需评估上限（建议 PRD 中注明，实现时如加 cap 属 core 改动）。

### 4.3 其他诊断面

- `_EffectiveRoutesSection`（network_settings:370-449）：手动 refresh，非响应式（注释 :367-369 自述）——属"诊断快照"语义，保留手动刷新即可。
- `_ThirdPartyReachabilitySection`（:454-565）：**initState 自动发 3 个 HEAD**（:481 `_check()`）——进入设置页即产生网络流量；可考虑改为按需（点开/手动），或保留自动但明示；action 语义应有 idle/checking/done。
- About 显示刷新率 tile（:109-121）：只读信息，无动作。
- About 导出日志（:99-103, :142-154）：action 语义完整（无日志→snackbar）。
- 更新检查（:123-135, :156-385）：完整 idle/checking/applying/结果状态机；失败原因归并是错配点（见 codebase-settings-pages.md §16）。

## 5. 设置项"真实消费者"核对（quality-guidelines 硬性规则）

每个展示的设置项须有页面外消费者。抽查结果：

- enableHistory→`localHistoryEnabledProvider`（HistoryTracker/Repository 消费 ✓）；enablePixivHistory→`pixivHistoryEnabledProvider` ✓
- hideMuted/blockR18/blockAI→feed 过滤消费 ✓（IllustCard/feed_grid import settings_controller）
- reduceMotion→`MotionTokens.enabled` 链 ✓；previewQuality/viewQuality/detailQuality→PixivImage/viewer ✓
- networkMode/dohEndpoints/echFrontHost→network_providers :25-26 ✓；imageSource→imageMirrorProvider :348 ✓
- maxDownloadCount→downloadManager scheduler cap ✓；namingRule/downloadDestination/downloadCaption→下载管线 ✓
- serverDisplay showAi/restrictedMode→服务端账号偏好 ✓
- 无孤儿设置项发现。

## 6. 术语/l10n 接缝

- `l10nLookup(context.l10n, key)`（`lib/l10n/lookup.dart`）允许字符串键访问——settingsText/_networkText/_probeText 三个等价包装并存（settings_helpers:13、network_settings:18、network_probe:25）——收敛点（不强制）。
- 新增 settings-specific 词项需同步 4 份 arb（zh/en/ja/ru）+ 重新生成 localizations——l10n 生成工作流属仓库既有流程；术语补充条目见 implementation-draft §10。
- `settings_test.dart:522` "all settings labels are available in all supported languages"——新增 key 必须四语齐全否则此测试失败（正向保障）。
