# Journal - Lopution (Part 2)

> Continuation from `journal-1.md` (archived at ~2000 lines)
> Started: 2026-09-22
> Started: 2026-09-23

---



## Session 59: W1 交互结果正确性实现（9 项确定性修复）
<!-- trellis-session: v=2 fp=b7ce73d55491599b -->

**Date**: 2026-09-22
**Task**: W1 交互结果正确性实现（9 项确定性修复）
**Branch**: `task/09-22-interaction-outcome-correctness`

### Summary

9 项确定性交互修复全部落地并合并（PR #56，merge 2ebf2b8）：小说显式返回离页+标签可点搜索、本地小说读位恢复、资料编辑脏确认、热门标签不丢内容、镜像源改名应用并测试、翻译凭据清除同步、反向搜图原地取消、登录 WebView 动作与后果一致。trellis-check 独立复核 9/9 pass；全量 ~1253 测试通过；analyze/format/diff-check/validate 全绿。真机运行时矩阵（触觉/Predictive Back/WebView2/屏幕阅读器）未验证，移交 W10。

### Git Commits

| Hash | Message |
|------|---------|
| `0d376d7` | chore(task): 启动 09-22-interaction-outcome-correctness |
| `a97a5c8` | fix(localnovel): 打开本地小说恢复上次阅读位置 |
| `cd551ee` | fix(profile): 资料编辑无改动直接返回，修正返回 tooltip |
| `8f7572a` | fix(search): 热门标签不再为凑整行丢内容 |
| `f1f3faf` | fix(settings): 翻译凭据清除同步输入框并区分成败态 |
| `d7223f2` | fix(search): 反向搜图区分取消搜索与离开页面 |
| `ffbf138` | fix(novel): 显式返回直接离页，系统返回仍先关 chrome |
| `fcca618` | fix(novel): 信息页标签可点击搜索 |
| `a6c26a4` | fix(settings): 镜像源测试改名应用并测试 |
| `b530534` | fix(login): 授权 WebView 动作名称与后果一致 |
## Session 61: W4 作品浏览/查看器/系列流程
<!-- trellis-session: v=2 fp=252e775d07abf66d -->

**Date**: 2026-09-23
**Task**: W4 作品浏览/查看器/系列流程
**Branch**: `task/09-22-artwork-viewer-series-flow`

### Summary

20 个 checkbox 全部落地：AppHaptics 角色 API + 19 处消费点；下载全部常显 + 显式多页选择模式（三形态断言）；查看器会话级 chrome、双击消歧、底栏工具栏、空态页码、键鼠等价（pointerSignalResolver 先到先得修复 Shift+滚轮翻页）、PopScope 仅由缩放态把关；窄屏 compact header + info 跳转；series firstContentId 解析 + 会话内存最近打开 + 开始阅读/返回第 n 话分源动作

### Main Changes

- image_viewer_page.dart 重写手势/键盘/滚轮/PopScope 模型；illust_detail_page.dart 下载模式骨架 + compact header；app_haptics.dart 角色 API；series_* firstContentId + recent_open_store

### Git Commits

(No commits - planning session)

### Testing

- [OK] 全量 1289 通过；analyze 干净；三形态下载模式断言；PopScope×route-replace 用例

### Status

[OK] **Completed**


## Session 60: W2 discovery-query-context：re-tap 通道 + 查询上下文路由化 + 反向搜图任务头
<!-- trellis-session: v=2 fp=9f816c8945538c91 -->

**Date**: 2026-09-23
**Task**: W2 discovery-query-context：re-tap 通道 + 查询上下文路由化 + 反向搜图任务头
**Branch**: `task/09-22-discovery-query-context`

### Summary

发现页查询上下文连续（W2）12 项全部落地：ReTapChannel 事件通道（ChangeNotifier，同槽连点不丢信号，移交 W3 复用）、排行/小说排行/推荐/新作分类与查询参数全部路由化（context.replace 回写 + didUpdateWidget 抑制自回显）、推荐与新作 TabBar 去缩字改滚动、新作类型选择器常驻且 re-tap 固定回顶、搜索筛选 13 字段 URL round-trip、热门标签自适应网格不丢项、结果页查询头可改+筛选摘要 chips+空态修改入口、建议项区分填入/立即搜索、反向搜图常驻任务头。trellis-check 全绿（修掉了 lookup.dart 幻影格式 diff + 补写了三个缺失的路由 round-trip 测试 + 更新 home_bar golden）；全量 1270 测试通过；analyze/format/diff-check/validate 全绿。桌面滚轮手感/进程死亡恢复/弹栈-回顶时序/大字号/屏幕阅读器未验证，移交 W10。

### Git Commits

| Hash | Message |
|------|---------|
| `8bfda22` | chore(task): 启动 09-22-discovery-query-context |
| `c867f63` | feat(nav): 新增 branch re-tap 回顶广播通道 |
| `05c30b0` | feat(ranking): 重复点击当前榜单回到顶部 |
| `d0d8bb3` | feat(ranking): 小说排行接入滑动切换与 mode 路由参数 |
| `22d7857` | feat(recommended): 内容类型接入路由参数 |
| `4c3d4fd` | feat(recommended): 标签不缩字、re-tap 回顶、刷新错误就近提示 |
| `a9ac774` | feat(new): 范围与类型接入路由参数 |
| `4351125` | feat(new): 类型选择器常驻，re-tap 固定回顶 |
| `17e4021` | fix(search): 路由序列化补齐全部筛选字段 |
| `6e9ff8e` | feat(search): 热门标签自适应网格并接入 re-tap |
| `a4237f2` | feat(search): 结果页查询头可编辑并补筛选摘要 |
| `618c1b6` | feat(search): 建议项区分填入与立即搜索 |
| `59462bd` | feat(search): 反向搜图常驻图片与引擎任务头 |

### Testing

- [OK] flutter analyze 0 issues; flutter test 1270/1270; 聚焦套件 root_swipe_switcher/search_catalog/new_content_feed/reverse_image/navigation_restoration/recommended_home 全绿

### Status

[OK] **Completed**

### Next Steps

- W3/W4 消费 ReTapChannel 契约；W4 worktree 继续实现
### Next Steps

- PR + CI 绿后 merge；W5 novel reader parity 开工


## Session 62: W3 作者页与收藏范围连续：头部等价路径/统计导航/分享直达/收藏范围传递/编辑弹层收敛
<!-- trellis-session: v=2 fp=cfe475e25863fc5b -->

**Date**: 2026-09-23
**Task**: W3 作者页与收藏范围连续：头部等价路径/统计导航/分享直达/收藏范围传递/编辑弹层收敛
**Branch**: `task/09-22-profile-bookmark-continuity`

### Summary

作者页头部展开/收起两态同源动作（收起 chrome 完全收起才挂载、溢出菜单补关注等价路径）、作品类型常驻+re-tap 回顶、统计可导航、社交链接打开/复制、分享直达；收藏范围经路由 query 入标签页、tag feed 本地过滤+范围标识、标签列表尾部重试；资料编辑底栏固定保存+预览形态+ContentWidths.form 限宽；收藏 sheet 控件全局一致+draft dirty 三路径确认+失败保留草稿

### Git Commits

| Hash | Message |
|------|---------|
| `ec79394` | fix(profile): 头部展开/收起动作同源，收起 chrome 完全收起才挂载 |
| `faa3357` | feat(profile): 收起态溢出菜单补齐关注等价路径 |
| `caeca95` | feat(profile): 作品类型常驻可见，重复点击固定为回到顶部 |
| `7d25df7` | feat(profile): 统计项可导航，头部与 about 共用映射 |
| `8694735` | feat(profile): 社交链接主操作打开、次操作复制 |
| `20d4124` | feat(profile): 作者分享与作品一致直达，复制链接独立 |
| `d42d766` | feat(bookmark): 收藏范围经路由 query 传入标签页 |
| `4b6e733` | fix(profile): 标签 feed 显示范围标识，heroScope 加 bookmarkTag |
| `80ce7de` | feat(bookmark): 标签 feed 本地过滤已加载内容 |
| `8f44c24` | fix(bookmark): 标签列表加载失败有尾部重试 |
| `6b5cf94` | feat(profile): 资料编辑保存入口固定在底栏 |
| `5cbf567` | feat(profile): 资料编辑预览形态与宽屏限宽 |
| `af8f881` | refactor(bookmark): 收藏编辑弹层控件与全局一致 |
| `32b6211` | feat(bookmark): 收藏编辑按 draft 语义，失败保留草稿 |

### Testing

- [OK] flutter analyze 0 issues；全量 flutter test 1322 过（2 项 sqlite lock 并发噪声单跑通过）；bookmark_switch_button 15 项、profile_edit 15 项、user_profile 21 项全绿

### Status

[OK] **Completed**

### Next Steps

- PR 合入后真机复核 design §五未验证清单（re-tap 手势竞争、sheet 键盘/限宽、reduced-motion、1.3x 长翻译、TalkBack/Narrator、iPad popover 锚点、长表单底栏）


## Session 63: W7 评论输入状态机收敛：四态互斥/回复页滚动并入/响应式网格/语义/发送进度/触觉接入
<!-- trellis-session: v=2 fp=d6cee1160f0586e1 -->

**Date**: 2026-09-23
**Task**: W7 评论输入状态机收敛：四态互斥/回复页滚动并入/响应式网格/语义/发送进度/触觉接入
**Branch**: `task/09-22-comment-input-state`

### Summary

阶段 0 依赖门禁核实（W1/W4 已合并、AppHaptics 角色 API 定位）后，A-D 四组 9 个 checkbox 全部落地：composer 收敛为 none/keyboard/emoji/stamp 四态互斥 + PopScope 仅拦面板；双页手动 insets + 键盘高度采样缓存（280dp 回退）；回复页根评论并入列表滚动、非列表态滚动兜底防溢出；reply pill 主动聚焦 + 引用条 Semantics 容器；发送成功清回复目标（mid-flight 改目标防误清）+ 回复页权限错误分支；emoji/stamp 网格按宽度定列（48dp/96dp cell，3-10/2-5 clamp）；cell button+label 语义 + commentStampLabel/commentSending 四语落地；发送中按钮内 spinner；AppHaptics.success() 接入两页发送成功点（文本/stamp 同点同档，composer 无业务副作用）

### Main Changes

- comment_input.dart：CommentComposerInputState 四态枚举 + 焦点单向桥 + PopScope 面板拦截 + 手动 bottomExtent + LayoutBuilder 宽度驱动网格 + cell/面板/引用条 Semantics + 发送中 spinner
- comments_page.dart：_CommentFeedView header 槽（根评论+标题随列表滚动，非列表态 SingleChildScrollView 兜底）、GlobalKey<CommentComposerState> + _replyTo 主动聚焦、_send 成功守卫清目标 + AppHaptics.success()、回复页权限错误分支
- 四语 ARB 新增 commentStampLabel(int)/commentSending + gen-l10n + lookup 再生成；state-management.md 网格表述改响应式契约；comments_replies_test.dart 新增/重写 23 个用例
## Session 64: W5 novel-reader-parity：在线/本地共享阅读舞台
<!-- trellis-session: v=2 fp=8f116f5c6de20eee -->

**Date**: 2026-09-23
**Task**: W5 novel-reader-parity：在线/本地共享阅读舞台
**Branch**: `task/09-22-novel-reader-parity`

### Summary

NovelReaderStage 共享舞台落地：在线/本地同一 chrome/设置/进度/返回语义；锚点通知区分恢复回显与用户翻页（D1 进度门）；宽屏正文按字号×40 限宽居中；本地 TXT 接入舞台（readOffset null/0 语义保持）；可操作进度滑杆+目录 sheet（无动画远跳）；左右方向键翻页；设置范围对齐模型约束、持久化失败可见；排版预算超限渲染错误态

### Main Changes

- 内核：goToPage(animate:false)、锚点来源标记、章节表暴露、限宽居中、预算超限错误态
- 舞台：novel_reader_stage.dart 抽取 chrome/设置/进度/binding，在线页 spec 装配
- 本地：local_novel_reader_page 接入舞台+文件信息 sheet，offset 只由真实翻页写入
- 新增：进度/目录 sheet（Slider 预览+确认跳页）、键盘左右翻页

### Git Commits

| Hash | Message |
|------|---------|
| `8b69127` | feat(comments): 发送成功接入共享触觉反馈 |

### Testing

- [OK] [OK] test/comments_replies_test.dart 35/35 通过；全量 1327 通过（account_store 2 失败单文件复跑全绿，判并行负载噪声）；analyze/format --set-exit-if-changed/diff --check/task validate 全绿
| `01729a1` | chore(task): 启动 09-22-novel-reader-parity |
| `fd63b55` | chore(task): 记录 W5 阶段 0 rebaseline 门禁结果 |
| `962a447` | feat(novel): 阅读器跳转接口支持无动画跳页 |
| `ad86a6a` | feat(novel): 锚点回调区分恢复回显与用户翻页 |
| `4bad740` | fix(novel): 排版预算超限渲染错误态而非未捕获异常 |
| `ea642a8` | feat(novel): 阅读器命令面暴露章节列表 |
| `eac0165` | feat(novel): 宽屏正文按字号相对限宽居中 |
| `935003a` | refactor(novel): 抽取共享阅读舞台，在线页改为 spec 装配 |
| `54cb511` | fix(novel): 打开阅读器不再写入未阅读的进度 |
| `312331e` | fix(novel): 阅读设置范围对齐模型约束，持久化失败可见 |
| `80ebd35` | feat(localnovel): 本地阅读器接入共享舞台 |
| `c569346` | feat(localnovel): 本地小说文件信息弹层 |
| `1255da5` | feat(novel): 可操作进度跳转与目录弹层 |
| `c2583fb` | feat(novel): 方向键翻页 |
| `2f24059` | chore(task): 勾选 W5 implement 已完成项 |

### Testing

- [OK] flutter analyze 0 issues；聚焦 46 项全绿；全量 flutter test 1321 过；git diff --check 净；task.py validate 过

### Status

[OK] **Completed**

### Next Steps

- 真机未验证项随 PR 声明：键盘↔面板动画、TalkBack/Narrator 焦点、OEM IME、真机触觉、1.3x 字体、横屏、reduced-motion、俄语长文案、predictive-back
- PR body 标未验证矩阵：宽屏实机行长、TalkBack/Narrator、桌面焦点让渡、大 TXT CPU/内存、CRLF 渲染
