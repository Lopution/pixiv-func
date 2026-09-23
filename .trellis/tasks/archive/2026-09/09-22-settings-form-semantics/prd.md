# 设置表单语义（Roadmap W8）

父任务：`.trellis/tasks/09-22-ui-interaction-consistency`（Astra UI 审查收敛路线图 §4.8）。
规划基线：`main@8067b2d`。全部代码断言与行号以本 leaf `research/` 为准（已对当前 HEAD
逐条复核）。

前置：W1 `09-22-interaction-outcome-correctness` 必须先合入 `main`。其 R6（browse「测试」
改名"应用并测试"）与 R7（凭据清除同步输入框、成败双色）落在本包 owning files 内但归 W1；
本包在其结果之上 rebaseline，不吸收 W1 范围（research/risks.md R1）。

## Goal

把 settings feature 的每个表单控件按父 §5.3 FormState 四态（immediate / draft / action /
destructive）分类落地：根页入口显示当前值，配置入口与内容入口归属清楚，保存位置展示人类
可读名，备份导入先选策略再同权确认，诊断页先摘要后细节，帧探针录制生命周期与控制页分离，
动作术语按 §5.7 种子 + settings-specific 补充表执行。只迁移设置/诊断页与其自有弹层；
弹层现状已全部走 `showAppDialog`（无裸 `showDialog`/`showModalBottomSheet`），本包只做
语义对齐，无入口替换工作；不新建跨 feature 基础设施（§4.11）。

## 拥有文件与边界

owning files（`lib/features/settings/` 全部 + 最小共享触点）：

- `settings_page.dart`、`settings_helpers.dart`、`network_settings_page.dart`（含
  `NetworkAdvancedSettingsPage`）、`network_probe_page.dart`
- `pages/`：theme / language / translate / translation_credentials / browse / history /
  muted_items / account / download_settings / download_destination / backup /
  frame_probe / about_settings
- 共享件扩展（与首个消费者同 stage 同 commit）：`lib/app/widgets/settings/settings_tile.dart`
  增 `subtitle`
- 显式声明的最小 core 触碰：`core/comments/translation_credentials.dart`（新增 `hasX()`
  存在性读法）、`core/debug/frame_probe.dart`（`_frames` 上限）
- `lib/l10n/app_{en,ja,ru,zh}.arb` + 既有 l10n 生成链

不拥有：`download_tasks_page.dart`、`HistoryPage`（W6——根页下载任务 tile 只允许读
`downloadManagerProvider.tasks` 快照做摘要，不改其 UI/动作映射）；onboarding/login/
WebView/Spotlight（W9）；`ReplicaSwitchTile`（settings 不消费，W9 视野）；`HistoryPage`
内"清除历史"入口（W6）。

消费的既有 owner：`AppBreakpoints`、`showAppDialog`/`showAppBottomSheet`、
`showAppSnackBar`、`persistSettings`、`settingsProvider` 细粒度选择器、
`SettingsTile`/`SettingsSection`/`SettingsControl`、`FrameProbe.instance`、
`backupServiceProvider`、`updateServiceProvider`、`safTreePickerProvider`。

## Astra 条目归属（自 traceability §4 复制）

| 条目 | 复核状态 | 本包处理 |
|---|---|---|
| 设置根页缺当前值摘要；查看历史绕经配置页 | 确认 | R2 |
| 主题/语言缺一致选中态的运行时确认 | 运行时 | R3（W10 终验） |
| 浏览设置日常偏好与图源优先级需重排 | 确认 | R4（测试副作用归 W1） |
| 翻译服务列表缺配置状态；成功/失败状态色未区分 | 确认 | R5（清除同步 UI 归 W1） |
| 账号：当前账号、切换中与删除重量、服务端/本地偏好归属 | 部分 | R5 |
| 下载设置/保存位置：即时与草稿混用；暴露技术位置值 | 确认 | R5、R6 |
| 屏蔽管理：解除动作用删除图标；失败输入恢复需回归 | 确认 | R5、R9 |
| 历史归属：本地/远程设置与内容入口分散，远程记录重复入口 | 确认 | R4、R5 |
| 网络高级：字段分别保存，缺统一草稿/应用/重置语义与摘要优先级 | 部分 | R5、R8 |
| 备份恢复："覆盖"以更强实心按钮出现，未先选策略再统一确认 | 确认 | R7 |
| 关于/更新：仓库地址无动作；失败原因未按可行动措施区分 | 确认 | R10 |
| 网络诊断：技术过程优先于服务结论；复制报告能力应保留 | 确认 | R11 |
| 帧探针：`dispose()` 停止录制，无法离开面板采样 | 确认 | R11 |
| 弹层/页面宽度/视觉角色（设置部分） | 确认/部分 | R1、R9 |

## Requirements

### R0. W1 先行项复核（rebaseline 门槛）

启动前确认 `main` 已含 W1：browse「测试」已改名（"应用并测试"，方案 A）且凭据 `_clear`
已同步清输入框+双色 `_status`。未合入则本包不启动；合入后核对 research 行号漂移，
并在 `settings_test.dart:1039/:1100/:1128`（图源）与凭据相关用例上跑基线。

验证：`git log` 见 W1 merge commit；`grep` 确认 `imageSourceTest` 新文案与
`controller.clear()` 已存在；聚焦测试基线绿。

### R1. 公共件：摘要槽、限宽、写入收敛、dirty 守卫、action 四态形

- `SettingsTile`（app/widgets/settings/settings_tile.dart:3-24）新增可选
  `subtitle`（`Widget?`）——根页摘要行载体；首个消费者=根页，同一 commit 接入。
- 所有 settings 页 `ListView` 外包 `Center + ConstrainedBox(maxWidth:
  AppBreakpoints.medium)`（600，form/settings 限宽；全宽度生效，无断点分支）。
  包装为 settings feature 内部 helper（`settings_helpers.dart`），不进 `lib/app/`。
- `persistSettings` 收敛：删除 `network_settings_page.dart:22-35` `_persistNetwork`
  与 `account_settings_page.dart:182-197` `_ServerDisplaySection._write` 两份近重复
  副本；`persistSettings` 接受可选 `failureMessageKey` 参数（`_write` 用
  `serverDisplayWriteFailed`）。
- draft 表单统一 `PopScope`/`canPop` dirty 离开确认（参照 `ProfileEditPage` 契约；
  无改动直接返回不叫"取消"——§5.7）。确认 helper 放 `settings_helpers.dart`，
  与首个消费者（browse 页）同 commit 落地。
- action 统一 idle/busy/success/error：按钮内 spinner（现有 probe/mirror/update 先例）
  + `showAppSnackBar` 终态 + 就地结果双色；不新建封装，各页内联。

验证：`SettingsTile(subtitle:)` 编译期可选、根页全量接入；600/840/1200 宽度下栏宽
恒 600 居中；`_persistNetwork`/`_write` 删除后 `grep` 无残留且调用点走
`persistSettings`；dirty 页 pop 触发确认、干净页直接返回。

### R2. 根页显示关键当前值，内容/配置入口归属清楚（§4.8-1）

`settings_page.dart` 全部 `SettingsTile` 补摘要（ subtitle 槽）：

- 账号设置：当前账号名（未登录→"未登录"）；主题：跟随系统/明亮/黑暗；语言：tag→
  显示名；翻译：服务商名 + 凭据态（"已配置/未配置"，经 R9 的 `hasX()` 读法）；
  浏览：当前图源名（custom→已存前缀，auto→winner 或"自动"）；屏蔽：条目计数；
  历史：开关态（"本地开 · Pixiv 开"式）；网络：当前模式名；下载：命名预设 +
  保存位置人类可读名（R6 解码函数）；下载任务：活动任务数（`downloadManagerProvider
  .tasks` 一次性快照，不订阅 `changes` 流——已知限制登记）；备份：静态 hint；
  关于：版本号（`PackageInfo` Future → FutureBuilder/缓存）。AccountCard 保持。
- 历史入口（D5）：浏览组「历史」tile 保留 → `/settings/history`（摘要=开关态）；
  「我的内容」组新增「查看历史」tile → `openHistory`（`/settings/history/view`），
  复用现有 `historyView` 文案。旧行为→新行为：原先须"设置→历史→查看历史"三级，
  现内容入口直达；配置入口路径不变。

验证：根页每个配置 tile 断言 subtitle 非空且反映当前设置值；两组入口路由断言（tile
→ `/settings/history`；新 tile → `/settings/history/view`）；既有路由顺序测试
`settings_test.dart:777` 更新并通过。

### R3. 主题/语言保留简单单选 + 选中态双通道（§4.8-2）

`theme_settings_page.dart`、`language_settings_page.dart` 结构不动（immediate 单选，
M3 惯例）：补 `Semantics(selected: true)` 第二通道（check 图标已有），限宽包装接入。

验证：widget test 断言选中行 `Semantics` selected 且其余行无；不以颜色/图标为唯一
指示（component-guidelines 可访问性）。

### R4. 浏览页：日常偏好优先 + 图源归组 + custom 图源 draft 显式化（§4.8-3）

`browse_settings_page.dart` 重排：浏览偏好组（blockR18/blockAI/hideMuted）→ 质量组
（preview/detail/view SegmentedButton，immediate）→ 外观组（reduceMotion）→ 图源组
（preset 单选 immediate + custom 输入 draft）。

- custom 输入区：dirty 离开确认（R1 守卫首个消费者）；保存按钮保持"保存"；测试按钮
  消费 W1 结果（W1 方案 A 下为"应用并测试"——名实一致，本包不再动语义）。
- `pixivHistory` 开关从 browse 页移除（:313-322），history 设置页为唯一 owner；
  browse 页不留重复入口。

验证：重排后组序断言；custom dirty 返回弹确认、干净直接返回；`pixivHistory` 在
browse 页不存在、history 页仍存在；图源现有用例 :1039/:1100/:1128 适配后通过。

### R5. FormState 逐页分类落地（§4.8-4 汇总）

| 页面 | 控件 | 分类与改动 |
|---|---|---|
| translate_settings | 服务商单选 | immediate（保持）；凭据入口行摘要="已配置/未配置"（`hasX()`） |
| translation_credentials | 5 字段表单 | draft：dirty 守卫、保存中按钮 spinner+禁用；`_clear` 加 `showAppDialog` 确认（对象=凭据、后果=需重新输入才可翻译）——destructive；`_status` 双色在 W1 已落地，本包核对即可 |
| account_settings | 切换账号 | action：`switchAccount` async 加 busy（行内 spinner 或切换中禁用全部行）；`_confirmRemove` 保持现有确认（destructive 范本）；当前行 check + 语义标签；服务端偏好区归属仅靠 section 的现状可接受（hint :205-212 已有） |
| download_settings | 最大并行数 Slider | immediate-with-preview（D2：拖动预览、松手 `onChangeEnd` 提交、失败回滚 :194-197 已有），语义写明；命名预设 immediate；custom 模板 draft：invalid 时保存按钮**禁用**（替 :152 静默 return）、dirty 守卫 |
| muted_items | 解除屏蔽 | action（可逆，无需确认——M3）：tooltip 改"解除屏蔽*"（R9），trailing 图标 `delete_outline` → `Icons.visibility_outlined`（D6）；pending 行 trailing spinner 替代纯置灰；`_addTag` 失败恢复输入（`_controller.clear()` 移到 toggle 成功后 :52）；`_SectionHeader` → `SettingsSection` |
| history_settings | 两开关 | immediate；`historyView` 入口摘要=当前开关态；pixivHistory 唯一 owner（R4 归并后） |
| network_settings（主页） | 模式三选 | immediate；`_EffectiveRoutesSection` 保持手动 refresh 快照语义；`_ThirdPartyReachabilitySection` 归 action 语义（检测中/可达/不可达已有三色）+ 顶部说明文案（见 D7 确认项） |
| network_settings（高级页） | DoH/ECH | **一页级 draft**：两字段共用单一"保存"提交；dirty 守卫覆盖两字段；"恢复默认值"（:356-360，一次写三项）加 `showAppDialog` 确认——destructive-lite |
| backup_settings | 导出/导入 | action 四态已有；策略流程见 R7 |
| frame_probe / network_probe | 录制/探测 | action；生命周期改造见 R11 |
| about_settings | 更新检查/下载/安装、导出日志 | action 四态已具；失败文案拆分见 R10 |

action 四态补齐点：账号切换 busy、unmute pending 可视、凭据 `_status` 双色（W1）。

验证：每类 FormState ≥1 代表测试——immediate（开关写失败 snackbar+值不变）、draft
（custom 模板 dirty 离开确认+invalid 禁用保存）、action（备份导入 busy→done 计数）、
destructive（移除账号/凭据清除确认流）。

### R6. 保存位置人类可读名，技术 URI 下沉（§4.8-5）

- 新增纯 Dart 解码 `safTreeDisplayName(String treeUri)`（feature 内文件）：`Uri.parse`
  末段 `Uri.decodeComponent` → `primary:` →"内部存储"、`XXXX-XXXX:` 卷标→"SD 卡
  （卷标）"、其余段以 `/` 拼接；解码失败回退末段原文（不吞错，R6=启发式已登记）。
- `download_destination_page.dart` SAF 已选 tile（:126-131）：主标题显示人类名，
  raw `safTreeUri` 下沉次级（subtitle 小字截断 + 长按复制）。
- `download_settings_page.dart::_destinationText`（:201-209）safFolder 分支同步显示
  人类名；相册分支摘要保持（自定义相册名/"PixivFunc 相册（默认）"已有）。

验证：单测覆盖 `primary:Download/pixiv`、SD 卷标、畸形 URI 回退；widget test 断言
主视图无 `content://` 原文、raw URI 在次级可达。

### R7. 备份先选策略再统一确认（§4.8-6，D3 两段 dialog）

`backup_settings_page.dart::_pickStrategy`（:97-123）改两步，均走 `showAppDialog`：

- Step1 选择：文件内容摘要（现有 prompt :104 信息量保留）+ `merge`/`overwrite` 两个
  等权 `RadioListTile`；"继续"在未选定前禁用（M3"Disable confirming actions until a
  choice is made"）。
- Step2 确认：同一确认层级——取消/确认两键，title 复述所选策略后果（merge="将添加
  N…"；overwrite="将清空本地历史并替换…"，服务端只增 note :104 已有文案保留）。
- 两策略确认按钮同权重（FilledButton），消除现 overwrite=FilledButton/
  merge=TextButton 的反向引导；mobile/desktop 呈现下动作序一致（门禁）。
- `BackupService.apply` 与 `BackupImportStrategy` 枚举不动（core 契约完备）。

验证：选择前"继续"禁用；两策略各走一遍断言确认 dialog title 与策略匹配；overwrite
确认后主按钮不再独占实心权重的断言（或结构性 finder）；既有导出确认流 :863 不回归。

### R8. 网络页摘要优先级与第三方探测确认项（§4.8-4 网络部分 + D7）

- 主页排序：模式选择（immediate，摘要=当前模式）→ 探测入口 → 第三方可达性 →
  路由快照 → 高级入口。
- **确认项（须 PRD 记录）**：`_ThirdPartyReachabilitySection` `initState` 即发 3 个
  第三方 HEAD 探测（:479-507）——进页即产生对外网络流量。定案：**保留自动触发**
  （D7），section 顶部加一行说明文案（"进入本页将自动检测下列第三方服务可达性"），
  语义归 action（检测中/可达/不可达三态已有）。若审阅要求改按需触发，s4 仅需删
  `initState` 内 `_check()` 一行，设计已预留该分支。
- 高级页统一 draft 与重置确认见 R5 表。

验证：进页后 section 显示说明文案；探测仍自动发起（行为不变）；三态渲染断言。

### R9. 术语迁移（§5.7 种子对齐 + settings-specific 补充）

种子对齐（改文案不改语义；四语统一审校，risks R12）：

- `unmuteTag/unmuteAuthor/unmuteWork` zh"取消屏蔽*"→"解除屏蔽*"（"取消"义释放给
  只终止当前操作）；en/ja/ru 对应词同审。
- `saveLocationUseCustomAlbum` 按钮换绑现有 `save` key（draft 持久化动词统一"保存"），
  旧 key 删除（grep 确认唯一引用 destination:85-113）。
- `imageSourceTest` 已由 W1 改名，本包只核对。
- `translateCredentialsClear` 保留"清除"动词 + R5 确认 dialog。
- `networkAdvancedReset`"恢复默认值"保留 + R5 确认 dialog。

settings-specific 补充词项登记（只补不改种子；design.md §四附表为权威清单）：
"应用"（一次性非持久化执行，本包内消费者=W1 定名后的镜像测试复合名）、"恢复默认值"、
"解除屏蔽"、"导出/导入"、"合并/覆盖"、"检查更新/下载并安装/查看"、"开始记录/停止"、
"开始探测"、"复制"、"已配置/未配置"、"正在切换"类 busy 文案。

验证：`settings_test.dart:522` 四语齐全测试兜底全部新 key；按补充表核对全部新文案；
grep "取消屏蔽" 无残留。

### R10. 关于页：仓库链接可打开/复制 + 更新错误按原因区分（§4.8-7）

- `aboutSource` tile（:94-98）：主点击 `launchUrl` 打开仓库（`url_launcher` 已依赖，
  spotlight/login 有先例）；次级 trailing icon `Clipboard.setData` 复制 + snackbar。
- `statusText`（:231-243）五态拆分：offline→"检查网络后重试"、rateLimited→"稍后重试
  （GitHub 限流）"、invalid→"清单无效（请反馈）"、busy→"已有更新任务进行中"、
  failed→通用失败；`_applyResult`（:244-251）failed/canceled 分开（permissionRequired/
  installStarted 已有专文）。新增 key 走四语 l10n。

验证：五种 check 失败态与 apply failed/canceled 各自文案断言；仓库 tile tap 触发
launchUrl（可注入/断言意图）、复制触发 snackbar。

### R11. 诊断先摘要后细节 + 帧探针生命周期分离（§4.8-10）

- `network_probe_page.dart`：顶部新增总览区（各 host conclusion 计数 + 最劣结论 +
  一句可行动建议，如"建议模式：ECH"）；per-host card 改摘要行（host+badge+
  firstError）+ 明细默认折叠（steps 收起）；复制报告保留（`toCopyableText` 含 env
  header 不动）；报告不持久化语义在页内 hint 写明（spec 要求）。
- `frame_probe_page.dart`：`dispose()` 不再调 `FrameProbe.instance.stop()`；
  `_recording` 续读 `FrameProbe.instance.recording`；页内常驻"录制中 · N frames"
  状态条 + 停止按钮；hint 改"可离开本页去目标页面滚动，回来停止并复制报告"。
- `core/debug/frame_probe.dart`：`_frames` 加上限（D4：10000 帧 FIFO 丢弃最旧，
  状态条提示已达上限）——声明为最小 core 触碰；start/stop/report 签名不动。

验证：离开/重进页面录制状态连续（`_recording` 仍 true、计数继续）；probe 页总览区
在明细之前渲染；cap 逻辑单测（超限丢最旧）。

## Acceptance Criteria

- [ ] immediate/draft/action/destructive 四类各 ≥1 个状态测试（见 R5 验证行），既有
      `settings_test.dart`/`settings_pop_repro_test.dart`/`server_display_settings_
      test.dart`/`backup_test.dart`/`mute_store_test.dart`/`network_probe_test.dart`
      受影响用例同步更新。
- [ ] 根页全部配置 tile 有当前值摘要；历史内容入口直达 `/settings/history/view`。
- [ ] draft 页 dirty 离开确认全覆盖；immediate 控件无"应用/保存"残留；destructive
      全部有对象+后果+确认。
- [ ] 宽度矩阵 320/390/600/840/1200 + 横屏 + 1.3x 字号：限宽栏内主操作可达。
- [ ] mobile sheet/desktop dialog 语义顺序一致：备份策略选择与确认在两种呈现下
      动作序相同。
- [ ] SAF 人类可读名覆盖 `primary:`/SD 卷标/回退三态；raw URI 不在主视图。
- [ ] 术语按 §5.7+补充表核对；`settings_test.dart:522` 四语齐全；grep 无"取消屏蔽"。
- [ ] 帧探针离开续录 + `_frames` cap；probe 页先摘要后细节；更新失败五态分文案。
- [ ] 每 stage 一个分支 `task/09-22-settings-form-semantics-sN`、每 commit 对应
      implement.md 一个勾选框；`flutter analyze --no-pub`、聚焦+全量 `flutter test`、
      `git diff --check`、`task.py validate` 全绿；本机不可验项 PR 标"未验证"。

## Decisions（规划定案）

- D1 设置栏宽=600（`AppBreakpoints.medium`，"可读栏宽"语义），全宽度生效不门控；
  W10 验收时与其他包栏宽核对一致性（risks R2 登记的跨包核对点）。
- D2 最大并行数 Slider 保 immediate-with-preview（onChangeEnd 提交+失败回滚已存在），
  语义写明"拖动预览，松手生效"；不改 draft。
- D3 备份策略=两段 `showAppDialog`（等权 radio 选择 → 同权确认），不用 bottom sheet；
  `showAppBottomSheet` 本包无消费者，不接入。
- D4 帧探针：`dispose` 不 stop；页内常驻录制状态条为唯一持续指示（不做全局 snackbar/
  根页角标——避免 scope 膨胀，登记为可选增强）；`_frames` cap=10000 FIFO 丢最旧并在
  状态条提示。
- D5 根页历史入口：浏览组配置 tile 保留 → `/settings/history`（摘要=开关态），
  「我的内容」组新增内容入口 → `/settings/history/view`；属导航习惯变化，PR 说明
  旧三级路径→新直达。
- D6 屏蔽 trailing 图标 `delete_outline` → `Icons.visibility_outlined`（恢复可见=可逆
  语义），tooltip 同步"解除屏蔽*"。
- D7 第三方可达性保留进页自动探测（行为不变），补说明文案归 action 语义；改按需仅
  删一行 `initState._check()`——作为审阅确认项写死默认分支。
- D8 凭据"已配置"读法=core 新增 `hasBaidu()/hasLlm()`（只判 key 存在，不读 secret；
  risks R5 推荐），UI 不 read 判空。
- D9 下载任务根页摘要=一次性快照（build 时读 `tasks` 活动数），不订阅 `changes` 流；
  无活动任务时显示静态 hint——已知限制登记。
- D10 dirty 守卫确认 helper 与 action busy 形均内联/收敛于 `settings_helpers.dart`
  （feature 内），不进 `lib/app/`；不抽 FormState 封装（六处 `_busy`/`_saving` bool
  散落可接受，risks R13）。

## Out of scope

- `download_tasks_page.dart` 与 `HistoryPage` UI/动作映射（W6）；阅读设置弹层（W5）；
  资料/收藏弹层（W3）；onboarding/login/Spotlight（W9）。
- `showAppBottomSheet` 接入（无消费者）；触觉反馈（W4 owner，本包可选消费不依赖）；
  新路由、新 provider、新持久化字段。
- core 语义变更：`BackupService.apply`、`FrameProbe` start/stop/report 签名、
  `SettingsController._writeTail`、`NetworkProbe.run` 均不动；唯一 core 改动=
  `hasX()` 读法与 `_frames` cap（已在 Goal/Decisions 显式声明）。
- 网络探测结果持久化（spec 明示 never saved）；第三方探测改按需（默认保留，D7）。
