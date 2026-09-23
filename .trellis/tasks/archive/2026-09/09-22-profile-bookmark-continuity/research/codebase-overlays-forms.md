# Codebase: 资料/收藏编辑弹层现状与 FormState 差距

## 1. Overlay 入口现状

`lib/app/motion/app_overlays.dart`：`showAppBottomSheet`（L9-31，isScrollControlled/showDragHandle/useSafeArea 参数，MotionTokens.sheet + reduced-motion noAnimation 门）、`showAppDialog`（L36-53，barrierDismissible/barrierLabel/useRootNavigator，MotionTokens.dialog）。

**W3 域内所有弹层已经走这两个入口**——不存在裸 `showModalBottomSheet`/`showDialog` 待迁移：

| 弹层 | 入口 | 位置 |
|---|---|---|
| 作者作品批量下载 | showAppDialog | user_page.dart L260-266 → AuthorWorksDownloadDialog |
| 资料分享 | showAppDialog | user_page.dart L544-581 |
| 资料编辑离开确认 | showAppDialog | profile_edit_page.dart L146-162 |
| 收藏编辑 sheet | showAppBottomSheet(isScrollControlled) | bookmark_switch_button.dart L45-57 |
| 关注范围 sheet | showAppBottomSheet | follow_switch_button.dart L36-114 |

所以「迁移」实际内容是：**FormState 语义收敛 + 宽屏断点/限宽 + 控件一致性**，不是换入口。

## 2. ProfileEditPage（lib/features/profile/profile_edit_page.dart，576 行）

### 返回契约（W1 消费点）
- `PopScope(canPop: session == null || !hasUnsaved)`（L175-179）：**系统返回已经只在 dirty 时拦截**，符合契约。
- `_attemptPop`（L137-167）：**显式返回无条件弹确认框**——无改动也弹。Astra #11/W1 修复点：`hasUnsavedChanges`（controller L65 = `draft?.hasChanges`）为 false 时应直接 pop。W3 不动判定规则，只在 W1 契约上做布局（design §4.3：「W1 先落地资料编辑的返回判断」）。
- 显式返回按钮 tooltip 是 `l10n.cancel`（L183-187）——§5.7：无改动返回不叫「取消」，W1/术语表核对时注意。
- `_HeaderBackButton` 的 latched-canPop 先例（profile_header_delegate.dart L632-657）可复用思路但 profile edit 是普通 AppBar leading，不存在该问题。

### 表单结构（_ProfileEditBody，L231-457）
- draft 语义：已有完整 draft 模型——`ProfileDraft.dirtyFields/hasChanges`（profile_edit_models.dart L211-223）、`buildPatch` 只发 dirty 字段（L225-241）、状态机 `loading/ready/submitting/confirmed/verificationPending/failure/canceled`（profile_edit_controller.dart L16-19+）、`fieldErrors`/`currentPasswordError`、capabilities 逐字段禁用。
- 文本控制器 `_syncText`（L276-282）只同步首个非空 draft；后续 draft 变化不回写（合理——用户编辑优先）。
- `_submit`（L301-306）：`controller.submit(currentPassword:)` + 清密码；成功/失败经 state.failure/`_Notice` 呈现（L329-341），snackbar 无——失败只靠表单内 Notice，成功无显式 snackbar 只有 confirmed Notice。
- **保存入口位置**：`FilledButton.icon` 在 ListView 末尾（L424-433）——长表单需滚到底。§4.3「固定保存入口」→ 应上移到 AppBar actions 或 bottomBar 固定区。
- **图片预览**：`_ImageField`（L459-508）头像/背景共用 54×54 方形预览（`PixivImage.avatar(...size:54)` / `Image.file` L478-489）。§4.3「头像/背景各自真实预览形态」→ 头像应圆形（PersonAvatar 已有 ring 变体）、背景应宽条（对照 header 的 `PixivImage.detail` cover 形态）。
- `requiresCurrentPassword` 时追加密码框（L411-422）。
- 整页 ListView 全宽（L321-322 padding 16）——宽屏无限宽；§5.5 form 限宽按 AppBreakpoints（compact 600 / medium 600 / expanded 1200，app_breakpoints.dart L5-27）由本包接入。
- `PopScope`+`Scaffold.resizeToAvoidBottomInset` 未显式设置——长表单+键盘在 profile edit 是否压缩页面需运行时核对（component-guidelines 常见错误清单有相关条目）。
- 初始化坑（L114-125 注释）：autoDispose controller 需 `listenManual` 保活——新布局若拆分 widget 注意不要重新引入 zero-listener 问题（quality-guidelines L199-211）。

### 测试现状
`test/profile_edit_test.dart`：patch 构建、校验、verificationPending 不提交、账号切换不提交、cancel 释放图片、**「renders beta56 field order and asks before leaving dirty form」（L415，widget 测试已钉住 dirty 离开确认）**、autoDispose 保活（L483）。改返回行为会碰 L415 的测试预期。

## 3. 收藏编辑 sheet（_BookmarkEditSheet，bookmark_switch_button.dart L159-535）

- 触发：`BookmarkSwitchButton` 长按（L105-107，bookmarked 与否都进 sheet；新建/编辑两态）。
- 容器：透明底 `showAppBottomSheet`+自绘圆角 24 Container，minHeight=0.35h/maxHeight=0.75h（L235-247）。
- 状态：`_restrict`（默认 public）、`_tags`、`_prefilled`（L177-180）；已收藏时 `bookmarkDetailProvider` 到达后回填 restrict+tags（L209-220），回填失败显示错误+重试（L301-323），未回填前 confirm 禁用（L431-432 注释：防覆盖私有/标签）。
- 控件差异：
  - 范围选择用 `_RestrictSelect` **DropdownButton** 药丸（L470-535）——而 follow sheet（follow_switch_button.dart L74-88）与 BookmarkTagsPage（L37-51）用 `SegmentedButton`，三处不一致。
  - 取消/确认用 `MaterialButton`+`headlineSmall` 文本（L391-462）——与全局 FilledButton/OutlinedButton 惯例不一致。
  - 「取消」直接 `pop()`（L415），**dirty（改了标签/范围）不确认**——§5.3 draft 模式要求离开按 dirty 确认；「确认」pop 后 `addWithRestrict`（L433-444，非 awaited——sheet 已关，失败经 BookmarkStore error → 按钮自身 snackbar 通道 L71-81）。注意确认时把 `_tagInput` 未提交的残文也并入 tags（L438-443）。
- sheet 无 `useSafeArea`/dragHandle；键盘弹出时 `isScrollControlled:true` 会顶起，0.75h 上限可滚动，行为基本可用但宽屏下 sheet 全宽（showAppBottomSheet 不传宽，Material sheet 默认全宽）——桌面/宽屏需考虑限宽（§5.5 弹层语义：同信息顺序，位置可变）。
- `userBookmarkTagSuggestionsProvider`（bookmark_tag_providers.dart L19-31）：只取第一页 tag 作建议。

## 4. AuthorWorksDownloadDialog（lib/features/profile/author_works_download_dialog.dart，174 行）

- FormState=**action**：`_Phase {enumerating, confirming, failed}`（L30）；CancelToken 随 dispose 取消（L47-50）；确认提交失败落到 failed 态而非静默关（L97-103）；确认 pop 回传 `group.childCount`（L96）。
- 语义基本符合 §5.3 action 型；缺口：无 destructive 级确认分级（批量下载可能上千页——confirm 文案已含数量 l10n.downloadAuthorConfirmBody），`cancel` 在枚举中可取消、confirming 态 cancel 只是 pop（枚举已发出的请求已被 cancelToken 取消，正确）。
- W3 只需保持其行为，宽屏 AlertDialog 由 showAppDialog 天然限宽，无需动作。

## 5. 其他共享件

- `PersonAvatar`（lib/app/person_avatar.dart L8-59）：圆形+可选 ring，preview 复用点。
- `AuthorSummary`（lib/app/widgets/author_summary.dart）：avatar+name+account 行，compact/regular 两变体；profile 未直接用它（header 是自定义布局），user feed 的 `_UserPreviewTile` 是平行实现（avatar+ListTile+FollowSwitchButton compact，profile_user_feed.dart L114-137）——W3 不强制收敛它（属 W6 对象基座议题），但统计/链接行新组件要避免再造第三份。
- `outboundUrlOpenerProvider`（core/platform/android_intent_channel.dart L98-114 + caption_rich_text.dart L45-55）：打开外部 http(s) 的唯一边界，社交链接「主操作打开」应消费它而非直接 url_launcher；内部 pixiv 链接解析先例在 caption_rich_text `_resolvePixivRoute`（L100-125）。
