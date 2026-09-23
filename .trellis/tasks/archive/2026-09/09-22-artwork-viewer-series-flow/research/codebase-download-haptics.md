# 代码调研：下载/保存入口全景 + 触觉现状审计

对应设计 §4.4/§5.6：W4 首次落地唯一触觉 owner，首个消费者是“下载/保存”。行号已核对。

## 1. 触觉现状（审计结论）

- `grep -rn "HapticFeedback\|haptic" lib/` → **产品代码零命中**；全仓库唯一出现是父任务 design.md 的需求文本。
- 即：当前没有任何触觉调用、没有包装层、没有设置开关。W4 需要从零建 owner，**不存在需要收敛的旧实现**——这比“替换散点调用”简单，但要防止各 feature 各自再写 `HapticFeedback.xxx` 直连。
- `material_ui` 包装层不含触觉导出；`HapticFeedback` 来自 `flutter/services.dart`。

## 2. 下载/保存入口清单（首个消费者候选触点）

### A. 详情页
- **AppBar 下载全部**（illust_detail_page.dart L167-185）：仅 `_downloadMode` 时出现；`download.downloadAll(entity)` → 成功 `downloadQueuedMessage` / 失败 `downloadSubmissionFailed(error)` snackbar。
- **下载模式入口**：图片/Ugoira/双栏 pager 长按 → `_toggleDownloadMode`（L291/312/336/409）；退出=点空白（外层 GestureDetector L398-402）。
- **单页下载角标**（page_image.dart）：`_DownloadBadge`（L294-325），`onTap` L232-：先 `setState(_optimisticDownloading = true)`（L238）乐观置位，`showAppSnackBar` 报错（L239-248）；`state == downloading` 时角标禁用（L325）。
- **下载态判定**（page_image.dart L115-132）：`download.stateFor(illustId, pageIndex)` + `manager.tasks` 双源——任务 status queued/running/finalizing/canceling ⇒ 活跃；failed/canceled/retryable/succeeded/orphaned ⇒ 非活跃。注释 L133-136：PixivImage 换 URL 保留上一帧，预览→详情→原图三档切换无闪断。

### B. 卡片/列表
- `illust_card_actions.dart` download 动作（L71-96）：`downloadAll(entity)` + snackbar；从 `showCardActionSheet`（illust_card.dart L173/186）触达。

### C. Ugoira
- `ugoira_viewer.dart` L230-243：导出 GIF IconButton，running/finalizing 显示 spinner，结果走 snackbar。

### D. 核心协调层 — `lib/core/download/illust_download_coordinator.dart`
- `downloadPage`（L22-34）：构造 typed `DownloadRequest` → `_manager.submit`，**返回去重后的 snapshot；空/不可下载 URL 抛错而非静默成功**（L20-21 注释）。
- `downloadAllPages`（L41-58）：批量 `submitGroup`，活跃任务重复提交会 dedupe 到同一任务（L40-41 注释）。
- `downloadAuthorWorks`（L89-98）：作者全作品分组提交。
- `_exportCaption`（L149-152）：**提交时**顺带导 caption 侧车文件，仅 illust-page 请求；dedupe 命中则静默跳过。
- provider 接线：L172 `captionExporter: ref.watch(captionExporterProvider)`。

### E. DownloadManager 公共面 — `lib/core/download/download_manager.dart`
- 事件流 `events`（L101）、变更流 `changes`（L105）、快照列表 `tasks`（L108）；
- `pause/cancel`（L223/256）、`pauseGroup/cancelGroup`（L312/333）、`recover`（L345）、`flushPersistence`（L553）；
- sink 由 `downloadSinkFactoryProvider`（download_providers.dart L32）注入；Android 走 MediaStore（platform_caps.dart L39 `supportsMediaStore`）。

## 3. 反馈语义现状（触觉要嫁接的位置）

| 动作 | 当前反馈 | 建议触觉级（§5.6 映射） |
|---|---|---|
| 卡片/详情 tap 进详情、tag tap 搜索 | 无 | none（普通列表 tap 不振） |
| 下载入队成功 | snackbar“已加入队列” | `selectionClick` 或 light（一次性确认） |
| 下载提交失败 | snackbar 带 error | `heavyImpact`（失败/危险需确认级） |
| 进入下载模式（长按） | 无反馈，模式位翻转 | `selectionClick`（进入管理/选择模式） |
| 批量下载提交 | snackbar | `heavyImpact`（保存/发送成功级） |
| Ugoira 导出完成/失败 | snackbar | 成功 `selectionClick`、失败 `heavyImpact` |
| 书签心切换 | 动画 | `selectionClick`（toggle 级） |

§5.6 约束：**触觉是冗余通道**——视觉必须先自足；包装层放 `lib/app/`（建议 `lib/app/haptics/` 或并入现有 app 工具目录），API 至少含 `selectionClick`/`heavyImpact` 分级；不接第二个平行实现。

## 4. 与 PixEz 的差异（详见 external-pixez-haptics.md）

- PixEz `HapticUtil` 有 settings 开关 + 最小间隔节流 + try/catch 平台异常；W4 薄包装应保留这三件事的最小等价（开关可挂 settings_controller 已有通道）。
- PixEz 在保存成功用 heavy、确认用 light；本项目 §5.6 分级表与其一致，可直接映射。
