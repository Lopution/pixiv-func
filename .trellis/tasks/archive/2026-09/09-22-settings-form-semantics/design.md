# 技术设计：设置表单语义（W8）

设计基线：`main@8067b2d`。逐文件精确改动方案见 `research/implementation-draft.md`
（对应父 design.md §4.8 十条）；控件清单与行号见 `research/codebase-settings-pages.md`；
路由/provider/探针生命周期见 `research/codebase-contracts-infra.md`；决策点与 stage 预拆
依据见 `research/risks.md`；M3/Android 惯例依据见
`research/external-material-settings-patterns.md`。本文件只固定结构、契约与边界。

## 一、阶段划分与依赖

W8 是最大工作包之一（父 implement.md §2），按 risks.md 预拆为**阶段 0 + 5 个串行
stage**，每 stage 一个分支一个 PR：

| Stage | 分支 | 内容 | owning files |
|---|---|---|---|
| 0 | 无（当前分支内核对） | Rebaseline：确认 W1 合入、聚焦测试基线、行号漂移复核 | — |
| s1-foundation-root | `task/09-22-settings-form-semantics-s1` | `SettingsTile.subtitle`（首个消费者=根页，同 commit）、限宽 helper、`persistSettings` 收敛（删 `_persistNetwork`/`_write`）、根页全量摘要+历史内容入口 | `app/widgets/settings/settings_tile.dart`、`settings_helpers.dart`、`settings_page.dart`、`network_settings_page.dart`（仅 `_persistNetwork` 删除）、`account_settings_page.dart`（仅 `_write` 删除）、`core/comments/translation_credentials.dart`（hasX）、arb×4 |
| s2-browse-simple | `…-s2` | 主题/语言选中态双通道、浏览页重排+custom draft 守卫（dirty-guard helper 首个消费者）、pixivHistory 归并 history 页、屏蔽页动词/图标/失败恢复/pending 可视/`SettingsSection` 统一 | `theme/language/browse/history/muted_items` 五页、`settings_helpers.dart`（dirty helper）、arb×4 |
| s3-account-translate-download | `…-s3` | 账号切换 busy、凭据清除确认 dialog、下载 Slider 语义写明+custom 模板 invalid 禁用保存+dirty 守卫、保存位置人类可读名（SAF 解码函数）+相册名"保存"换绑 | `account/translate/translation_credentials/download_settings/download_destination` 五页、`saf_tree_name.dart`（新）、arb×4 |
| s4-network-diagnostics | `…-s4` | 网络主页排序+第三方探测说明文案、高级页页级 draft+恢复默认值确认、probe 总览区+明细折叠、帧探针 dispose 不 stop+`_frames` cap+状态条 | `network_settings_page.dart`、`network_probe_page.dart`、`frame_probe_page.dart`、`core/debug/frame_probe.dart`、arb×4 |
| s5-about-backup-terms | `…-s5` | 仓库链接打开/复制、更新失败五态分文案、备份两段策略流、术语全量核对+宽度矩阵验收 | `about_settings_page.dart`、`backup_settings_page.dart`、arb×4 |

依赖：s1 先行（s2-s5 都消费 subtitle/限宽/helper）；s2-s5 owning files 不相交，串行合入，
每 stage 基于已合并 `main` 切新分支。l10n key 按 stage 批量加入，避免 4×arb 反复冲突
（risks.md 拆分支线）。

## 二、关键契约

### 1. `SettingsTile.subtitle`（唯一共享件扩展）

```dart
class SettingsTile extends StatelessWidget {
  const SettingsTile({
    required this.icon, required this.title, this.subtitle, this.onTap, ...
  });
  final Widget? subtitle; // 摘要行载体；不传则与现状渲染一致
}
```

- 摘要=当前值（Android summary 一级惯例，external §2）：显示具体值而非描述 title；
  "未配置"沿用 `imageSourceCustomUnset` 模式。
- 仅 settings feature 消费（grep 验证无外部使用），签名演化安全；扩展+根页接入同 commit
  （s1），满足 §4.11"扩展须同包接入真实消费者"。

### 2. 限宽契约（D1）

```dart
// settings_helpers.dart（feature 内）
Widget settingsNarrowBody(Widget child) => Center(
  child: ConstrainedBox(
    constraints: const BoxConstraints(maxWidth: ContentWidths.settings), // 600
    child: child,
  ),
);
```

- 全宽度生效（窄屏约束不收紧布局，宽屏收栏），无断点分支；单列非双栏（external §3/§6-7）。
- 不新建跨 feature shell；内容宽度读 `ContentWidths.settings`（600，
  `lib/app/layout/content_widths.dart`，父 §5.5 角色表）——断点常量
  不当内容宽度用；若该文件未建，本包首个消费时创建全表。

### 3. `persistSettings` 收敛

```dart
Future<void> persistSettings(BuildContext context, WidgetRef ref,
    Future<void> Function(SettingsController c) write,
    {String? failureMessageKey}); // 默认 settingsWriteFailed
```

删 `_persistNetwork`（network:22-35）与 `_ServerDisplaySection._write`（account:182-197，
后者传 `serverDisplayWriteFailed`）。immediate 失败回滚由 `SettingsController._writeTail`
保证（旧值保持），可见性靠 snackbar——契约不变。

### 4. dirty 守卫（draft 通用，feature 内 helper）

```dart
// settings_helpers.dart；参照 ProfileEditPage PopScope 契约
Widget guardDraft({required bool dirty, required Widget child}); // PopScope 包装
// dirty pop → showAppDialog（取消/放弃两键，放弃动词用"放弃"不用"取消"）
```

消费者：browse custom（s2 首接入）、download custom 模板/destination 相册名（s3）、
credentials、network 高级页（s4）。干净表单直接返回不叫"取消"（§5.7）。

### 5. FormState 逐页分类总表（§5.3 落点）

| 页面 | immediate | draft | action | destructive |
|---|---|---|---|---|
| settings_page | — | — | — | 账号导出复制（已有确认，范本） |
| theme/language | 单选（check+Semantics selected） | — | — | — |
| translate_settings | 服务商单选 | — | — | — |
| translation_credentials | — | 5 字段保存 | — | `_clear`（新增确认） |
| browse | preset/质量/开关 | custom 前缀 | 应用并测试（W1 名） | — |
| history_settings | 两开关 | — | — | — |
| muted_items | — | — | unmute（可逆无确认，pending 可视）、addTag | — |
| account_settings | 服务端偏好（乐观更新） | — | switchAccount（busy） | removeAccount（已有确认） |
| download_settings | Slider(preview)/命名预设/downloadCaption | custom 模板（invalid 禁保存） | — | — |
| download_destination | 相册选/SAF 选 | 自定义相册名（"保存"） | SAF picker | — |
| backup_settings | — | — | 导出/导入 | 导入策略确认（R7） |
| network 主页 | 模式三选 | — | 第三方可达性（自动+D7 文案）、路由快照 refresh | — |
| network 高级页 | — | DoH+ECH 页级单保存 | — | 恢复默认值（新增确认） |
| network_probe | — | — | 探测运行 | — |
| frame_probe | — | — | 录制 start/stop（生命周期独立于页） | — |
| about | — | — | 更新检查/下载/安装、日志导出、仓库打开/复制 | 安装确认（已有） |

同控件不混模式；Slider 的 `_draftMaxDownloads` 仅作拖动预览值，提交点仍是
`onChangeEnd`——immediate-with-preview，PRD D2 写明。

### 6. SAF 人类可读名（R6）

`lib/features/settings/saf_tree_name.dart`（新，feature 内纯函数）：

```dart
String safTreeDisplayName(String treeUri);
// content://…/tree/primary%3ADownload%2Fpixiv → "内部存储/Download/pixiv"
// …/tree/XXXX-XXXX%3A… → "SD 卡（XXXX-XXXX）/…"
// 解析失败 → 末段原文（不吞错）
```

消费者：`download_destination_page` 已选 tile（主标题）+ `download_settings._
destinationText`（safFolder 分支）+ 根页下载摘要。raw URI 下沉 subtitle 截断+长按复制。

### 7. 凭据"已配置"读法（D8，最小 core 触碰）

`TranslationCredentialsStore`（core/comments/translation_credentials.dart:75-81）新增
`Future<bool> hasBaidu()`/`hasLlm()`——只判 key 存在性，不读 secret 值。消费者：translate
页凭据入口行摘要 + 根页翻译摘要。UI 侧不 `read()!=null` 判空（避免读 secret）。

### 8. 备份两段策略流（D3）

Step1 `showAppDialog`：摘要文案 + `RadioListTile`×2 等权 + 取消/继续（未选定禁用继续）。
Step2 `showAppDialog`：取消/确认，title 复述后果（merge=添加；overwrite=清空本地历史并
替换+服务端只增 note）。两策略确认同权（FilledButton）。dialog 上限两动作（M3 §4），
mobile/desktop 动作序一致（门禁）。

### 9. 诊断呈现契约

- probe 页：总览区（conclusion 计数+最劣+建议）在 per-host card 之前；card=摘要行
  （host+`_ConclusionBadge`+firstError）+ 明细默认折叠；`toCopyableText` 不动；结果不
  持久化（页内 hint 写明，spec 要求）。
- 帧探针：`dispose()` 不 `stop()`；`_recording` 读 `FrameProbe.instance.recording`；
  状态条"录制中 · N frames"（达 cap 提示"丢弃最旧帧"）；`_frames` cap=10000 FIFO
  （core/debug/frame_probe.dart，最小触碰已声明）；hint 引导离开采样返回停止。
- 第三方可达性：保留 initState 自动探测（D7），section 顶部说明文案；改按需=删一行。

## 三、术语契约（§5.7 种子 + settings-specific 附表）

种子对齐（改值不改义，四语审校）：

| key | 现状 zh | 目标 |
|---|---|---|
| `unmuteTag`/`unmuteAuthor`/`unmuteWork` | 取消屏蔽* | 解除屏蔽*（"取消"义释放） |
| `saveLocationUseCustomAlbum` | 使用自定义相册 | 换绑现有 `save`="保存"，旧 key 删 |
| `imageSourceTest` | 测试 | W1 已改"应用并测试"，核对即可 |
| `translateCredentialsClear` | 清除凭据 | 保留"清除"+确认 dialog |
| `networkAdvancedReset` | 恢复默认值 | 保留+确认 dialog |

settings-specific 补充词项（**只补不改种子**；本表为权威登记，W10 按表核对）：

| 词项 | 语义 | 消费点 |
|---|---|---|
| 应用 | 一次性非持久化显式执行 | "应用并测试"复合名（W1）内 |
| 恢复默认值 | 字段组重置到默认 | network 高级页 |
| 解除屏蔽 | 屏蔽项可逆移除 | muted 页 |
| 导出/导入 | 备份/日志/凭据文件操作 | backup/about/account |
| 合并/覆盖 | 备份导入策略对 | backup step1/step2 |
| 检查更新/下载并安装/查看 | 更新动作族 | about |
| 开始记录/停止、开始探测、复制 | 诊断动作族 | frame_probe/network_probe/about |
| 已配置/未配置 | 凭据状态摘要 | translate/根页 |
| 正在切换 | busy 文案 | account 切换 |
| 放弃 | 丢弃 draft 草稿确认键 | dirty 守卫 dialog |
| 继续 | 多步流推进键 | backup step1 |

immediate 控件不显示"应用/保存"；draft 动词统一"保存"；"取消"只终止当前操作。

## 四、复用与禁止

- 复用：`AppBreakpoints`、`showAppDialog`、`showAppSnackBar`、`persistSettings`、
  `SettingsSection/Control`、`PackageInfo`、`url_launcher`（spotlight/login 先例）、
  `Clipboard`、`openHistory` 门面、`l10nLookup`、既有 arb→gen-l10n→gen_l10n_lookup 链。
- 禁止：新 provider/路由/持久化字段；跨 feature shell/helper；`lib/app/` 新件（除
  `SettingsTile.subtitle` 扩展）；`showAppBottomSheet`（无消费者）；触觉封装（W4
  owner 未定时不得自造）；`showAppSnackBar` 之外的平行反馈通道；`NetworkProbe.run`/
  `FrameProbe` start/stop/report/`BackupService.apply`/`SettingsController` 签名改动；
  W6 页面（download_tasks/HistoryPage）UI。
- `settingsText`/`_networkText`/`_probeText` 三个 l10nLookup 等价包装并存是登记在案
  的收敛点，本包不强制（避免扩散 diff）。

## 五、状态覆盖与测试策略

- 状态设计：loading/error（`settingsUnavailable` 壳已有）、empty（屏蔽/任务摘要空态=
  静态文案）、busy（spinner+禁用）、cancel（dialog 取消不升层）、back（dirty 守卫/
  干净直返）、reduced-motion（本包无动画新增，`reduceMotion` 开关链已存在）。
- 每类 FormState 代表测试：immediate=开关写失败 snackbar+值不变；draft=custom 模板
  dirty 确认+invalid 禁保存；action=备份导入 busy→done 计数；destructive=移除账号/
  凭据清除确认流。
- 纯函数单测：`safTreeDisplayName`（primary/SD 卷标/畸形回退）、`_frames` cap。
- widget 测试：根页摘要、历史双入口路由、选中态 Semantics、browse 重排与 dirty、
  凭据确认流、备份两段流、probe 总览序、帧探针离开续录。
- 既有测试更新点：`settings_test.dart:777`（根页路由顺序）:863（导出确认流）:989
  （DoH 校验）:1039/:1100/:1128（图源）:522（四语）；`settings_pop_repro_test.dart`
  路由栈不炸回归。
- 测试陷阱（quality-guidelines）：`material_ui` 遮蔽（断言用其 SnackBar/SwitchListTile
  类型）；`settingsProvider` AsyncNotifier 错误态断言用 listen 不 await `.future`；
  `translationCredentialStoreProvider`/`backupFilePickerProvider` 可 override。
- 回归测试先对旧实现证伪（quality-guidelines"proven against the defect"）：新断言先在
  未修代码上确认失败。

## 六、运行时证据缺口（PR 标"未验证"）

SAF 真机 picker 与各厂商 tree URI 形态、predictive back 真手势下 dirty 守卫、
TalkBack/Narrator 朗读 selected/录制状态条、1.3x 大字体摘要行溢出、桌面宽度矩阵实测、
`PackageInfo` 首帧延迟观感。
