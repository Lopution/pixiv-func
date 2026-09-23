# 执行计划：设置表单语义（W8）

需求见 `prd.md`，技术设计见 `design.md`，逐文件精确改动方案见
`research/implementation-draft.md`（行号已对 `main@8067b2d` 核实）。本包按
`research/risks.md` 预拆为**阶段 0 + 5 个串行 stage**，每 stage 独立分支
`task/09-22-settings-form-semantics-sN`、独立 PR；s2-s5 依次基于已合并 `main` 切出。

## 环境

按 `.trellis/spec/frontend/quality-guidelines.md` 的 Build Toolchain 约定：

```bash
export PATH=/opt/flutter-3.47.2/bin:$PATH
```

不要手改 `android/local.properties`。新增 l10n key 流程：编辑
`lib/l10n/app_{en,ja,ru,zh}.arb` → `flutter gen-l10n` → `python3 tool/gen_l10n_lookup.py`；
key 按 stage 批量加入避免 4×arb 反复冲突。

验证命令（每 stage 收尾都要跑）：

```bash
flutter analyze --no-pub
flutter test
git diff --check
python3 ./.trellis/scripts/task.py validate .trellis/tasks/09-22-settings-form-semantics
```

> 全量 `flutter test` 存在本机已知 loopback 噪声（见 quality-guidelines）：看似不相关的
> 测试文件抛 `TimeoutException`、单文件重跑即绿——先按 spec 判定噪声再排查。
> widget 测试注意 `material_ui` 遮蔽（Snackbar/SwitchListTile 断言用其类型）与
> `settingsProvider` AsyncNotifier `.future` 错误态不 settle（用 listen 断言 error 态）。

本机不可验项（真机 SAF picker/厂商 URI 形态、predictive back 真手势、TalkBack/Narrator、
1.3x 大字体溢出、桌面宽度实测）在 PR body 标"未验证"，不得由 widget test 推断通过。

## 阶段 0：Rebaseline（在当前规划分支内核对，不切 task 分支）

- [ ] 确认 W1 `09-22-interaction-outcome-correctness` 已合入 `main`：`git log` 见其
      merge commit；`grep` 核实 browse「测试」已改名"应用并测试"（方案 A）、凭据
      `_clear` 已同步清输入框+`_status` 双色。**未合入则停止**，本包不吸收 W1 范围
      （risks R1）。
- [ ] 在合入后 `main` 上复核 research 行号漂移，运行聚焦测试记录真实基线：
      `flutter test test/settings_test.dart test/settings_pop_repro_test.dart test/server_display_settings_test.dart test/backup_test.dart test/mute_store_test.dart test/network_probe_test.dart`。
- [ ] 若 W1 落地内容使 s2/s3 相关控件事实变化（如镜像测试复合名），按 PRD R4/R9
      的既定接法消费，不回改本规划。

## 阶段 s1：foundation-root（`task/09-22-settings-form-semantics-s1`）

- [ ] `lib/app/widgets/settings/settings_tile.dart` 增可选 `subtitle`（`Widget?`）；
      `settings_helpers.dart` 增 `settingsNarrowBody`（Center+ConstrainedBox 600=
      `AppBreakpoints.medium`）；`persistSettings` 增可选 `failureMessageKey`；
      删除 `network_settings_page.dart:22-35` `_persistNetwork` 与
      `account_settings_page.dart:182-197` `_write` 副本并迁移调用点。
      提交：`refactor(settings): SettingsTile 增摘要槽并收敛 persistSettings 副本`
- [ ] `settings_page.dart` 全量 `SettingsTile` 接 `subtitle` 摘要（账号/主题/语言/
      翻译/浏览/屏蔽/历史/网络/下载/下载任务快照/关于版本号；备份=静态 hint），
      「我的内容」组新增「查看历史」tile → `openHistory`（D5）；全部页体过
      `settingsNarrowBody`。依赖 core `hasBaidu()/hasLlm()`（同 commit 前置小改
      `translation_credentials.dart:75-81` 区域）供翻译摘要。
      测试：根页摘要渲染、历史双入口路由、`:777` 路由顺序更新。
      提交：`feat(settings): 根页入口显示当前值摘要并直达历史`
- [ ] 其余 settings 页 `ListView` 逐个过 `settingsNarrowBody`（纯包装，无行为变化）。
      提交：`feat(settings): 设置页内容栏限宽 600`
- [ ] s1 新增 l10n key（已配置/未配置、未登录、开/关态拼接等）四语齐全；
      `settings_test.dart:522` 通过。提交并入所在 checkbox。
- [ ] s1 收尾：`flutter analyze --no-pub`、聚焦+全量 test、`git diff --check` 绿；
      PR `gh pr create --fill`，CI 绿后 `gh pr merge --merge`。

## 阶段 s2：browse-simple（`task/09-22-settings-form-semantics-s2`）

- [ ] `settings_helpers.dart` 增 dirty 守卫 `guardDraft`/`confirmDiscardDraft`（PopScope
      参照 ProfileEditPage；确认 dialog 取消/放弃两键）——首个消费者=browse custom
      输入区，同 commit。提交：`feat(settings): draft 表单 dirty 离开确认 helper`
- [ ] `browse_settings_page.dart` 重排（偏好组→质量组→外观组→图源组）；custom 输入
      接 dirty 守卫；pixivHistory 开关移除（:313-322）；测试按钮消费 W1 文案。
      测试：组序、dirty 确认、`:1039/:1100/:1128` 适配。
      提交：`feat(settings): 浏览页偏好优先重排与自定义图源草稿守卫`
- [ ] `history_settings_page.dart`：pixivHistory 唯一 owner 落定；`historyView` 入口
      摘要=当前开关态。提交：`feat(settings): 历史记录开关归并历史设置页`
- [ ] `theme_settings_page.dart`/`language_settings_page.dart`：选中行补
      `Semantics(selected: true)` 第二通道。提交：`feat(settings): 主题语言选中态补语义通道`
- [ ] `muted_items_page.dart`：unmute tooltip 改"解除屏蔽*"+图标
      `Icons.visibility_outlined`（D6）；`_addTag` 的 `_controller.clear()` 移到 toggle
      成功后；pending 行 trailing spinner；`_SectionHeader` → `SettingsSection`。
      测试：失败输入恢复、pending 可视、动词文案。
      提交：`fix(settings): 屏蔽解除动词与失败输入恢复对齐术语`
- [ ] s2 l10n：unmute* 四语审校改值、放弃/当前语义标签等；s2 收尾验证+PR。

## 阶段 s3：account-translate-download（`task/09-22-settings-form-semantics-s3`）

- [ ] `account_settings_page.dart`：`switchAccount` 加 busy（行内 spinner/切换中禁用
      行）；当前行 check 补语义标签；"正在切换"文案。提交：
      `feat(settings): 账号切换补 busy 态`
- [ ] `translation_credentials_page.dart`：`_clear` 前 `showAppDialog` 确认（对象=
      凭据、后果=需重新输入）；dirty 守卫接入；核对 W1 已落地的清输入框+双色
      `_status`。测试：确认流取消/确认两分支。
      提交：`feat(settings): 凭据清除加确认并接 dirty 守卫`
- [ ] `translate_settings_page.dart`：凭据入口行摘要="已配置/未配置"（`hasX()`）。
      提交：`feat(settings): 翻译页凭据入口显示配置状态`
- [ ] `download_settings_page.dart`：Slider 语义写明（immediate-with-preview，注释/
      hint）；custom 模板 invalid 时保存按钮禁用（替 :152 静默 return）；dirty 守卫。
      提交：`fix(settings): 命名模板无效时禁用保存并接草稿守卫`
- [ ] 新增 `lib/features/settings/saf_tree_name.dart` `safTreeDisplayName`（纯 Dart
      解码+回退）；`download_destination_page.dart` SAF tile 主标题=人类名、raw URI
      下沉 subtitle 截断+长按复制；`_destinationText` safFolder 分支同步；自定义相册
      按钮换绑 `save` key、删 `saveLocationUseCustomAlbum`。测试：解码单测三态、
      widget 断言主视图无 `content://`。
      提交：`feat(settings): 保存位置主视图人类可读名`
- [ ] s3 l10n（凭据确认、SD 卡/内部存储等）四语；s3 收尾验证+PR。

## 阶段 s4：network-diagnostics（`task/09-22-settings-form-semantics-s4`）

- [ ] `network_settings_page.dart` 主页排序（模式→探测入口→第三方可达性→路由快照→
      高级）；`_ThirdPartyReachabilitySection` 顶部补"进入本页自动检测"说明文案
      （D7 保留自动）。提交：`feat(settings): 网络页摘要优先级与可达性说明`
- [ ] `NetworkAdvancedSettingsPage`：DoH/ECH 合并页级 draft（单一"保存"提交两字段）；
      "恢复默认值"加 `showAppDialog` 确认；dirty 守卫覆盖两字段。
      测试：`:989` DoH 校验适配、重置确认流。
      提交：`feat(settings): 网络高级页统一草稿与重置确认`
- [ ] `network_probe_page.dart`：顶部总览区（conclusion 计数+最劣+建议）；per-host
      card 摘要行+明细默认折叠；报告不持久化 hint。复制报告不动。
      提交：`feat(settings): 网络探测先摘要后细节`
- [ ] `frame_probe_page.dart`：`dispose()` 不 `stop()`；页内"录制中 · N frames"状态条
      +停止按钮；hint 改引导文案；`core/debug/frame_probe.dart` `_frames` cap=10000
      FIFO+状态条上限提示。测试：离开/重进续录、cap 单测。
      提交：`feat(settings): 帧探针录制生命周期与控制页分离`
- [ ] s4 l10n（可达性说明、probe 总览/建议、帧探针状态/hint、重置确认）四语；
      s4 收尾验证+PR。

## 阶段 s5：about-backup-terms（`task/09-22-settings-form-semantics-s5`）

- [ ] `about_settings_page.dart`：`aboutSource` tile 主点击 `launchUrl` 打开仓库 +
      trailing 复制（`Clipboard.setData`+snackbar）。提交：
      `feat(settings): 仓库链接可打开与复制`
- [ ] `statusText`/`_applyResult` 拆分：offline/rateLimited/invalid/busy/failed 五态
      分文案 + apply failed/canceled 分开；新 key 四语。测试：各态文案断言。
      提交：`fix(settings): 更新失败按可行动原因区分文案`
- [ ] `backup_settings_page.dart::_pickStrategy` 两段流：step1 摘要+等权 RadioListTile
      （未选禁用继续）→ step2 同权确认（title 复述后果）。测试：禁用态、两策略确认
      title、mobile/desktop 动作序一致。提交：
      `fix(settings): 备份导入先选策略再同权确认`
- [ ] 术语全量核对：grep "取消屏蔽"无残留；按 design.md §三附表核对全部新文案；
      `settings_test.dart:522` 四语齐。提交：`chore(settings): 术语表全量核对`
- [ ] 宽度矩阵 320/390/600/840/1200+横屏+1.3x 字号验收（widget/golden 可覆盖部分
      自动化，其余标"未验证"）；s5 收尾验证+PR。

## 收尾（s5 合入后）

- [ ] `flutter analyze --no-pub` 与全量 `flutter test` 通过（噪声按 spec 判定）；
      `git diff --check` 干净；`task.py validate` 通过。
- [ ] W8/W9 门禁逐条核对：四态测试、只迁设置页、复用既有 owner、宽度矩阵、
      mobile/desktop 语义顺序一致（父 implement.md §6）。
- [ ] 收尾记账：`add_session.py` + `task.py archive`（随最后 PR 的提交）。

## 边界（不做）

- `download_tasks_page.dart`/`HistoryPage`（W6）、阅读设置（W5）、资料/收藏（W3）、
  onboarding/login/Spotlight（W9）；`showAppBottomSheet`（无消费者）；触觉封装
  （W4 owner）；新 provider/路由/持久化字段；`BackupService.apply`/`FrameProbe`
  start/stop/report/`NetworkProbe.run`/`SettingsController` 签名；probe 结果持久化；
  第三方探测改按需（默认保留，PRD D7 确认项）。
