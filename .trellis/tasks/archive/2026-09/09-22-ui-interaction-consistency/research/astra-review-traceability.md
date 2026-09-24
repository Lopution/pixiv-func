# Astra UI 审查复核与追踪矩阵

## 1. 目的与方法

本文把 `/root/Astra.txt` 的审查建议对照当前仓库代码，区分已经吸收、当前仍可由静态代码确认、仅部分改善和必须运行后验证的内容，并为后续叶子任务提供唯一归属。

复核只读取了审查文本、当前代码、测试、Git 历史和已归档 Trellis 任务；没有运行 Flutter 应用。因此：

- Widget 树、事件处理、状态写入和路由后果可以标记为“静态确认”；
- 溢出、动画手感、真实焦点顺序、鼠标/键盘体验和设备性能不能标记为已复现；
- 后续叶子任务开始时仍须基于当时的 `main` 再定位，不能把本快照当作永久事实。

## 2. 基线

| 项目 | 值 |
|---|---|
| Astra 审查基线 | `main@f655e35` |
| 产品静态复核基线 | `main@8215fa85f58dd6834f5f2c503991349840842510` |
| 当前规划分支基线 | `origin/main@3c6c1bd130ae87058ac57c0987756538333b58e4` |
| 审查方式 | Widget 树与事件处理静态复核 |
| 已知限制 | 未运行当前 Flutter 构建，未做真机/桌面输入与视觉验收 |
| 前置任务 | `.trellis/tasks/archive/2026-09/09-11-ui-redesign/` |

自 Astra 基线以来，与报告直接相关的已合入变化：

- `d4b8e29` 合并 PR #48，统一 SnackBar 入口、位置与动效；该项记为已吸收，不再重复建设。
- `a5ab03c` 合并 PR #50，更新详情元信息排版与可选择 ID；多页作品的信息位置仍未解决。
- `f655e35..8215fa8` 的产品改动范围很小，绝大多数 Astra 项仍能在当前代码中定位。
- `8215fa8..3c6c1bd` 只有 `.github/workflows/ci.yml` 变更，因此规划基线前移不改变产品静态复核结论。

## 3. 计划工作包

以下编号表示父任务中的计划叶子边界；本轮只建立父任务，不创建或启动这些叶子。

| 编号 | 计划 slug | 主要范围 |
|---|---|---|
| W1 | `interaction-outcome-correctness` | 按钮名称与结果、返回优先级、进度恢复、测试/清除/取消、WebView 重试等确定性错误 |
| W2 | `discovery-query-context` | 推荐、排行、新作、搜索和以图搜图的分类与查询上下文 |
| W3 | `profile-bookmark-continuity` | 作者头部、作品分类、统计/链接/分享、收藏范围与资料编辑 |
| W4 | `artwork-viewer-series-flow` | 多页详情、下载/屏蔽、全屏查看器、动图和系列阅读 |
| W5 | `novel-reader-parity` | 在线/本地小说完整阅读舞台、设置与可操作进度 |
| W6 | `entity-management-consistency` | 小说/插画对象组件，历史、稍后再看、追更、本地书库和下载任务 |
| W7 | `comment-input-state` | 回复页滚动结构、回复目标、键盘/表情/贴图互斥与发送状态 |
| W8 | `settings-form-semantics` | 设置摘要、配置归属、即时/草稿/提交状态、诊断生命周期及设置自有表单/弹层/限宽迁移 |
| W9 | `onboarding-auth-content-layout` | 引导、登录、授权 WebView、Spotlight 及这些页面自有的 dialog/内容限宽迁移 |
| W10 | `motion-integration-acceptance` | 动效角色、reduced motion、跨工作包一致性和最终验收 |

W1 先于会再次触碰同一页面的 W2/W3/W4/W5/W8/W9；W4 作为触觉共享入口的首个消费者先于 W6/W7；W10 最后执行。其他工作包可以按依赖分批推进，但不得在同一工作树并行编辑同一文件。

## 4. Astra 编号项追踪

状态含义：`确认` 表示当前代码仍直接体现该问题；`部分` 表示已有一部分方向但验收未满足；`吸收` 表示当前 `main` 已有明确提交覆盖；`运行时` 表示静态代码不足以确认实际体验。

| Astra | 当前 HEAD 复核 | 状态 | 归属 |
|---|---|---|---|
| 1 推荐页 | `recommended_home_page.dart` 的顶部分类仍用 `FittedBox(scaleDown)`；小说和用户条目仍是页面/独立实现，刷新错误仍进入列表尾部结构。保留现有紧凑主体。 | 关闭 · unit-widget-tested（W2/W6；W10 C2/C6 复核） | W2；对象基座由 W6 提供 |
| 2 排行榜 | `ranking_page.dart` 使用 `RootSwipeSwitcher + TabSlideStack`，`novel_ranking_page.dart` 仍按索引直接替换正文；小说入口仍是插画榜 AppBar 图标；`IllustCard` 没有排名契约。 | 关闭 · unit-widget-tested（W2/W6；W10 C6 双注入复核） | W2 收敛切换与模式；排名变体及其排行页接入归 W6 |
| 3 新作页 | `new_page.dart` 的 `_selectorExpanded` 仍由重复点击当前标签切换，换范围/类型后收起；当前“范围＋类型”并非持续可见。 | 关闭 · unit-widget-tested（W2 已删 `_selectorExpanded`；W10 C3） | W2 |
| 4 搜索首页 | `search_page.dart` 的热门标签仍固定三列，并用 `tags.length - tags.length % 3` 丢弃不足整行内容；反向搜图和 Spotlight 仍是较强入口。 | 关闭 · unit-widget-tested（W1/W2） | W1 修内容丢弃，W2 收敛层级 |
| 5 输入/结果连续性 | 输入页已有内联 `SearchBar`、类型和筛选，但 `SearchResultPage` AppBar 仍只显示不可编辑关键词；无结果仍主要重试原查询；点击建议仍立即 `_submit()`。 | 关闭 · unit-widget-tested（W2；W10 C4 恢复等级核对） | W2 |
| 6 小说卡片 | `NovelRow` 与 `NovelCard` 仍是两份实现，封面尺寸、信息量、按压反馈不同。 | 关闭 · unit-widget-tested（W6） | W6 |
| 7 插画徽标 | `IllustCard` 已集中渲染徽标，但 R-18、AI、页数、动图仍存在不同容器尺寸/圆角；不得把统一扩大为固定裁切。 | 关闭 · unit-widget-tested（W6）；视觉等价人工面未验证 | W6 |
| 8 作者页关键操作 | 展开态与收起态菜单仍提供不同操作；`profile_header_delegate.dart` 在完全收起前通过 `IgnorePointer` 阻止已经可见的收起工具栏交互。 | 关闭 · unit-widget-tested（W3；W10 C10 语义复核） | W3 |
| 9 作者二级分类/统计/简介 | `user_page.dart` 仍用重复点击展开作品类型；社交地址只是 `SelectableText`；统计缺少完整可导航契约；作者分享仍先开选择对话框。 | 部分 in-flight（PR #79 系列统计口径）；其余 unit-widget-tested（W3） | W3 |
| 10 收藏标签上下文 | `BookmarkTagsPage` 默认公开且调用方未传入作者页当前范围；标签 feed 顶部主要显示标签名；分页尾部未完整区分可加载/加载中/失败；无本地过滤。 | 关闭 · unit-widget-tested（W3） | W3 |
| 11 资料编辑 | 显式返回始终走 `_attemptPop()` 确认，而系统返回只在有改动时拦截；保存位于长表单底部；头像/背景共用方形预览。 | 关闭 · unit-widget-tested（W1/W3；W10 C9 抽查） | W1 修返回，W3 完成表单体验 |
| 12 多页详情信息 | `illust_detail_page.dart` 手机布局仍先拼接全部 media slivers，再拼接 meta slivers；PR #50 改善文字但未改变信息位置。 | 关闭 · unit-widget-tested（W4） | W4 |
| 13 下载/屏蔽隐藏模式 | 图片长按仍切换下载模式；标签长按仍切换 block mode，随后普通点击语义改变。 | 关闭 · unit-widget-tested（W4） | W4 |
| 14 全屏查看器 | `image_viewer_page.dart` 仍以常驻 AppBar、分页与 `InteractiveViewer` 为主；无单击显隐、双击/复位、页码选择、保存/分享/信息，空列表仍可能形成 `1 / 0`。 | 关闭 · unit-widget-tested（W4；W10 C5/C10）；手势体感未验证 | W4 |
| 15 动图操作 | 播放/暂停差异合理；GIF 保存能力仍与下载模式和资源准备耦合，缺少持续可见的准备中/导出中/失败入口。 | 关闭 · unit-widget-tested（W4） | W4 |
| 16 系列目录 | `illust_series_page.dart` 仍是 header + 普通 `IllustFeedGrid`；打开页面即依据 latest content 更新查看游标；没有继续阅读、章节身份与真实阅读进度契约。 | 关闭 · unit-widget-tested（W4） | W4 |
| 17 在线阅读器 | `novel_page.dart` 的 `PopScope(canPop: !_chromeVisible)` 会让显式 `maybePop()` 先隐藏 chrome；信息弹层中的 `TagChip` 无动作。 | 关闭 · unit-widget-tested（W1/W5；W10 C9 补第三腿） | W1 修硬错误，W5 统一完整阅读体验 |
| 18 本地阅读器 | 页面仍是普通 AppBar + 信息行 + `NovelReader`；持久化 `read_offset`，但没有换算成 `initialAnchor`；未共用在线阅读设置/沉浸舞台。 | 关闭 · unit-widget-tested（W1/W5） | W1 修恢复，W5 完成 parity |
| 历史/稍后再看/追更/本地书库 | 历史仍用长按删除且固定两列估算预览宽度；稍后再看缺页面级管理；追更打开前更新游标；本地书库尾部主按钮仍是删除。 | 关闭 · unit-widget-tested（W6） | W6 |
| 19 回复页 | 根评论仍在可滚动回复列表之外；选择回复目标不主动聚焦；发送后只清文字，回复目标保留规则未明确。 | 关闭 · unit-widget-tested（W7） | W7 |
| 20 评论输入器 | 打开表情会收键盘，但输入框重新获焦不会对称关闭面板；网格固定列数；插入表情后不恢复焦点；失败保留草稿的现有行为应保留。 | 关闭 · unit-widget-tested（W7；W10 C7/C10） | W7 |
| 21 以图搜图 | `_cancelAndPop()` 在搜索中把“取消”实现为离开页面；各阶段替换整块正文；结果与空结果没有持续可达的图片/引擎任务头。失败态已有一部分更换/重试入口。 | 关闭 · unit-widget-tested（W1/W2；W10 C9 抽查） | W1 修取消结果，W2 完成任务上下文 |
| 22 Spotlight | 文章正文仍是全宽 `ListView`，段落不可选择，AppBar 无分享/打开原文；列表基本结构可保留。 | 关闭 · unit-widget-tested（W9） | W9 |
| 23 引导 | Welcome 已有滚动/限宽；语言和主题仍使用固定 `Column/Spacer`，并通过 `FittedBox` 缩字。 | 关闭 · unit-widget-tested（W9） | W9 |
| 24 登录帮助 | `login_page.dart` 的 `_help` 分支仍用剪贴板入口替换注册/登录主按钮，主任务会消失。 | 关闭 · unit-widget-tested（W9） | W9 |
| 25 手机/桌面 WebView | 两端致命错误“重新打开”仍只执行 `Navigator.pop(false)`；可恢复错误缺少明确 reload。 | 关闭 · unit-widget-tested（W1/W9） | W1 修语义/动作，W9 收敛布局 |

## 5. 设置、下载与全局规则追踪

| Astra 子项 | 当前 HEAD 复核 | 状态 | 归属 |
|---|---|---|---|
| 设置根页 | 已分组，但主题、语言、图像质量等入口仍缺当前值摘要；查看历史仍绕经配置页。 | 关闭 · unit-widget-tested（W8） | W8 |
| 主题/语言 | 单选页面可保留，主要缺一致选中态的运行时确认。 | 关闭 · unit-widget-tested（W8/W10）；选中态运行时面未验证 | W8/W10 |
| 浏览设置 | 自定义图源 `testMirror()` 仍先调用应用/持久化路径；日常偏好与图源优先级需重排。 | 关闭 · unit-widget-tested（W1/W8） | W1 修测试副作用，W8 重组 |
| 翻译设置/凭据 | 服务列表缺配置状态；清除只删存储未同步输入框；成功/失败状态色未区分。 | 部分 in-flight（PR #82 翻译摘要重探）；其余 unit-widget-tested（W1/W8） | W1 修清除，W8 完成状态语义 |
| 账号管理 | 当前账号、切换中与删除重量、服务端/本地偏好归属仍需显式表达。 | 关闭 · unit-widget-tested（W8） | W8 |
| 下载设置/保存位置 | 即时保存与草稿保存混用；用户界面仍暴露技术位置值，需展示可理解名称。 | 关闭 · unit-widget-tested（W8）；相册名草稿守卫 in-flight（PR #82） | W8 |
| 屏蔽管理 | 多类条目纵向铺开，解除动作主要是删除图标；失败输入恢复需回归。 | 关闭 · unit-widget-tested（W8） | W8 |
| 历史归属 | 本地/远程设置和内容入口分散，远程记录设置有重复入口。 | 关闭 · unit-widget-tested（W8） | W8 |
| 网络基础/高级 | 基础/高级分层已存在；高级字段分别保存，缺统一草稿/应用/重置语义和摘要优先级。 | 关闭 · unit-widget-tested（W8） | W8 |
| 备份恢复 | “覆盖”仍以更强实心按钮出现，尚未先选策略再统一确认。 | 关闭 · unit-widget-tested（W8） | W8 |
| 关于/更新 | 仓库地址没有动作；失败原因仍需按用户可采取措施区分。 | 关闭 · unit-widget-tested（W8） | W8 |
| 网络诊断 | 技术过程优先于服务结论；复制完整报告能力应保留。 | 关闭 · unit-widget-tested（W8） | W8 |
| 帧探针 | 页面 `dispose()` 仍停止录制，无法离开面板去目标页面采样。 | 关闭 · unit-widget-tested（W8）；PR #82 触碰部分 in-flight | W8 |
| 下载任务 | 组与子任务平铺；暂停任务仍映射到 retry 图标/动作；缺完成后查看结果与提交后快捷入口。 | 关闭 · unit-widget-tested（W6） | W6 |
| 弹层 | 收藏编辑与阅读设置密度、操作重量和提交时机不一致；失败保留草稿需成为共享契约。 | 关闭 · unit-widget-tested（W3/W5/W8/W9） | W3 收藏/资料；W5 阅读设置；W8 设置；W9 仅自有 dialog |
| 页面宽度 | 主壳已有 rail，但设置、文章和多处表单仍把内容全宽铺开。具体断点和溢出需运行时验证。 | 关闭 · unit-widget-tested（各 leaf；W10 宽度矩阵复核）；真实窗口缩放未验证；comment_item 390dp 溢出 49px 为新发现缺口 | W3/W5/W6/W8/W9 按文件所有权 |
| 动效 | 已有共享 MotionTokens 和 #48 反馈动效，不应重建 token；分类切换、头部交互、查询连续性和 reduced motion 仍需行为级验收。 | 关闭 · unit-widget-tested（W10 阶段2闸口+矩阵）；体感/性能未验证 | W10 |
| 视觉角色 | 现有主题/token 可复用；NovelRow/Card、旧弹层和辅助文字仍未完全按角色收敛。 | 关闭 · unit-widget-tested（各 leaf）；视觉等价人工复核未验证 | W3/W5/W6/W8/W9 按页面；W10 验收 |

本轮规划复核另增以下非 Astra 契约项，已在 `design.md` §5.6/5.7 与运行时矩阵登记，验收时视同条目处理：

- 触觉反馈分级（共享入口由首个真实消费者 W4 建立并接入，W6/W7 消费，W10 验收）—— 关闭 · unit-widget-tested；体感未验证；
- 屏幕阅读器代表路径与焦点顺序（TalkBack/Windows Narrator，W3/W4/W7 优先，W10 验收）—— unit-widget-tested（W10 三优先页语义树断言）；TalkBack/Narrator 未验证；
- 重复点击当前分类固定为回到顶部、不带刷新的约定（W2/W3）—— 关闭 · unit-widget-tested（W10 C3）；宽布局 NavigationRail 与 settings 缺口见台账 §8；
- 动作术语表（§5.7 种子随规划冻结，W8 只补充 settings-specific 词项，W10 核对）—— 关闭 · implemented（W10 C8 逐词核对+冻结测试）；`searchCancel` 偏离记录台账 §8；
- 查询上下文的进程死亡恢复等级声明（各 QueryContext 叶子）—— 声明已核对（W10 C4）；进程死亡恢复本身未验证。

## 6. 不重复建设与不直接采信项

- PR #48 的 SnackBar 统一不进入新实现范围，只在最终集成回归中检查没有被绕过。
- PR #50 的详情元信息排版保留；W4 只调整信息可达性和层级，不恢复旧标题布局。
- 不把 Astra 的“可能拥挤”“动效观感”等静态推断写成已复现缺陷；对应工作包必须用目标尺寸、字体缩放、键盘/鼠标和 reduced-motion 场景验证。
- 不为统一而建立平行组件家族；优先扩展 `lib/app/widgets/`、现有 feed 组件、主题和 MotionTokens。
- 断点、双栏与弹层优先复用 `lib/app/layout/app_breakpoints.dart`、`lib/app/layout/two_pane.dart` 和 `lib/app/motion/app_overlays.dart`；共享扩展必须与首个真实消费者同阶段落地。
- 不将“相同对象一致”误解为“所有列表相同”：排名、日期、进度、更新状态是受支持的命名变体。

## 7. 实施前重检清单

每个计划叶子开始前必须：

1. 记录当时 `main` SHA，并重新打开其拥有的文件与测试。
2. 检查本矩阵中是否已有条目被其他已合并任务吸收，更新状态而不是重复实现。
3. 将自己拥有的 Astra 子项拆成可测试的行为断言；不以截图相似度代替动作结果。
4. 明确与其他工作包共享的组件/状态契约，避免两个分支分别创造近似组件。
5. 将自动测试、桌面检查、模拟器和真机证据分开记录；缺失设备证据就保持未验证。

## 8. W10 关闭注记（2026-09-24）

- 逐项证据见 `.trellis/tasks/09-22-motion-integration-acceptance/acceptance-ledger.md`：
  §1 动效契约、§2 导航/re-tap、§3 反馈通道、§4 表单/返回/术语、§5 无障碍、§6 运行时矩阵、§7 已吸收修复、§8 有意取舍与已知偏离、§9 未验证面总清单。
- 上表「状态」列已由 W10 验收刷新为终态分级；「in-flight」行对应在途 PR #79/#82，不记绿。
- 未验证面（设备/桌面 runtime/AT/性能/体感/进程死亡/长翻译人工 pass/视觉等价人工复核）在台账 §9 全列，并由 PR body 同步披露——不由 widget 测试推断通过。
