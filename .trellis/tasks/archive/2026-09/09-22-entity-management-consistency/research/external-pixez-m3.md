# 外部对标：PixEz / Material 3 / Pixiv-Shaft

取证方式：`raw.githubusercontent.com` 拉取 PixEz master 源码（网络不稳定，`novel_card.dart` 未取得，以 Shaft 提交记录替代）+ m3.material.io 官方指引。

## 1. PixEz `lib/page/history/history_page.dart`（174 行，已取得全文）

- AppBar title 是**内联 `TextField`**：输入即按关键字过滤历史（标题/画师名），尾部 close 图标清空恢复（114-136）。→ 对应 §4.6 "历史按日期/类型帮助找回"：PixEz 选择了搜索而非分组头。
- `FloatingActionButton(delete)` → AlertDialog 确认 → `deleteAll`（138-173）。
- 网格 `max(2, width/200)` 列（46-51）；条目 `GestureDetector`：tap→详情（`HapticUtil.selectionClick()`），**long-press→AlertDialog→删单条**（`HapticUtil.heavy()`）（53-91）。
- 与我们相同：长按=删除确认；与我们不同：有搜索、有 FAB 清空（我们在 AppBar）、条目是裸图无标题。

## 2. PixEz `lib/page/novel/history/novel_history_page.dart`（已取得全文）

- 更简单：`ListTile(title, subtitle=userName)`，**trailing delete 图标无确认**直接删单条；FAB delete_forever + AlertDialog 删全部。
- 对我们的启示（反向）：单条删除无确认是 PixEz 的粗糙处；§5.3 destructive 要求确认边界或撤销，我们不抄这一点。

## 3. PixEz `lib/page/task/job_page.dart`（下载任务页，533 行，已取得全文）

- AppBar actions：简/详列表切换（`_itemSimple`，缩略图开关）、升降序 `sort`（旋转动画图标）、`more_vert` → bottom sheet 批操作：**retry_failed_tasks / retry_seed_task / clear_completed_tasks**（199-293）。
- 顶部 `SortGroup` chips 过滤：all / running / complete / failed（298-330）。
- 条目 `Card(r12)` + `InkWell`：**tap → `IllustLightingPage(id)` 打开作品详情**（368-376）——即"下载完成后查看结果"在 PixEz 就是条目主点击。
- 条目动作：refresh（重试）+ delete（删记录）两个图标；详模式额外 100×100 缩略图 + `CircularProgressIndicator` 覆盖层；底部 `LinearProgressIndicator`（377-521）。
- 状态模型比我们的窄（seed/running/complete/failed 四态），无暂停/取消语义——我们的九态机更完整，缺的是 UI 映射精度与"查看/移除"。

## 4. PixEz `lib/utils/haptic_util.dart`（已取得全文）

- 单一共享封装 `HapticUtil`：`userSetting.hapticFeedback` 开关 + 每档节流（50-150ms）+ 平台门（Android/iOS only）。
- 分级：selectionClick(50ms) / light(80) / medium(100) / heavy(120) / success(120) / warning(150) / error(150)。
- 用法分布：作品点击=selectionClick、长按手势=heavy、下载完成=success、二次确认=warning。
- **差异警示**：PixEz 给普通列表点击也加触觉；本路线图 §5.6 明确"普通列表点击不加触觉"——只借"单一薄封装+分级"结构，不借调用密度。

## 5. Pixiv-Shaft `recy_novel` 小说卡（来源：commit 2a1e7f9 消息，骨架对齐真实布局）

- 封面 `90×134dp` 圆角 `12dp`；卡内 padding `14dp`；卡左右 margin `12dp`；头像 `26dp`。
- 结构：左封面 + 标题两行 + 系列行 + 封面底部作者·日期 + 标签 chip 流。
- `cell_novel_v3` meta 行：红心收藏数 + 字数 + 日期（系列页专属）。
- 对 W6 的意义：Shaft 的小说卡信息密度远高于我们现有两种（series/tags/bookmark 都有槽位），但 §5.2 只要求"识别+基础动作+命名 variant 槽位"，不要求复刻全部字段。

## 6. Material 3 指引（m3.material.io）

**Selection / Lists：**
- 进入选择模式：**长按条目**（或点头像等快捷位）；再点选其它项；逐个点掉或点工具栏动作退出。
- 选中态呈现：check 图标 / checkbox / 表面变色，整个 list item 是一个选中单元。
- 桌面：选择是主活动时 checkbox 常驻；次活动时 hover 才显示、有选中后全显示。
- **多选条目不得有嵌套次级动作**（"can't have secondary nested actions"）——管理模式下的条目动作应全部收进 selection AppBar/底栏，条目本身不再挂 trailing 动作。
- 一个列表同时只能有一种 selection mode；长按+拖动可批量框选（若未占用拖动手势）。

**AppBar 切换**（Android 开发者示例）：选择态 `TopAppBar` 标题换为"Selected N items"、actions 换为批量动作、容器色换 `primaryContainer`——即 M3 contextual action bar 模式。

## 7. 对本包的可迁移结论

| 主题 | 外部做法 | 拟采纳度 |
|---|---|---|
| 历史找回 | PixEz 内联搜索框 | 部分：搜索是增强项；§4.6 只要求"按日期/类型帮助找回"，日期分组头+类型标识更轻 |
| 下载任务 | 点击=查看结果；状态过滤 chips；批量重试/清空 | 采纳：条目点击打开作品、（可选）状态过滤、批量操作进 AppBar/菜单 |
| 管理模式 | 长按进选择→AppBar 换计数+动作 | 采纳（M3 标准）；条目在模式下不挂次级动作 |
| 触觉 | PixEz 单封装+分级+节流+开关 | 采纳结构；密度按 §5.6（点击不加） |
| 小说卡 | Shaft 单卡槽位化（封面/标题/系列/meta/标签） | 采纳槽位思路；字段按 variant 命名启用 |
