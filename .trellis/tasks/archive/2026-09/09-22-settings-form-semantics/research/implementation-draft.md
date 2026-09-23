# W8 实现草案：逐条对应 design.md §4.8

> 基线 `main@8067b2d`；前置 W1 未合入（planning），W8 须在其镜像测试/凭据清除/返回语义修复之上 rebaseline。
> 原则：只迁设置/诊断页；复用 `AppBreakpoints`、`showAppDialog`/`showAppBottomSheet`、`SettingsTile/Section/Control`、`persistSettings`、`showAppSnackBar`；扩展共享件必须与同包真实消费者同 stage 落地（§4.11/§5.5）。

## 公共件改动（先于各页）

1. **`SettingsTile` 增 `subtitle`（可选 Widget/String）**——摘要行载体（external §2 惯例）；首个消费者=根页，同一 commit 接入。`SettingsTile` 仅 settings feature 消费（grep 验证无外部使用），签名演化安全。
2. **设置内容限宽包装**：所有 settings 页 `ListView` 外包 `Center + ConstrainedBox(maxWidth: N)`。建议 N=600（与 `AppBreakpoints.medium` 对齐，"可读栏宽"语义）；全宽度生效（窄屏时约束不收紧布局，宽屏收栏）——比"仅 ≥1200 才限宽"更符合 form/settings 角色，且实现无断点分支。决策点：N 值与是否门控（见 risks.md R3）。
3. **`persistSettings` 收敛**：删 `network_settings_page.dart:22-35` `_persistNetwork` 副本与 `account_settings_page.dart:182-197` `_write` 副本差异，统一走 `persistSettings`（可接受 messageKey 参数化）。文件内改动，低风险。
4. **dirty 离开确认**：draft 表单统一 `PopScope`/`canPop` 守卫（参照 `ProfileEditPage` 的 PopScope 契约，component-guidelines Predictive Back 节）；无改动直接返回不叫"取消"（§5.7）。
5. **action busy/success/error 统一形**：按钮内 spinner（现有 probe/mirror/update 先例）+ snackbar 终态 + 就地结果（双色）。不新建封装，各页内联实现已够用；若 ≥3 页出现同形 busy 按钮再考虑提取。

## §4.8-1 根页显示关键当前值，内容/配置入口归属清楚

`settings_page.dart`：
- `SettingsTile` 全部补 summary：
  - 账号设置：当前账号名（或"未登录"）
  - 主题：跟随系统/明亮/黑暗（themeCode→l10n dark/light/system）
  - 语言：languageTag→显示名（zh-CN→简体中文…，可复用 `_items` 表外提）
  - 翻译：服务商名 + 凭据配置态（"已配置/未配置"，需 `TranslationCredentialsStore` 轻量存在性读法——新增 `hasBaidu()/hasLlm()` 或 read 判空，不读 secret 值）
  - 浏览：当前图源名（`ImageSourceMode`→label；custom→已存前缀；auto→winner 或"自动"）
  - 屏蔽：条目计数（tags+users+works 合计或分述）
  - 历史：本地/Pixiv 开态（"本地开 · Pixiv 开"）
  - 网络：当前模式名
  - 下载：命名预设 + 保存位置类型（human 名，见 §4.8-5）
  - 下载任务：活动任务数（读 `downloadManagerProvider.tasks`，注意其非 Riverpod 态——摘要用一次性读或 `changes` 流 listen；可降级为无摘要）
  - 备份：无自然当前值→静态 hint 或不带摘要
  - 关于：版本号（复用 PackageInfo——注意是 Future，可用 FutureBuilder 或现有缓存）
- 归属：现有五组（外观/浏览/我的内容/网络与下载/数据）+开发者已合理；把"历史"根 tile 的 summary 与 `/settings/history` 配置页关系写清（内容入口 `openHistory` 是否上根页——Astra"查看历史仍绕经配置页"，建议根页历史 tile 直接进 `/settings/history/view` 或保留进设置页但摘要显示开关态；决策点 R5）。
- AccountCard 保持。

## §4.8-2 主题/语言保留简单单选

- `theme_settings_page.dart`、`language_settings_page.dart` 不动结构；补：选中态双通道（check 图标 + Semantics `selected`/文字"当前"），满足 Astra"一致选中态"运行时项；限宽包装。

## §4.8-3 日常浏览偏好优先，图源/网络配置归组

`browse_settings_page.dart` 重排：
- 新顺序建议：浏览偏好组（pixivHistory 或移走——见 §4.8-4、blockR18/blockAI/hideMuted）→ 质量组（preview/detail/view SegmentedButton）→ 外观组（reduceMotion）→ 图源组（preset 列表 + custom 输入 + 保存/测试）。
- custom 输入区：draft 语义显式化——dirty 离开确认；保存按钮文案保持"保存"；测试按钮按 W1 落地结果消费（若 W1 改为纯测试无持久化，则恢复"测试"名与语义；若保留副作用则名"应用并测试"——**W8 不替 W1 决定**，只在 implement.md 写两种接法）。
- pixivHistory 开关在 browse(:313-322) 与 history_settings(:38-47) 重复——归并到 history 页（浏览页移除或改为摘要链接），单一 owner。

## §4.8-4 翻译/账号/下载命名/保存位置/屏蔽/历史/网络高级：FormState 分类落地

逐页现状→目标（分类细节见 codebase-settings-pages.md）：

- **翻译选择页**：服务商单选=immediate（保持）；凭据入口行摘要="已配置/未配置"。
- **凭据页**（draft+destructive）：
  - `_clear` 加 `showAppDialog` 确认（对象=凭据、后果=需重新输入才可翻译）；
  - 成功后清空输入框（消费 W1 修复；若 W1 未落地此页相关项，W8 顺手补 controller.clear()——verify W1 scope）；
  - `_status` 成功/失败双色（primary/error）；
  - dirty 离开确认；保存中按钮 spinner。
- **账号页**：切换账号加 busy（行内 spinner 或整行 disabled while switching——`switchAccount` 是 async action）；删除保现有确认 dialog；"当前"check 行不可点（已有）+ 语义标签；服务端偏好区保留 section 分隔，标题/hint 已够，可加"由服务器保存"徽记非必须。
- **下载设置页**：
  - Slider：现状 onChangeEnd 提交=immediate-with-preview，保留但语义写明（"拖动预览，松手生效"）；或改 draft（应用按钮）——建议保 immediate（低风险），`onChangeEnd` 失败回滚已存在 :194-197。
  - 命名预设 immediate；custom 模板 draft：invalid 时保存按钮**禁用**而非静默 return（:152），errorText 已有；dirty 离开确认。
- **保存位置页**：见 §4.8-5。
- **屏蔽页**：解除动词改"解除屏蔽"（图标可考虑 `Icons.visibility_off_outlined`/undo 类替代 delete_outline，或保图标改 tooltip——决策点）；`_addTag` 失败恢复输入（把 `_controller.clear()` 移到成功后）；pending 行可视（trailing spinner 而非仅 disable）；`_SectionHeader` 换 `SettingsSection`。
- **历史设置页**：两开关 immediate；`historyView` 入口摘要=当前开关态；pixivHistory 重复入口归并后此处为唯一。
- **网络高级页**：DoH/ECH 两 draft 字段合并为**一页级 draft**：单一"保存"按钮提交两字段（或保留分别保存但统一为"编辑→保存/放弃"的草稿区）；**"恢复默认值"前加确认**（destructive-lite：重置两个已存值）；dirty 离开确认覆盖两字段。
- **action 语义总表**（idle/busy/success/error）：备份导出/导入、镜像测试、更新检查/安装、探测运行、日志导出、帧探针录制——每处确认四态可区分（现状基本齐，缺：账号切换 busy、unmute pending 可视、凭据 status 双色）。

## §4.8-5 保存位置主视图人类可读名，技术 URI 下沉

`download_destination_page.dart` + `download_settings_page.dart::_destinationText`：
- SAF 已选 tile :126-131 的 `safTreeUri` 原始串下沉：主标题显示人类名（解码 `Uri.parse(uri)` 末段：tree URI 形如 `.../tree/primary%3ADownload%2Fpixiv` → "内部存储/Download/pixiv"；primary→"内部存储"，sd 卡卷标→"SD 卡"）；
- 纯 Dart 解码即可（无需新平台通道）；raw URI 放次级（展开详情/长按复制/或 subtitle 小字截断）；
- `_destinationText`（download_settings:201-209）safFolder 分支同步显示人类名；
- 相册分支摘要：自定义相册名已显示 :206；内置相册="PixivFunc 相册（默认）"已有。
- 风险：tree URI 格式厂商差异（`primary:`、`XXXX-XXXX:` 卷标）——解码失败回退显示 URI 末段原文。

## §4.8-6 备份先选策略再统一确认

`backup_settings_page.dart::_pickStrategy` :97-123 改两步：
- Step1：`showAppDialog`/`showAppBottomSheet` 呈现文件内容摘要（现有 prompt :104 信息量保留）+ **等权策略选择**（RadioListTile 式两项或两个同权按钮；confirm 在选定前禁用——M3 惯例）；
- Step2：同一确认 dialog（取消/确认两键，title 复述后果：merge="将添加 N…"，overwrite="将清空本地历史并替换…"+服务端只增 note :104 已有）；
- 两策略共用同一确认层级与按钮权重——消掉 overwrite=FilledButton/merge=TextButton 的不对称。
- 备选：单 dialog 内 radio + 确定（两步合一），只要确认按钮对两策略同权即满足"同一确认层级"。
- `BackupService.apply` 与策略枚举不动（core 契约已完备）。

## §4.8-7 仓库链接可打开/复制 + 更新错误按可行动原因区分

`about_settings_page.dart`：
- `aboutSource` tile :94-98 加动作：主点击 `launchUrl`（`url_launcher` 已依赖，spotlight_article_page.dart 有先例）打开 `https://github.com/Lopution/Pixiv-func`，次级（trailing icon 或长按）`Clipboard.setData` 复制 + snackbar。
- 更新失败拆分 `statusText` :231-243：`offline`→"检查网络后重试"、`rateLimited`→"稍后重试（GitHub 限流）"、`invalid`→"清单无效（请反馈）"、`busy`→"已有更新任务进行中"、`failed`→通用失败；`_applyResult` :244-251 同理（permissionRequired/installStarted 已有专文，failed/canceled 分开）。新增 key 走四语 l10n。

## §4.8-8 迁移纪律（复用与扩展）

- 限宽/弹层/dirty 守卫全部按上文公共件接入，**不新建跨 feature shell**；`SettingsTile.subtitle` 扩展随根页消费者同 commit。
- 无 `showModalBottomSheet` 现存调用；若备份策略选择用 bottom sheet 则走 `showAppBottomSheet`（首个消费者=备份页）。
- 不消费 W4 触觉 owner（若已合入可给 destructive 确认加约定级震动——可选，非范围必须）。

## §4.8-9 术语迁移（§5.7 种子 + settings-specific 补充）

种子对齐（改文案不改语义）：
- `unmuteTag/unmuteAuthor/unmuteWork` zh："取消屏蔽*"→"解除屏蔽*"（"取消"保留给终止当前操作）；
- `saveLocationUseCustomAlbum`"使用自定义相册"→"保存"（draft 持久化动词统一）；
- `imageSourceTest` 按 W1 结果对齐（"测试"无副作用 / "应用并测试"）；
- `translateCredentialsClear`"清除凭据"保留"清除"动词 + 确认 dialog。

settings-specific 新增词项（**不改变种子语义**，建议在 implement.md/术语附表登记）：
- "应用"：一次性非持久化执行动作（本包内可能无消费者则仅注册，或随 W1 镜像测试落地）；
- "恢复默认值"：重置字段组到默认（网络高级）；
- "解除屏蔽"：屏蔽项的可逆移除（种子已含，登记用法）；
- "导出/导入"：备份与凭据/日志文件操作；
- "合并/覆盖"：备份导入策略对；
- "检查更新/下载并安装/查看"：更新动作族（"查看"符合种子"查看"类）；
- "开始记录/停止（录制）"、"开始探测"、"复制（报告）"：诊断动作族；
- "已配置/未配置"：凭据状态摘要词；
- "正在切换"类 busy 文案（账号切换）。

## §4.8-10 诊断先摘要后细节 + 帧探针生命周期分离

`network_probe_page.dart`：
- 顶部新增**总览区**：全部 host 的结论汇总（各 conclusion 计数/最劣结论 + 一句可行动建议，如"建议模式：ECH"）；`_ConclusionBadge` 复用。
- per-host card 改为摘要行（host+badge+firstError）+ 明细折叠（ExpansionTile/展开按钮，steps 默认收起）；复制报告保留（现有 `toCopyableText` 已含 env header）。
- 状态留存限制：报告不持久化（spec 要求），离开即丢保持——在页内 hint 或文档写明。

`frame_probe_page.dart`：
- `dispose` 不再 `stop()`；`_recording` 从 `FrameProbe.instance.recording` 读（已是）；
- 页内常驻"录制中 · N frames"状态条 + 停止按钮；hint 更新为"可离开本页去目标页面滚动，回来停止并复制报告"；
- （可选）录制中在设置根页 tile 摘要/全局 snackbar 给指示；
- `_frames` 无上限问题登记 PRD（建议实现时加 cap 或文档化"长录制内存增长"）。
- core `frame_probe.dart` 语义不动（start/stop/report 签名保持）。

## 验收映射（implement.md §6 W8/W9 门禁）

- immediate/draft/action/destructive 各一个代表测试：immediate（开关写失败 snackbar+值不变）、draft（custom 模板 dirty 离开确认+invalid 禁用保存）、action（备份导入 busy→done 计数）、destructive（移除账号/凭据清除确认流）；
- 宽度矩阵 320/390/600/840/1200 + 横屏 + 1.3x 字号：限宽栏内主操作可达；
- mobile sheet/desktop dialog 语义顺序一致：备份策略选择与确认在两种呈现下动作序相同；
- 术语：按 §5.7+补充表核对全部新文案；`settings_test.dart:522` 四语齐全测试兜底新 key。

## 现有测试必须保持/更新

- `settings_test.dart` 根页路由顺序 :777（加 summary 不破）；导出确认流 :863；DoH 校验 :989；图源 :1039/:1100/:1128；
- `settings_pop_repro_test.dart`（设置写不炸路由栈）；
- `download_tasks_page_test.dart`、`backup_test.dart`、`mute_store_test.dart`、`server_display_settings_test.dart` 联动回归。
