# 外部对标：Material 3 / Android 设置模式

> 来源：m3.material.io、developer.android.com、source.android.com 设置指南（2026-09 检索）。
> 用途：为 §4.8 的 immediate/draft/action/destructive 分类、摘要行、层级结构提供平台惯例依据。

## 1. 即时 vs 草稿：控件类型决定提交时机（M3 Switch guidelines）

- **Switch 必须即时生效**："Switches change settings and other options immediately… without needing to save"；反模式："Avoid using a switch to select multiple options that require people to save — switches should be immediate"。→ 本仓库所有 `SettingsControl`/SwitchListTile 应永远是 immediate；与 §5.7"immediate 控件不显示应用/保存"一致。
- **单选列表（radio/check 行）同样即时**：M3 settings pattern 里单选 subscreen 用行选中态表达当前值，点选即生效——主题/语言/翻译服务商/网络模式的 ListTile+check 现状正确，保留。
- **文本/多字段表单走显式提交**：内容设计惯例（settings save patterns 综述）把 auto-save 限定于"简单、非破坏、即时可见效果"的项；需要校验、批量或可审阅的表单用 explicit save。→ 自定义图源/相册名/命名模板/DoH/ECH/凭据 = draft+保存，与 §5.3 吻合。
- **辅助规则**：draft 表单离开有 dirty 确认义务（M3 full-screen dialog 规则："If the user has made any changes, they are prompted to confirm the discard action"）——设置页内嵌表单可借用同一规则：dirty 时 pop 前确认。

## 2. 摘要行：title 下显示当前值是 Android 一级惯例

- Android Settings 设计指南（source.android.com settings-guidelines）："Below the title, show the status to highlight the value of the setting. Show the specific details instead of just describing the title."——**summary 是设置项的一等要素**，"glance at settings screens and understand all of the individual settings and their values"。
- Preference 库机制（developer.android.com customize-your-settings）："A `Preference` that persists data must display the current value in its summary"；`ListPreference`/`EditTextPreference` 自带 `SimpleSummaryProvider` 把已存值渲染为 summary，未设置显示 "Not set"。
- → 对本包的映射：`SettingsTile` 需要 summary 槽位；根页配置入口显示当前值（主题=跟随系统、语言=简体中文、翻译=关闭/百度+凭据态、浏览=图源+质量、网络=模式、下载=预设+位置、历史=本地/远端开关态）；纯内容入口（稍后再看等）不需要值摘要但可以有计数/状态；"未配置"类显示对应 "Not set" 惯例（现有 `imageSourceCustomUnset`="未配置" 已符合）。

## 3. 信息层级与分组（Android settings pattern）

- 顶层是**概览页**："Users can quickly see the most important and frequently used settings and their values"；≥15 项应分 subscreen（本仓库已分）。
- 分组惯例：PreferenceCategory/section header；**依赖关系**：父开关控制子项可用性，不可用项给出原因（"Place a dependent setting below the setting on which it depends"）。→ 翻译凭据入口依赖服务商选择（现状已条件渲染 :51-63，可加"已配置/未配置"态）；屏蔽页的 hideMuted 依赖屏蔽集存在。
- 标签规则：title 简短、非否定式（"Don't/Never"→"Block"）、第二人称；本仓库现状基本符合。
- **大屏**：PreferenceHeaderFragmentCompat/SlidingPaneLayout 提供 list-detail 两窗格；但 §4.8 对本包只要求"限宽"（form/settings 角色），不要求双栏——settings 列表采用单列+限宽即可，勿过度设计。

## 4. 破坏性/确认模式（M3 Dialogs + confirmation patterns）

- 确认 dialog：确认按钮用**具体动词**（"Send/Create/Delete"，避免 Done/OK/Close）；dismissive 在左、confirming 在右；最多两个动作；"Dialogs should contain a maximum of two actions"——**备份策略 dialog 目前三个动作（取消/合并/覆盖）违反此条**，印证 §4.8"先选策略再统一确认"的两步改造：第一步选择（radio/选项，允许无 confirm 按钮禁用态：M3 "Disable confirming actions until a choice is made"），第二步单确认 dialog。
- 危险确认：title 复述动作后果（"Delete history?"）；破坏性主按钮可用 error/danger 色表达，但**不应靠"权重最大"吸引点击**——现 backup dialog 里 overwrite=FilledButton、merge=TextButton 属反向引导（更强破坏拿更强按钮）。
- 可逆操作不需要确认（"Confirmation isn't necessary when the consequences are reversible"）——屏蔽条目解除（可再屏蔽）不需要确认 dialog，但动词应改"解除屏蔽"；真正不可恢复的（清除凭据、覆盖导入、移除账号、清历史）需要确认边界。

## 5. 行动作（action）的状态表达惯例

- busy：按钮内 spinner 或 LinearProgressIndicator（本仓库 update 区已用后者 :269；probe/mirror test 用按钮内 spinner :212-218,:263-268——惯例已存在，推广即可）。
- 成功/失败：瞬时 snackbar（#48 已统一）+ 需要留存的结果就地显示（凭据页 `_status` 行——需双色化）；action 结果若改变可见状态应同步 UI（W1 的"清除同步 UI"项即此惯例）。
- 诊断页的"先摘要后细节"：Android 设置概览逻辑同构——结论=summary 先行（如设置首页每行显示状态），技术明细折叠至次级（对应 probe 页 per-host card 内 steps 默认收起/或摘要卡在上、明细卡在下）。

## 6. 对本包的直接推论清单

1. `SettingsTile` 增加可选 `subtitle`（summary）——对应惯例 §2。
2. Switch/单选行全部 immediate，不挂保存按钮；draft 控件集中在"输入区+保存按钮"结构，dirty 离开确认。
3. 备份导入改为两步：策略选择（等权选项，confirm 在选定前禁用）→ 统一确认层（两个策略共用同一确认 dialog 结构与权重）。
4. 解除屏蔽 verb 对齐"解除屏蔽"；可逆，无需确认。
5. 凭据清除、覆盖导入、移除账号 = destructive：对象+后果+确认。
6. Probe 页：conclusion/建议先行，steps 明细次级。
7. 宽屏：单列限宽（form/settings 角色），非双栏。
8. "未配置/Not set"摘要惯例沿用 `imageSourceCustomUnset` 模式。
