# PRD：交互一致性后续修复（roadmap 收口遗留项）

父任务：`09-22-ui-interaction-consistency`（W1–W10 全部已合入归档）。
本任务收口各 leaf check 复核与外部评审（基线 `main@84037e8`）遗留、经核实仍存在于
`main@463ff33` 的具体缺陷。不重开叶子、不改架构，每项都是窄修复 + 定向测试。

## 已核销（勿再修）

- tagInput 残文计入 `_isDirty`：已由 W10 `734f943` 修复。
- `_loadPrefs` 失败永久 loading：当前代码已有 try/catch + `_loadError` 重试。
- 收藏 sheet submitting 可关闭：W3 有意取舍（in-flight 结果经 store 上报），保持。
- W6-downloads（#84）、W10（#86）：已合入归档。

## 需求项

### A 收藏编辑弹层（bookmark_switch_button.dart）

- **A1 预填充丢标签（数据丢失，最高优先级）**：`awaitingPrefill` 期间
  restrict `SegmentedButton`（L393-407）与标签输入仍可交互；用户先改可见性 →
  `stillPristine`（L309）为假 → L312 整组预填充跳过 → 旧标签被空值提交。
  **修法**：`awaitingPrefill && !prefillFailed` 时禁掉整组编辑区（restrict、
  标签输入、建议 chips），不只 confirm。不引入逐字段 dirty 框架。
- **A2 键盘 viewInsets**：sheet 高度定死 `heightOf(context)*0.35~0.75`
  （L356-360），未按 viewInsets 收缩；先核 `showAppSheet` 是否已代处理，
  无则让可用高度随键盘收缩，保证输入框与确认键不被盖住。

### B 作者页与导航

- **B1 头部折叠中间态死区**（profile_header_delegate.dart）：
  `expandedDetailsFadeStart=0.55`、`expandedIdentityExitProgress=0.78`（L156-157）
  移除展开区，collapsed toolbar 仅 `isFullyCollapsed`（L422）才构建，
  `IgnorePointer`（L517）再封一层 → 78%~100% 区间返回/更多等关键操作不可达。
  **修法**：返回/更多等关键操作做贯穿折叠全程的常驻层；展开与收起只变身份布局。
  测试必须断言 60%/80%/95% 中间进度下关键操作可实际触发（不只两端态）。
- **B2 re-tap 全 tab 归零**（user_page.dart:549 一带）：共享
  `PrimaryScrollController.animateTo(0)` 遍历全部 attached positions，
  keep-alive 兄弟 tab 位置一起被清。
  **修法**：从活跃 tab 的 context 取 `Scrollable.of(ctx).position.animateTo(0)`
  只动当前 position。**不要**给每 tab 包独立 PrimaryScrollController
  （会脱离 NestedScrollView innerController，header 折叠会坏）。
  测试断言：当前 tab 回顶 **且** 其他 tab 位置不变，缺一不可。
- **B3 NavigationRail 绕过回顶通道**（home_page.dart:169）：
  `onDestinationSelected: widget.navigationShell.goBranch` 直连，不经
  BranchSlideStack 的 re-tap 分发。
  **修法**：底栏与侧栏走同一导航动作入口（同 index→re-tap、异 index→goBranch）。
  测试：窄屏/宽屏下「同目标重复选择、目标分支有二级页、目标列表已滚动」结果一致。
- **B4 作者页标签可读性**（profile_header_delegate.dart L917-952）：
  `clamp(0.55,1.0)` 只限 baseSize，外层 `FittedBox.scaleDown` 不受限继续缩。
  **修法**：与发现页同决策序——合理文案 → 足够空间 → 必要横滚 → 有限缩放兜底。
  测试：窄屏+长翻译+5 标签主页代表场景（现有用例仅中文 4 标签不溢出）。

### C 阅读器（novel_reader_stage.dart / reader_settings.dart）

- **C1 进度 % 口径统一**：footer `(_page+1)/_pageCount*100`（L192/364）vs
  sheet `preview/(pageCount-1)*100`（L462）。统一为 footer 口径；
  补 1/2/N 页边界用例。
- **C2 `_persistAnchor` 静默**：L183 `unawaited(save)` 无 catch。
  **修法**：catch + debugPrint，不上 snackbar（锚点失败非关键路径，避免刷屏）。
- **C3 进度存储 LRU 失效**（reader_settings.dart:186-192）：jsonDecode 的 Map
  覆盖已有 key 不重排，注释所称「重写移到末尾」不成立 → 常读旧条目被先淘汰。
  **修法**：写入前 `all.remove(key)` 再赋值；补淘汰回归测试（重读旧条目后
  新写入不淘汰它）。
- **C4 设置保存失败语义**（已定方向）：保留当前「先 setState 后 save、
  失败 snackbar」行为，但把 `novelSettingsSaveFailed` 文案改为明确表达
  「设置仅本次生效，未能保存」（四语言 arb 全改 + gen-l10n + lookup +
  dart format）。不做内存回滚（滑块连发会引入竞态跳变）。

### D 入场错峰（feed_entrance.dart）

- **D1 持续滚动新曝光卡不再吃 index 错峰**：`_staggerIndex=index.clamp(0,8)`、
  `listStaggerStep` 延迟烘进 controller 时长（L119-123/L232-240），与曝光/
  缓存无关，每张卡都付 0~240ms 零透明等待。
  **修法**：错峰只作用于首屏批（如首批 N 张或首次布局帧内的可见卡）；
  持续滚动新曝光卡直接入场。测试：已缓存内容慢速滚动，无装饰性空白等待。

### E 记账同步

- **E1**：父任务 `09-22-ui-interaction-consistency` 的「当前停止点」仍写
  W2/W4 实现中；W6 已勾选条目残留 store.add 撤销/seenId 旧措辞（#83 已改实现）。
  更新为实际状态——只改文档，不改代码语义。

## 验收

- 每项配定向测试；全量 `flutter test`、`flutter analyze --no-pub`、
  `dart format --set-exit-if-changed`、`git diff --check` 绿。
- 本机不可验项（真机折叠手感、键盘实际遮挡、设备 LRU 观感）在 PR body 标「未验证」。

## 边界

- 不重开 W1–W10 叶子；不新建动效组件/通道/常量集；不改网络与数据层语义；
  不做逐字段 draft 合并框架；不为「更现代」追加动效。
