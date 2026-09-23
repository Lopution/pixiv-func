# Check Report: 09-22-settings-form-semantics s1–s4

- 复核人：trellis-check sub-agent
- 基线：origin/main @ b31846c（detached HEAD，worktree `/root/Pixiv-func-checkw8`）
- 范围：PR #63(s1) / #69(s2) / #71(s3) / #72(s4) 合入的产品改动
- 方法：逐 merge diff 复核 + 对照任务 prd/design/implement 与父契约 §5.3/§5.5/§5.7 + analyze/test

## 进度日志（增量）

- [init] 报告创建，开始加载任务产物与 diff。
- [R1/s1] `settings_tile.dart:9,18,25` `subtitle` 可选槽落地，不传渲染不变——PASS。
- [R1/s1] `settings_helpers.dart:27-32` `settingsNarrowBody`=Center+ConstrainedBox(`ContentWidths.settings`=600)，读 `lib/app/layout/content_widths.dart` 而非 `AppBreakpoints.medium`，符合父 §5.5——PASS（implement.md 写"AppBreakpoints.medium"是文档措辞漂移，代码按 design.md §2 正确消费 ContentWidths）。
- [R1/s3] `persistSettings` 增 `failureMessageKey`（settings_helpers.dart:91-109）；`_persistNetwork` 已删（network 页三处直连 persistSettings）；`_ServerDisplaySection._write` 收敛为薄封装传 `serverDisplayWriteFailed`（account_settings_page.dart:223-233）——PASS。
- [R9] zh arb：`unmute*`="解除屏蔽*"、`imageSourceApplyAndTest`="应用并测试"（W1 未回改）、`saveLocationUseCustomAlbum` 全仓零残留、`取消屏蔽` 零残留、`profileEditLeaveConfirm`="放弃修改"（放弃≠取消）、`translateCredentialsClear`/`networkAdvancedReset` 保留——PASS。
- [R2/s1] `settings_page.dart` 根页全量摘要落地：账号名/主题/语言/翻译(provider+凭据态)/浏览图源/屏蔽计数/历史开关态/网络模式/下载预设+位置/任务数一次性快照(ref.read，D9)/备份静态 hint/版本 FutureBuilder；「我的内容」组新增「查看历史」→ openHistory('/settings/history/view')，浏览组历史 tile 仍 → /settings/history（D5）——PASS。
- [R3/s2] theme/language 用 `ListTile.selected` 属性（其内部即 Semantics(selected:)），测试 :2008 用 getSemantics 断言 isSelected——PASS（顺带得 selected 视觉态）。
- [R4/s2] browse 重排序=偏好→质量→外观(reduceMotion+haptics)→图源；custom 输入接 guardDraft；pixivHistory 开关已移除；「应用并测试」消费 W1 名——PASS。
- [s2] muted_items：unmute 图标 visibility_outlined+tooltip 解除屏蔽*、pending 行 spinner（MuteStore.pending 协议）、_controller.clear() 移到 toggle 成功后、_SectionHeader→SettingsSection——PASS。
- [s2] history 页：pixivHistory 唯一 owner；historyView 入口带开关态摘要——PASS。
- [s3] account：_switchingTo busy（目标行 spinner+全列 tap/remove 禁用+semanticsLabel 正在切换）、当前行 selected+check、_confirmRemove 保持——PASS。
- [s3] credentials：_clear 前 showAppDialog（对象+后果）、dirty guard、busy spinner 双按钮、清除同步清输入框+双色 _status——PASS。
- [s3] translate 页：凭据入口摘要 已配置/未配置（hasX()），且 push 返回时 setState(_refreshCredentials) 正确刷新——PASS。
- [s3] download_settings：Slider immediate-with-preview（onChangeEnd 提交+失败回滚已读回 committed）、custom 模板 invalid 禁保存+errorText、guardDraft——PASS。
- [R6/s3] saf_tree_name.dart：primary→内部存储、XXXX-XXXX→SD 卡（卷标）、畸形/桌面路径回退原文；destination 页主标题人类名+raw URI 下沉 subtitle 截断+长按复制；download_settings 摘要与根页摘要同走 downloadDestinationLabel——PASS（注：回退返回整串原文而非 design「末段原文」，测试 :1258 钉死该行为，属有意偏差——更诚实，记录不判负）。
- [R8+R5/s4] network 主页排序 模式→探测→第三方可达→路由快照→高级；第三方 initState 自动探测保留+顶部说明 networkThirdPartyAuto（D7）——PASS。
- [s4] 高级页：页级 draft 单保存提交两字段、dirty 守卫覆盖两字段、保存按钮干净时禁用、恢复默认值 showAppDialog 确认（取消分支测试断言零写入）——PASS。
- [R11/s4] probe 页：NetworkProbeOverview（结论计数+最劣+建议）渲染在 per-host card 之前；card=摘要行+ExpansionTile 默认折叠；networkProbeNotPersisted hint；复制报告不动——PASS。
- [R11/s4] frame_probe：dispose 不 stop、_recording 读 instance.recording、录制状态条+停止+cap 提示、initState 续 ticker、_frames cap=10000 FIFO——PASS。
- [弹层] settings 全域无裸 showDialog/showModalBottomSheet；全经 showAppDialog——PASS。
- [限宽] 所有含 ListView 的 settings 页均过 settingsNarrowBody（含 W6 的 download_tasks，仅纯包装无行为变化）——PASS。
- [l10n] 抽查新 key 四语齐全；lookup.dart 动态 key 全覆盖。
- [测试真实性] server_display 错误态用 container.listen+轮询（:263 注释解释 .future 不 settle）；账号 busy 用 gate Completer 观测中态；dirty guard 双侧分支实走 handlePopRoute——PASS。

## 发现的问题

1. **`settings_page.dart` `_TranslationSummary` 凭据态摘要会过期/错配**（issue，可修）：
   `initState` 里一次性缓存 `store.hasBaidu()/hasLlm()` Future（:384-392），
   无 `didUpdateWidget`——`settings.translationProvider` 变化（如 baidu→llm）时
   State 存活、`_configured` 不更新，会把**旧 provider** 的凭据态显示在新 label 下；
   从凭据页配置返回根页（Navigator.pop 不触发 rebuild）同样陈旧。translate 页
   自身用 push().then+setState 刷新了，根页没有。修复：`didUpdateWidget`
   按 provider 变化重探 + RouteAware(didPopNext) 经 RouteObserverScope/
   replicaRouteObserver 刷新（settings 分支 settingsRouteObserver 已就位，
   routes.dart:1136/864）。测试 :1047 用整树重泵验证，不能覆盖真实 pop 路径——
   属测试覆盖局限而非断言造假。
2. **`download_destination_page` 相册名 draft 缺 dirty 守卫**（issue，可修）：
   design §4 消费者清单明列「destination 相册名（s3）」，AC「draft 页 dirty
   离开确认全覆盖」；该页相册名 TextField+保存按钮是 draft 语义但无 guardDraft、
   无 dirty 跟踪；且 `!_albumFocusNode.hasFocus` 同步条件在失焦后遇重建会把
   未保存输入静默回滚为持久值。修复：加 `_albumDirty`+onChanged+guardDraft，
   同步条件改为 `!_albumDirty`（与 browse/download 页同构）。
3. **`download_settings_page.dart:145` `hintText` 渲染字面 key**（issue，可修，
   pre-existing 非本包引入、本包搬移过该行）：`settingsText(context,
   'namingTemplateHint')` —— 该 getter 有 5 个参数，lookup.dart 无条目，
   落到 `_ => key` 返回字面量 'namingTemplateHint'。修复：直接调
   `context.l10n.namingTemplateHint('{artist}','{title}','{id}','{page}','{ext}')`
   （四语值同为 `{artist}_{title}_{id}_p{page}.{ext}`）。另注：
   settings_test.dart:681 四语 smoke 列表含该 key 但 l10nLookup miss 时返回
   key 本身非空 → 该测试对参数化 key 无检出力（记录，不改测试语义）。
4. **`frame_probe_test.dart` tearDown 注释过界**（trivial）：注释称
   "leave no frames behind"，但 `debugRecordTimings(const [])` 并不清 `_frames`；
   当前两测试不互扰，后续断言 frameCount 的用例会被污染。修复：tearDown 里
   start()+stop()（start 清帧）或修正注释。
5. **`network_settings_page.dart:251` 注释说 "rewrites two stored fields"**，
   实际写三项（dohEnabled+endpointOverride+echFrontHost）——trivial 注释修正。

## 修复记录（fix/09-22-settings-s1-s4-check）

1. `settings_page.dart` `_TranslationSummary` 过期摘要 → 已修：
   - `_probe()` 抽出为可重入的 key-existence 探测（:389-397）；
   - `didUpdateWidget` 在 provider 变化时重探（:400-407）；
   - `with RouteAware` + `didChangeDependencies` 订阅 `RouteObserverScope.maybeOf`
     的 branch observer，`didPopNext` 重探，`dispose` 退订（:409-447）。
   - 注：`didPopNext` 首版写成 `setState(() => _configured = _probe())`
     触发 "setState() callback argument returned a Future"（箭头闭包把
     Future 回传给 setState），已改块体闭包；`test/settings_test.dart`
     新增两个用例（provider 切换重探 :1102、根页 resurface 重探走真实
     `router.push/pop` :1158），两例均绿。
   - 注2：测试初版多 pop 一次（credentials push 只压一层，第二次 pop 抛
     GoError "nothing to pop"），已删。
2. `download_destination_page` 相册名 draft 缺守卫 → 已修：
   `_albumDirty`（:31）+ `guardDraft(dirty: _albumDirty)`（:62）+
   `onChanged` 置脏（:93）+ 保存成功后清脏（:121-124）+ 同步条件改
   `!_albumDirty`（:55）；随带删去只为旧同步条件存在的 `_albumFocusNode`
   （dead code）。新增 widget 测试「save location album name draft asks
   before leaving」覆盖 干净直退/脏拦取消留页/保存后直退+落库 三分支，绿。
3. `download_settings_page.dart` `hintText` 字面 key → 已修：
   `context.l10n.namingTemplateHint('{artist}','{title}','{id}','{page}','{ext}')`
   （签名见 app_localizations.dart:1795，渲染即 `{artist}_{title}_{id}_p{page}.{ext}`）。
4. `frame_probe_test.dart` tearDown → 已修：`FrameProbe` 加
   `@visibleForTesting debugClearTimings()`（frame_probe.dart:65），
   tearDown 改调它（frame_probe_test.dart:25），注释同步更正。
5. `network_settings_page.dart` 注释 "two stored fields" → 已修为
   three（dohEnabled/endpointOverride/echFrontHost，:250-252）。

## 验证结果（本地，flutter 3.47.2）

- `flutter analyze --no-pub`：**0 issues**（此前 `_albumDirty` 未接入时的
  prefer_final_fields 误报随接入消失）。
- `flutter test test/settings_test.dart`：**45/45 通过**。
- `flutter test` server_display_settings + backup + mute_store +
  network_probe + frame_probe + network_probe_page +
  translation_credentials_page：**65/65 通过**（EXIT=0；server_display 的
  retry-listen 用例 ~35s 属正常等待）。
- `flutter test` settings_pop_repro + card_action：**15/15 通过**。
- `git diff --check`：干净。
- `python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-settings-form-semantics`：
  通过（仅 state-management.md 超注入上限的 warning）。
- 环境注记：早前一轮 flutter test 曾卡在 sqlite3 native-assets hook
  >7min（与 w8 worktree 并行 flutter test 争用），本轮 hook 正常跑完；
  首次全绿前的 21 连败实为 `didPopNext` 的 setState-Future 断言 +
  多余 pop 的级联污染，修复后归零——非环境问题。

## 遗留/未验证

- `safTreeDisplayName` 对无法解析的畸形 URI 原样返回原文（saf_tree_name.dart），
  与 design「取末段」表述有偏差，但现有测试明确锁定 honest-fallback 行为，
  判定为有意为之，不改。
- `settings_test.dart` 四语 smoke 对参数化 l10n key 无检出力（lookup miss
  返回 key 本身非空即过）——记录，不改测试语义。
- 真机项未验证：SAF picker、predictive back、屏幕阅读器、1.3x 字体、桌面
  宽度矩阵——widget test 无法覆盖，按约定标记未验证。
- s5 范围内的 about/backup/术语/宽度矩阵核对不属于本次 s1-s4 复核。
