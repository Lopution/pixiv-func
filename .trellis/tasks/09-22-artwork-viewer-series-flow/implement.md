# 执行计划：作品浏览、查看器与系列流程（W4）

需求见 `prd.md`，技术设计见 `design.md`，逐项精确改动方案见
`research/implementation-draft.md`（行号已对 `main@8067b2d` 核实；
research 与 HEAD 的出入以 HEAD 为准并在实现时回注）。

默认单分支 `task/09-22-artwork-viewer-series-flow`，四阶段串行提交。
若 PR 超可审查规模按 design.md §一 拆 `…-s1`（阶段 1+2）/ `…-s2`
（阶段 3）/ `…-s3`（阶段 4）串行合入——三阶段都碰
`illust_detail_page.dart` 与 arb，不可并行。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

不要手改 `android/local.properties`。新增 l10n key 流程：编辑
`lib/l10n/app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` →
`python3 tool/gen_l10n_lookup.py`。**生成物冲突永不手解**：rebase 后
arb 按并集保留双方 key，重新跑 gen-l10n 与 gen_l10n_lookup.py（多叶
并行改 arb 是已知冲突区，见 design.md §六）。

验证命令（每阶段收尾都要跑）：

```bash
flutter analyze --no-pub
flutter test
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-artwork-viewer-series-flow
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：
> 看似不相关的测试文件抛 `TimeoutException`、单文件重跑即绿——先按
> spec 判定噪声再排查。

本机不可验项（真机触觉、predictive back 手势、TalkBack/Narrator、
`immersiveSticky` 真机表现、桌面滚轮手感、1.3x 大字体溢出）在 PR body
标「未验证」，不得由 widget test 推断通过。

## 阶段 1：AppHaptics + 设置开关 + 首个消费者接入

- [ ] 新建 `lib/app/haptics/app_haptics.dart`：`abstract final class
      AppHaptics`——`configure({required bool Function() isEnabled})`、
      `selectionClick()`（50ms 节流）、`heavyImpact()`（120ms 节流）、
      `try/catch` 吞平台异常、`@visibleForTesting` 触发观察器/重置钩子；
      头注释写明「唯一触觉入口，禁止直连 `HapticFeedback`」。
      测试：`test/app_haptics_test.dart`——开关关→不触发；节流窗口内
      丢弃；观察器断言级别序列；平台异常路径不炸。
      提交：`feat(app): 新增 AppHaptics 唯一触觉薄封装（selectionClick/heavyImpact）`
- [ ] `lib/core/settings/app_settings.dart` 增 `enableHaptics = true`
      （`_bool` 读取 + toJson + copyWith）；`settings_controller.dart`
      增 `setHapticsEnabled`。
      测试：settings 序列化 round-trip + 缺省值回读 true。
      提交：`feat(settings): 新增 enableHaptics 设置字段（默认开）`
- [ ] `browse_settings_page.dart` 增一行 `SettingsControl`（接
      `setHapticsEnabled`，走 `persistSettings`）；新 l10n key
      `enableHaptics`/`enableHapticsHint` 四语 + 重新生成 lookup；
      `app.dart` `PixivFuncApp.build` 一行
      `AppHaptics.configure(isEnabled: () => ref.read(settingsProvider)
      .valueOrNull?.enableHaptics ?? true)`。
      测试：设置页开关切换持久化断言；`component-guidelines.md` 增补
      「触觉统一走 `AppHaptics`，禁止直连 `HapticFeedback`」条目同
      commit 落地。
      提交：`feat(settings): 浏览设置接入触觉开关并接线 AppHaptics`
- [ ] 首个消费者触点接入（与 snackbar/状态变更同点同分支）：
      `illust_detail_page.dart` `_toggleDownloadMode`（进入 heavyImpact/
      退出 selectionClick）与 AppBar `downloadAll` 成功/失败
      （heavyImpact）；`page_image.dart` `_DownloadBadge.onTap` 成功/失败
      （heavyImpact）；`illust_card_actions.dart` 下载动作成功/失败；
      `ugoira_viewer.dart` `_export` 完成/失败（heavyImpact）。
      测试：各触点经 debug 观察器断言级别；开关关时零调用且视觉反馈
      不变。
      提交：`feat(haptics): 下载/保存触点接入 AppHaptics 首消费者`

## 阶段 2：下载模式显式化 + tag 长按菜单

- [ ] `illust_detail_page.dart`：AppBar `downloadAll` 解除
      `_downloadMode` 门控改常显；新增 `Set<int> _selectedPages`；
      `bottomNavigationBar` 槽挂 `AnimatedSwitcher`（`MotionTokens.fast`）
      模式底栏「已选 n / 共 N · 全选 · 完成（n=0 禁用）· 取消」；
      完成 = 循环 `download(entity, i)` 提交所选页 → 成功 snackbar +
      heavyImpact 后退模式，失败保留模式与选中态。
      新 l10n key：`downloadSelectPages`/`downloadSelectedCount`/
      `selectAll`/`done`（四语）。
      提交：`feat(illust): 下载全部常显，批量页选择改为显式模式`
- [ ] `page_image.dart`：`DetailPageImage` 增 `selected`/`onToggleSelect`；
      下载模式内 `onTap` 从开查看器改为切选中（placeholder 页仍不可点）；
      角标优先级 spinner > 选中 check > 未选中空心圈；选中切换
      `selectionClick`。
      `detail_image_pager.dart` 双栏同语义接入。
      提交：`feat(illust): 页选择模式角标与点击语义`
- [ ] `info_block.dart`：`TagChip.onLongPress` 从 `_blockMode` 翻转改为
      `showAppBottomSheet` 菜单（搜索该 tag / 复制+snackbar+selectionClick /
      屏蔽·解除屏蔽 `muteStoreProvider.toggleTag` / 进入批量屏蔽模式）；
      菜单弹出 `heavyImpact`。`TagChip` 签名不动。
      新 l10n key：`tagActionSearch`/`tagActionCopy`/`tagActionMute`/
      `tagActionUnmute`/`tagActionMuteMode`/`tagCopied`（四语）。
      提交：`feat(illust): tag 长按改为直接动作菜单`
- [ ] `ugoira_viewer.dart`：导出钮脱离 `downloadMode && _asset != null`
      门控改常显；三态同一入口（准备中 disabled / 进度 spinner+tooltip /
      失败 error 图标点击重试）；详情页对 ugoira 作品的长按接线移除
      （无页可选）。
      新 l10n key：`ugoiraExporting`（带 `{percent}`，四语）。
      提交：`feat(illust): Ugoira 导出改常显入口并三态一致`
- [ ] 旧测试同 commit 改写：`illust_detail_page_test.dart` 下载模式组
      「点角标即下载」「Download All 门控」用例 → 选择模式链路断言；
      `ugoira_viewer_test.dart` long-press 交接用例 → 常显导出断言。
      提交：`test(illust): 改写固化旧下载模式/tag 长按语义的用例`

## 阶段 3：查看器改造（stage `-s2` 候选）

- [ ] `image_viewer_page.dart`：`StatefulWidget`→`ConsumerStatefulWidget`，
      构造增 `IllustEntity? entity`；`routes.dart` `_ImageViewerRoute` 透传
      已解析实体（不改 extra 形态，entity null 分支动作不渲染）。
      提交：`feat(viewer): ImageViewerPage 接入实体参数`
- [ ] chrome 显隐：`_chromeVisible` + 媒体区单击/底栏全屏钮/键盘 `F`
      三通道；隐藏 = AppBar+底栏+`SystemUiMode.immersiveSticky`；
      `MotionTokens.fast` 淡入淡出；隐藏态手势仍可用。
      新 l10n key：`viewerEnterFullscreen`/`viewerExitFullscreen`（四语）。
      测试：三通道切换断言 + 隐藏态手势可用断言。
      提交：`feat(viewer): chrome 可显隐与沉浸式全屏`
- [ ] 双击缩放循环 + 单击消歧：同一 `GestureDetector` 注册
      `onTap`+`onDoubleTap`（框架消歧，单击延迟 ~kDoubleTapTimeout）；
      双击 fit→2.5（双击点 focal）→fit，`Matrix4` tween +
      `MotionTokens.fast`；底栏 `fit_screen` 复位 + 键盘 `0`。
      新 l10n key：`viewerFitScreen`（四语）。
      测试：**单击-不触发-双击-再单击序列**（单击后等超时→chrome 切；
      双击→缩放且 chrome 不切）；focal 缩放矩阵断言。
      提交：`feat(viewer): 双击缩放循环与单击消歧`
- [ ] 底栏工具栏：左 `n/total`（InkWell → `showAppBottomSheet` 页码
      选择 → `jumpToPage`）；右 `fit_screen`/全屏/保存当前页/分享/信息。
      保存 = `download(entity, _activePage)` + `stateFor` 角标语义 +
      heavyImpact；分享 = `SharePayload.illust`；信息 = bottom sheet
      （标题/作者/日期/ID/页数 + 「保存全部」+「打开详情页」）。
      新 l10n key：`viewerSavePage`/`viewerJumpToPage`/`viewerInfo`/
      `viewerOpenDetail`（四语）。
      测试：页码 sheet 跳页、保存/分享/信息动作（entity null 时不渲染）、
      tooltip 存在性。
      提交：`feat(viewer): 底部工具栏与页码/保存/分享/信息动作`
- [ ] 空态修复：`_pageCount==0` → 计数位 `SizedBox.shrink`（移除
      `1 / 0`）；保存/分享/跳页/适应/全屏 visible-disabled；body
      `viewerNoImages` 占位保留。
      测试：空态无 `1 / 0` + 动作禁用断言；**旧 `find.text('1 / 0')`
      断言同 commit 改写**。
      提交：`fix(viewer): 空列表不再渲染误导性 1/0 页码`
- [ ] 键鼠等价：`CallbackShortcuts`（`←`/`→`/`J`/`K`/`+`/`-`/`0`/`F`/
      `Esc`/`Backspace`/`S`/`I`）+ `Focus(autofocus)` +
      `Listener(onPointerSignal)` 滚轮指针焦点缩放、Shift+滚轮翻页。
      测试：键盘映射逐键断言；滚轮缩放方向断言。
      提交：`feat(viewer): 键盘快捷键与鼠标滚轮等价`
- [ ] PopScope 三级优先级：`canPop = !_activeZoomed && _chromeVisible`；
      `!didPop` 分支 zoomed→复位、chrome 隐→恢复；显式 back、
      `DragToDismiss`、`Esc`/`Backspace` 保持命令式 `pop()`。
      测试：**放大→返回→复位不退；chrome 隐→返回→恢复不退；再次
      返回→退；replaceImageViewerPage 换页后（新 route/State）同一
      拦截仍生效**（risks R2，用真实 replace 复现）。
      提交：`feat(viewer): 系统返回先复位缩放/恢复 chrome 再退路由`

## 阶段 4：信息可达 + 系列语义（stage `-s3` 候选）

- [ ] `illust_detail_page.dart` 窄屏分支：AppBar 下固定 compact header
      条（标题 ellipsize + 作者 + `n/共N页` + 「信息」钮）；metaSlivers
      `InfoBlock` 外包 `GlobalKey` 容器；「信息」→
      `Scrollable.ensureVisible(key.currentContext,
      duration: MotionTokens.medium)`；双栏不渲染 header；entity null
      降级态不渲染。
      新 l10n key：`illustInfoJump`（四语）。
      测试：窄屏 header 渲染+点击滚动断言；≥1200 不渲染断言。
      提交：`feat(illust): 窄屏常驻信息条与信息区直达`
- [ ] `series_models.dart` `IllustSeriesEntity` 增 `firstContentId`
      （merge 保留旧值）；`series_repository.dart` `_parseSeriesWorksPage`
      解析 `illust_series_first_illust.id`；`series_store.dart` merge 检查。
      测试：解析单测（fixture 已含 `illust_series_first_illust`）+ merge
      保留断言。
      提交：`feat(series): 解析 illust_series_first_illust 为 firstContentId`
- [ ] 新建 `lib/core/series/series_recent_open_store.dart`：
      `NotifierProvider` 内存 Map（键 `'$accountId:manga:$seriesId'` →
      `(illustId, contentOrder)`）；`illust_series_section.dart` 在
      context 解析处 post-frame `record`。
      测试：record/read/多账号隔离/**不落盘**断言。
      提交：`feat(series): 会话内存态记录最近打开的系列作品`
- [ ] `illust_series_page.dart` `_SeriesHeader` 增动作行：「开始阅读」
      （`firstContentId != null` 才渲染 → `openIllust`）+「返回第 n 话」
      （memory 命中才渲染，contentOrder 缺失用兜底文案）；
      `markSeen` 不动。
      新 l10n key：`seriesStartReading`/`seriesBackToEpisode`/
      `seriesBackToLast`（四语）。
      测试：两动作渲染条件与导航断言；**markSeen 游标与返回目标用
      不同状态/断言**（W4 门禁）。
      提交：`feat(series): 系列页开始阅读与返回第 n 话诚实语义`

## 验证清单（实现期必须逐项跑/写）

- [ ] **单击/双击序列**：单击→等超时→chrome 切；双击→缩放、chrome 不动；
      双击后再单击→chrome 切（risks R1）。
- [ ] **PopScope × route-replace**：放大→滑动翻页（`replace` 重建 route）
      →返回应退路由（State 已重置）；同页内放大→返回→复位不退
      （risks R2）。
- [ ] **l10n arb 并行冲突**：与并行叶子的 arb/生成物冲突时，arb 并集
      保留双方 key → `flutter gen-l10n` → `gen_l10n_lookup.py`，生成物
      不手解（design.md §六）。
- [ ] **下载模式 chrome**：窄屏纵列/双栏 pager/横屏三形态呈现断言；
      全屏图片作品退出路径=取消钮（risks R4）。
- [ ] **降级态**：entity null/Restricted/Error 快照下 compact header、
      查看器动作、系列动作行的渲染规则逐一断言。
- [ ] **触觉双路径**：开关开→观察器收级别序列；开关关→零调用且
      snackbar/角标等视觉反馈不变。
- [ ] **运行时矩阵**（PR 标注真实结果或「未验证」）：320/390/600/840/
      1200dp + 手机横屏；默认与 1.3x 字号 + 中英长翻译；触摸/键盘/
      鼠标滚轮；TalkBack 或 Narrator 代表路径（查看器优先）；
      默认与 reduced motion；loading/empty/error/busy 各态。

## 收尾

- [ ] `flutter analyze --no-pub` 与全量 `flutter test` 通过（噪声按
      spec 判定）；`git diff --check` 干净；`task.py validate` 通过；
      `grep -rn "HapticFeedback" lib/` 仅命中 `app_haptics.dart`。
- [ ] PR：`gh pr create --fill`；CI 绿后 `gh pr merge --merge`。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随本 PR 的最后
      提交）。

## 边界（不做）

- 持久化章节进度 schema（PRD D4）；`WatchlistReadCursor` 语义变更；
  PixEz 式 tag 收藏 / pinch-zoom overlay；Ugoira 查看器变体；
  `TagChip`/`BookmarkSwitchButton` 签名变更；`PhotoView` 等新依赖；
  W2/W3/W5/W6/W7/W8/W9 owning files（含设置页表单结构迁移、novel 侧
  tag 行为、下载任务页 UI）；网络/持久化业务语义。
