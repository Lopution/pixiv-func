# W8 codebase 调研：设置页控件清单与 FormState 分类

> 基线：`main@8067b2d`（2026-09-22，W1 仍 planning，全部"W1 修…"项在当前代码中仍存在）。
> FormState 分类依据父任务 design.md §5.3：immediate / draft / action / destructive。
> 行号已逐条对照当前文件核对。

## 0. 共用写入与呈现件

| 件 | 位置 | 语义 |
|---|---|---|
| `persistSettings` | `lib/features/settings/settings_helpers.dart:17-30` | 包一层 try/catch，失败 `showAppSnackBar(settingsWriteFailed: $error)`；成功无提示。是所有"immediate"控件的唯一写入路径 |
| `_persistNetwork` | `network_settings_page.dart:22-35` | `persistSettings` 的逐字副本（仅作用域私有）——重复实现，收敛点 |
| `_ServerDisplaySection._write` | `account_settings_page.dart:182-197` | 第三份近重复封装，错误文案用 `serverDisplayWriteFailed` |
| `settingsUnavailable` | `settings_helpers.dart:32-51` | settings==null 时的 loading/error 壳（FeedLoading/SettingsLoadError） |
| `openSettingsPage` | `settings_helpers.dart:53-55` | `context.push` 包装 |
| `SettingsTile` | `lib/app/widgets/settings/settings_tile.dart:3-24` | icon+title+chevron；**无 subtitle/summary 参数**——根页无法展示当前值，是"根页显示关键当前值"的直接缺口 |
| `SettingsSection` | `lib/app/widgets/settings/settings_section.dart:3-21` | 分组头（primary 色、Semantics header） |
| `SettingsControl` | `lib/app/widgets/settings/settings_control.dart:3-26` | `SwitchListTile` 薄封装；无 busy/pending/disabled 语义 |
| `ReplicaSwitchTile` | `lib/app/widgets/replica_switch_tile.dart:3-35` | 另一套开关 tile（InkWell+Row+Switch），仅 onboarding language/theme 与 login_page 使用——settings 不消费，归 W9 视野，不并 |
| `SettingsController` | `lib/core/settings/settings_controller.dart:26-164` | `_writeTail` 串行写队列；失败抛 `SettingsWriteException`，旧值保持（state 不变）→ immediate 语义天然满足"失败回滚并可见"中的回滚半段，可见性靠 snackbar |
| `showAppDialog`/`showAppBottomSheet` | `lib/app/motion/app_overlays.dart:9-53` | settings 内 4 处 dialog 已全部走 `showAppDialog`；**无裸 `showDialog`/`showModalBottomSheet`**（grep 验证）。弹层迁移只剩语义角色核对，无入口替换工作 |

## 1. 根页 `settings_page.dart`（SettingsPage/_SettingsList）

- 结构：`Scaffold(resizeToAvoidBottomInset:false)` + `RootSwipeSwitcher` + `ListView`（:45-63, 95-205）。
- AccountCard：:98，唯一带摘要的入口（name/mail/subtitle，`account_settings_page.dart:19-46`）。
- 配置入口：账号 :99-103、账号导出 :107-112、外观组 :113-128（主题/语言/翻译）、浏览组 :129-144（浏览/屏蔽/历史）、我的内容组 :148-163（稍后再看/追更/本地小说——内容入口）、网络组 :164-179（网络/下载/下载任务）、数据组 :180-185（备份）、关于 :187-191、开发者组 :195-202（帧探针，`developerOptionsProvider || !kReleaseMode` 门控）。
- **摘要行现状：全部 `SettingsTile` 只有 title，无当前值**（组件不支持）。Astra"主题、语言、图像质量等入口仍缺当前值摘要"确认。
- "查看历史"无直达：根页→`/settings/history`（开关页）→`historyView` tile→`/settings/history/view`；`openHistory`（routes.dart:1390）直达 `/settings/history/view` 但根页没挂它。
- `_confirmCopyAccount` :211-233：destructive-adjacent（明文凭据上剪贴板），已有 `showAppDialog` 警告 + Android<13 敏感标记二次提示 :246-254——destructive 语义范本已存在。
- 内容入口 vs 配置入口分组已存在（library 组独立于偏好组），可保留，仅需在 PRD 固化为契约。

## 2. `pages/theme_settings_page.dart`（51 行）

- 三选 ListTile+check（:24-47），`persistSettings(selectTheme)` immediate。符合 §4.8"保留简单单选"。缺：选中态除 check 图标外无第二通道（运行时确认项）。

## 3. `pages/language_settings_page.dart`（53 行）

- 四选 ListTile+check（:34-48），`persistSettings(selectLanguage)` immediate。同上。

## 4. `pages/translate_settings_page.dart`（92 行）

- 服务商四选 :35-50 immediate（`selectTranslationProvider`）。
- **缺配置状态摘要**：baidu/LLM 行不显示凭据是否已配置（Astra"服务列表缺配置状态"）。`TranslationCredentialsStore`（`core/comments/translation_credentials.dart:75-81`）只有 read/write/delete 方法，无 "isConfigured" 轻量查询——需要一个配置状态读出口（读 key 存在性，不读 secret 值）。
- 凭据入口条件渲染 :51-63；hints :64-78。

## 5. `pages/translation_credentials_page.dart`（263 行）

- draft 表单：5 个 controller（:23-27），`_load()` 回填已存凭据 :47-72（含 secret 回填 obscured 字段——有意设计注释 :53-56）。
- `_save` :74-129：本地校验（baidu 非空 / LLM https+apiKey），写 `CredentialStore`；成功 `_status=Saved`，失败分类（invalid/store error）。
- `_clear` :131-156：**只删 store 不清输入框**（W1"清除同步 UI"项，当前未修）；**无确认 dialog**——deleting secure storage 是 destructive，缺确认边界。
- `_status` 单一 `colorScheme.primary` 渲染 :206-213——**成功/失败同一颜色**（Astra"成功/失败状态色未区分"）；无 busy 指示在按钮上（`_saving` 只 disable 按钮 :214-224）。
- 无 dirty-guard/PopScope：编辑后直接返回静默丢草稿。

## 6. `pages/browse_settings_page.dart`（387 行）——FormState 混用最严重

- 图源选择：5 preset ListTile+check :144-165，custom tile :166-185（subtitle 显示已存值或"未配置"——已有摘要行），`auto` 行 subtitle 显示竞速 winner（`autoImageSourceWinnerProvider` :138, 151-155）。
- custom 输入框 :186-198（`_customDirty` 脏标记 :24, 196）+ **保存按钮 :206-209**（`_saveCustomSource` :74-79）+ **测试按钮 :210-220**（`_testMirror` :85-119）。
- **错配点 A（W1 项）**：`_testMirror` :86 先调 `_applyCustomInput()` → 内部 `_selectSource(normalized)` :68 **先持久化再测试**；"测试"实际="保存并应用+测试"。W1 未落地，W8 迁移时建立在其修复结果上（改名"应用并测试"或纯测试）。
- **错配点 B**：draft（custom source 输入框）与 immediate（preset 选择、三个 SegmentedButton、四个开关）同页混排无说明；输入框无 dirty 离开确认。
- 三个质量 SegmentedButton：preview :235-250 / detail :262-280 / view :291-310，均 immediate。
- 开关组：pixivHistory :313-322（**与 history_settings_page :38-47 重复入口**）、blockR18 :323-330、blockAI :331-338、hideMuted :339-347、reduceMotion :349-357，均 immediate。
- "日常浏览偏好优先，图源/网络配置归组"现状：图源在最顶部 :143，日常偏好在中下部——顺序与目标相反，需重排。

## 7. `pages/history_settings_page.dart`（65 行）

- localHistory 开关 :29-37、pixivHistory 开关 :38-47（immediate；**与 browse 页重复**——"远程记录设置有重复入口"确认，归属应唯一化）。
- `historyView` 内容入口 tile :48-53 → `openHistory`。
- 无"清除历史"入口（清除在 HistoryPage 内，归 W6）。

## 8. `pages/muted_items_page.dart`（158 行）

- 三类条目纵向平铺：tags :83-95 / users :98-111 / works :114-128，各 `ListTile(dense)` + trailing `Icons.delete_outline` IconButton。
- **图标/动词错配**：动作是"解除屏蔽"（可逆，MuteStore toggle），视觉是删除图标 + tooltip `unmuteTag/unmuteAuthor/unmuteWork`（zh 文案"取消屏蔽/取消屏蔽作者/取消屏蔽此作品" :90,:106,:123）——"取消屏蔽"与 §5.7 种子"解除屏蔽"不一致，且"取消"种子义=只终止当前操作，存在术语冲突。
- `_addTag` :47-54：**`_controller.clear()` 在 :52 先于异步 toggle 完成**，失败时输入已丢（"失败输入恢复需回归"确认）。
- pending 语义：`state.pending.contains(MuteKey.x)` 时 onPressed=null :91-93,:107-109,:124-126——按钮置灰但无 spinner/action 状态可视。
- `_SectionHeader` :140-158 是 `SettingsSection` 的私有重复（样式略异：无 16 横向 padding）。
- 条目 onTap 导航（openTagSearch/openUser/openIllust）与 unmute 共存于同行，语义可保留但需确认主次。

## 9. `pages/account_settings_page.dart`（253 行）

- AppBar `+` 添加账号 :76-80 → openLogin。
- 账号列表 :100-132：current 行 check 图标 :111-115 + trailing delete IconButton :116-121；**点按切换 `switchAccount` :124-131 无 busy/进行中指示**（action 语义缺 busy；切换可能异步耗时）。
- `_confirmRemove` :146-173：`showAppDialog` + cancel/confirm——destructive 模板正确（"移除账号"用词符合 §5.7"移除"级）。
- `_ServerDisplaySection` :179-252：服务端偏好与本地账号同页，有 section 头 + hint :205-212；loading/error/data 三态完整 :213-249；**错误重试走 `ref.invalidate` + TextButton(retry) :222-225**。
- Astra"切换中与删除重量、服务端/本地偏好归属"：删除与 current-check 同 trailing 并排、服务端设置以普通 Divider 分隔（:137），归属仅靠 section 标题。

## 10. `pages/download_settings_page.dart`（218 行）——immediate/draft 混用第二现场

- 最大并行数 Slider :67-76：**draft 显示 + onChangeEnd 隐式提交**（`_saveMaxDownloads` :188-198 失败时回滚 `_draftMaxDownloads`）——immediate 语义下用 draft 变量做乐观值，可保留但属"同一控件混模式"边缘（拖动中显示未提交值）。
- 保存位置 tile :78-84 → `/settings/download/destination`，subtitle=`_destinationText` :201-209 只显示**类型名**（相册/自定义相册(名)/文件夹），不显示实际位置。
- downloadCaption SwitchListTile :85-96 immediate。
- 命名预设四选 :105-120 immediate；custom 模板 TextField+实时 errorText+preview+变量说明 :121-148 + **保存按钮 :149-169**（draft）。
- **错配点**：custom 模板 invalid 时保存按钮静默 return（:152 `if (!isValidTemplate) return;`——按了没反应）；draft 无 dirty 离开确认。

## 11. `pages/download_destination_page.dart`（150 行）

- 相册选 :58-72 immediate；自定义相册名 TextField :73-84 + "使用自定义相册"按钮 :85-113（draft，invalid→snackbar :93-97）。
- SAF 文件夹 tile :115-125 → `_pickSafFolder` :137-149（系统 picker → persist）。
- **技术 URI 外露**：已选 SAF 时副标题直接渲染 `destination.safTreeUri` :126-131（`content://...` 原始串）——"主视图展示人类可理解名称，技术 URI 下沉"的直接改造点。`DownloadDestination`（`core/download/download_destination.dart:11-117`）无 displayName 派生方法，需在 UI 层解码 treeUri 末段或经 SAF 通道查 DocumentFile 名。

## 12. `pages/backup_settings_page.dart`（158 行）

- `_busy` :37 控制两个 tile enabled；export :39-52 / import :54-92 均为 action 语义（idle/busy/success/error via snackbar）。
- `_pickStrategy` :97-123：`AlertDialog` 内 TextButton(取消) / TextButton(合并 :111-114) / **FilledButton(覆盖 :115-119)**。
- **错配点**：合并/覆盖权重不对称——覆盖（更强破坏）拿 FilledButton 主按钮位，合并只是 TextButton；"先选策略再统一确认"要求：第一步策略选择（等权 radio/选项），第二步同一确认层级执行。`BackupImportStrategy`（`core/backup/backup_envelope.dart:7-17`）语义现成：merge=服务端屏蔽只增+本地并集+历史 upsert；overwrite=设置整体替换+本地作品屏蔽替换+历史清后重插（服务端仍只增，note 文案 :104 已写明）。
- `backupServiceProvider` 依赖可用账号（`backup_service.dart:142-147` 无账号抛 accountRequired）——import 前未登录错误走 snackbar，可考虑前置禁用/说明。

## 13. `network_settings_page.dart`（566 行，含 NetworkSettingsPage + NetworkAdvancedSettingsPage + 2 个 section）

- 主页面：模式三选 _modeTile :86-109/:138-161 immediate；**_EffectiveRoutesSection :370-449**（非响应式，initState 快照+手动 refresh :420-424，host→routeKind 行 :437-445）；探测入口 :113-119；**_ThirdPartyReachabilitySection :454-565**（**initState 自动发起 3 个 HEAD 探测 :479-507**——进页面即发网络请求；refresh :524-528；可达/不可达/检测中三色 :544-561）；高级入口 :123-129。
- 高级页：DoH endpoints TextField+保存 :308-330（`_saveEndpoints` :202-221，逗号分隔 https 校验 :223-243）；ECH host TextField+保存 :332-354（`_saveEchHost` :245-262）；**"恢复默认值" tile :356-360**（`_resetDefaults` :264-281 一次写三项+回填 controller）。
- **错配点**：两个字段各自保存按钮（draft 分散，无统一 apply/reset 语义——Astra"高级字段分别保存，缺统一草稿/应用/重置语义"确认）；字段 dirty 但离开无确认；`_persistNetwork` 是 `persistSettings` 重复（:22-35）。

## 14. `network_probe_page.dart`（447 行）

- `_targets` :66-79 动态含镜像 host；`_runAll` :102-123 全并发 Future.wait，`_running` 门控；`_runOne` :125-247 逐层（system-dns/doh/tcp/tls/http/ech/no-sni）。
- 呈现：run 按钮 :261-275 → 每 host 一个 `_HostProbePanel` card :277-283；panel 内 host 名+**结论 badge :322**（`_ConclusionBadge` :389-447，8 种 conclusion 各自颜色）+ DNS 分歧提示 :347-356 + **逐层明细全量铺开 :357-366** + 复制报告 :368-380。
- **错配点**：明细与结论同级平铺——"技术过程优先于服务结论"（先摘要后细节：总体结论/建议先行，逐层明细折叠或下沉）；无跨 host 总摘要行；复制报告能力保留要求已在 Astra 注明。
- 生命周期：结果存 page State（`_finished`/`_errors` :81-83），离开页面即丢；probe future 无取消（dispose 后 completion 被 mounted 守卫，无副作用）。

## 15. `pages/frame_probe_page.dart`（99 行）

- `FrameProbe.instance` 单例（`core/debug/frame_probe.dart:17-105`：start/stop/report，帧数据 verbatim 保留）。
- **dispose() :27-31 调 `FrameProbe.instance.stop()`**——离开控制页即终止录制（"无法离开面板去目标页面采样"确认）；`_ticker` 500ms 只刷帧计数 :43-46；`_toggle` :33-49 开始/停止+生成 report；report `SelectableText`+复制 :76-94。
- "录制生命周期与控制页分离"改造点：recording 由单例持有已经成立，缺的是 dispose 不 stop、离开后录制中状态可见（返回设置/全局可见指示）、报告在重进页面时可再取（`_frames` 在 stop 后仍可读）。

## 16. `pages/about_settings_page.dart`（385 行）

- 版本 tile :69-74 + 7-tap 开发者解锁 `_onVersionTap` :33-47；许可证 :75-84；归属 :89-93。
- **仓库链接无动作** :94-98（`aboutSource` ListTile 无 onTap；`url_launcher: ^6.3.2` 已在 pubspec :51，spotlight/login 已有 launchUrl 先例）。
- 导出日志 :99-103 → `_exportCrashLog` :142-154（share_plus 分享文件；无日志时 snackbar :147-149）。
- 显示刷新率 :109-121（仅 Android，`FlutterDisplayMode.active` FutureBuilder——信息只读）。
- 更新区 `_AboutUpdateSection` :156-385：capability 分支（storeManaged/fdroid/disabled :208-222）→ `_githubUpdateControls` :228-302；`_check` :304-326、`_cancelApply` :328-341、`_confirmAndApply` :343-384（`showAppDialog` 确认 :347-363）。
- **错配点**：`statusText` :231-243 把 invalid/rateLimited/offline/failed/busy **五种状态并入同一 `aboutUpdateFailed`**（"按可行动原因区分"的直接对象）；`_applyResult` 类似 :244-251；按钮三态（检查/取消/下载）合并在一个 FilledButton :276-298。

## 17. `pages/download_tasks_page.dart`（257 行）——仅联动理解，归 W6

- 记录接口面：`DownloadManager` 直读 + `changes` 流监听 :20-35；组卡 `_DownloadGroupSection` :70-172（pause/resume/cancel 图标按钮）；任务卡 `_DownloadTaskTile` :174-257（**retryable 状态映射 refresh 图标+"重试" :205-218**——Astra"暂停任务仍映射到 retry 图标"项在此，归 W6）；paused 是 `failureKind==paused` 的 retryable 子态 :250-253。
- W8 边界：不迁移此页，但根页"下载任务"入口 tile 摘要（如活动任务数）若做需要读 `_manager.tasks`——注意 DownloadManager 不在 Riverpod 态里，是 `downloadManagerProvider` 单例。

## 18. 宽度现状（全设置 feature）

- `features/settings/` 内 **零 `ConstrainedBox`/`maxWidth`/`AppBreakpoints` 引用**（grep 全 lib 验证）——所有页面 ListView 全宽铺开。
- 现有宽度先例：`welcome_page.dart:26-28` `ConstrainedBox(maxWidth:520)`（onboarding，归 W9）；`illust_detail_page.dart:403` `AppBreakpoints.useTwoPaneDetail`；`AppBreakpoints.expanded=1200` 注释"content pages may cap their readable column"。form/settings 限宽数值无现存 owner——需选定（如 ≤840dp 栏宽），属本包决策点。

## 19. 动词清单（设置场景现状，zh）

| 动词 | l10n key / 位置 | 用途 | §5.7 对齐 |
|---|---|---|---|
| 保存 / 已保存 | `save`/`saved` (zh:576-577) | browse custom source、DoH、ECH、命名模板 | 持久化 draft=保存 ✓ |
| 测试 | `imageSourceTest` (zh:294) | browse:85-119 | 实为"保存+测试"——W1 项，待改名或去副作用 |
| 清除 | `translateCredentialsClear` (zh:407) | credentials:131-156 | 破坏性动作无确认；settings-specific 补充词 |
| 恢复默认值 | `networkAdvancedReset` (zh:135) | network:264-281,356-360 | "重置"级——建议注册"恢复默认值/重置"词项 |
| 取消 | `cancel` (zh:581) | 各 dialog、更新下载取消、下载任务取消 | ✓ 只终止当前操作 |
| 确定 | `confirm` (zh:582) | 各确认 dialog | ✓ |
| 移除 | `removeAccount` (zh:217) | account:146-173 | ✓ 列表条目移除 |
| 取消屏蔽 | `unmuteTag/Author/Work` (zh:336-339) | muted:88-127 | **冲突**：§5.7 种子为"解除屏蔽"，"取消"义被占用 |
| 删除 | `historyDelete*` (zh:183-185) | HistoryPage（W6 页面） | ✓ 不可恢复级 |
| 重试 | `retry`/`retryDownload` (zh:424,578) | 加载失败、下载重试 | ✓ 同操作重试 |
| 刷新 | `refresh` (zh:579) | 路由快照、可达性手动刷新 | settings-specific：诊断快照刷新 |
| 检查更新 / 下载并安装 / 查看 | `aboutCheckUpdate`/`aboutUpdateDownload`/`aboutUpdateOpen` (zh:489,497,492) | about | 更新动作族——补充词 |
| 导出 / 导入 / 复制 | `backupExport`/`backupImport`/`copy`/`accountTransferExportTitle`/`aboutExportLogs` | backup/account/about | settings-specific 补充词 |
| 合并 / 覆盖 | `backupMerge`/`backupOverwrite` (zh:266-267) | backup:111-119 | settings-specific 补充词 |
| 开始记录 / 停止 / 开始探测 | `frameProbeStart/Stop`/`networkProbeRun` (zh:147-149) | 诊断页 | settings-specific 补充词 |
| 使用自定义相册 | `saveLocationUseCustomAlbum` (zh:352) | destination:85-113 | 实为 draft 保存——建议归"保存"语义 |
| 添加 / 添加账号 | `add`/`addAccount` (zh:513,215) | muted 输入、account AppBar | ✓ |

注：**l10n 中不存在"应用"键**（grep 全 arb）。§5.7 种子"一次性但不持久化的显式执行动作用'应用'"在设置场景的可能消费者：W1 定名后的镜像测试（若选"应用并测试"则属复合名）——需在术语补充中注册"应用"词项及适用边界。

## 20. 测试基线（已存在，迁移时须保持/扩展）

- `test/settings_test.dart`（1184 行）：SegmentedButton 质量选择 :629、settings primitives 结构 :688、读失败重试 :729/:749、根页路由顺序 :777、AccountCard 单路由 :819、导出确认流 :863、DoH 端点校验 :989、图源 preset/custom/invalid/实测 :1039/:1100/:1128。
- `test/settings_pop_repro_test.dart`（119 行）：设置写入不重置路由栈。
- `test/server_display_settings_test.dart`（389 行）：服务端偏好区。
- `test/download_tasks_page_test.dart` / `test/backup_test.dart` / `test/mute_store_test.dart` / `test/network_probe_test.dart`：相邻覆盖。
- 测试环境坑（quality-guidelines）：`material_ui` 遮蔽 SwitchListTile/SnackBar，widget test 须 pump material_ui 的 MaterialApp；`settingsProvider` 是 AsyncNotifierProvider——`.future` 失败永不 settle，断言 error 态用 listen。
