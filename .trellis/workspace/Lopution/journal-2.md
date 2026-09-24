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


## Session 65: W9 引导登录与内容布局：四阶段收敛引导/登录/WebView/Spotlight 表现层
<!-- trellis-session: v=2 fp=0da005ece52a2816 -->

**Date**: 2026-09-23
**Task**: W9 引导登录与内容布局：四阶段收敛引导/登录/WebView/Spotlight 表现层
**Branch**: `task/09-22-onboarding-auth-content-layout-s4`

### Summary

四阶段串行落地：ScrollableFormShell（可滚动+限宽+钉底 CTA）承载 welcome/language/theme/协议/启动错误/登录页；登录页主动作恒定在场、帮助次级化、剪贴板导入 busy 态；Spotlight 正文 700 限宽+段落级可选择+分享/打开原文；WebView 错误层级两端共用 LoginWebViewErrorCard（recoverable→重载+关闭，fatal 登录→原地重启 PKCE，fatal 注册→仅重载），错误文案统一为 '<type> <host>' 不带 query，mobile signup 补 onProgress 对齐 desktop。W1 动作语义全部保持原契约。

### Main Changes

- 新增 app/widgets/scrollable_form_shell.dart + layout/content_widths.dart 角色限宽常量；引导三页/协议页/登录页/启动错误页迁入 shell；Spotlight 限宽+段落 SelectableText+share/open_in_new；login_webview_error_card.dart 两端共用 + describeWebViewFailure 统一 '<type> <host>' 文案；mobile signup NavigationDelegate 补 onProgress

### Git Commits

| Hash | Message |
|------|---------|
| `8ea60a0` | feat(app): 新增可滚动限宽钉底 CTA 的 ScrollableFormShell |
| `d9c81e6` | feat(onboarding): 引导页迁移到共用可滚动限宽 shell |
| `972107c` | feat(onboarding): 用户协议页限宽且正文可选择 |
| `478b927` | feat(app): 启动错误页主操作升级并限宽 |
| `3967359` | feat(login): 登录页迁入可滚动限宽 shell |
| `a9e4749` | feat(spotlight): 文章正文限宽并可按段落选择 |
| `528d9f0` | feat(spotlight): 文章页支持分享与浏览器打开原文 |
| `c1ba35c` | refactor(login): 两端授权 WebView 共用错误层级卡 |
| `6a401d5` | fix(login): WebView 错误文案与注册进度条两端对齐 |

### Testing

- [OK] flutter analyze 0 issue；全量 flutter test 1432 通过；focused login_navigation/login_webview_desktop 全绿（desktop fatal 用 detached+resumed 组合规避 framesEnabled 停摆）；git diff --check 净；task.py validate 过

### Status

[OK] **Completed**

### Next Steps

- PR body 已声明未验证项：WebView2 缺失态真机、桌面端实机错误卡行为、两端真机端到端登录


## Session 66: W8 设置表单语义：FormState 四态分类落地与摘要/限宽/dirty 守卫/诊断生命周期
<!-- trellis-session: v=2 fp=207049c95d8c86e3 -->

**Date**: 2026-09-23
**Task**: W8 设置表单语义：FormState 四态分类落地与摘要/限宽/dirty 守卫/诊断生命周期
**Branch**: `task/09-22-settings-form-semantics-s5`

### Summary

五阶段串行合入（PR #63/#69/#71/#72/#80）：s1 SettingsTile 摘要槽+settingsNarrowBody 限宽 600+persistSettings 收敛+根页全量摘要与历史直达；s2 dirty 守卫 helper+浏览页重排+pixivHistory 归并+屏蔽动词改解除+主题语言 Semantics selected；s3 账号切换 busy+凭据清除确认+翻译配置态摘要+命名模板 invalid 禁保存+SAF 人类可读名；s4 网络页摘要优先级+高级页统一草稿与重置确认+探测先摘要后细节+帧探针生命周期分离与 _frames cap；s5 仓库链接打开/复制+更新失败五态分文案+备份两段策略流（等权选择→同权确认）+术语全量核对+宽度矩阵验收。

### Main Changes

- s1：SettingsTile.subtitle、settingsNarrowBody(ContentWidths.settings=600)、persistSettings 收敛删 _persistNetwork/_write、根页全量摘要+查看历史直达
- s2：guardDraft/confirmDiscardDraft dirty 守卫、浏览页偏好优先重排、pixivHistory 归并历史页、屏蔽解除动词+失败输入恢复+pending 可视、主题语言 Semantics(selected)
- s3：账号切换 busy、凭据清除确认+dirty 守卫、翻译页已配置摘要（core hasBaidu/hasLlm 存在性读法）、命名模板 invalid 禁保存、safTreeDisplayName 纯 Dart 解码+raw URI 下沉
- s4：网络主页摘要优先级+第三方可达性说明、高级页页级 draft 单保存+恢复默认值确认、probe 总览区+明细折叠、帧探针 dispose 不 stop+录制状态条+_frames cap 10000 FIFO
- s5：仓库 tile launchUrl 打开+trailing 复制、更新检查 offline/rateLimited/invalid/busy/failed 五态分文案+apply canceled/failed 分开、备份导入两段 showAppDialog（等权策略选择继续禁用→同权确认标题复述后果）、术语核对+宽度矩阵（320/390/600/840/1200+横屏+1.3x）无溢出断言、弹窗 scrollable 修横屏溢出

### Git Commits

| Hash | Message |
|------|---------|
| `699e204` | chore(task): 启动 09-22-settings-form-semantics |
| `b8c0801` | refactor(settings): SettingsTile 增摘要槽并收敛 persistSettings 副本 |
| `f1531a6` | feat(settings): 根页入口显示当前值摘要并直达历史 |
| `9387631` | feat(settings): 设置页内容栏限宽 600 |
| `96bd822` | chore(task): 记录阶段 0 rebaseline 基线 |
| `e7a4192` | chore(task): 记录 s1 PR 链接 |
| `6c93e33` | chore(task): 切换阶段分支 s2 |
| `2680197` | feat(settings): draft 表单 dirty 离开确认 helper |
| `6811e9b` | feat(settings): 浏览页偏好优先重排与自定义图源草稿守卫 |
| `3de4cd8` | feat(settings): 历史记录开关归并历史设置页 |
| `d3a465d` | feat(settings): 主题语言选中态补语义通道 |
| `8e69b3b` | fix(settings): 屏蔽解除动词与失败输入恢复对齐术语 |
| `8f2b6d4` | chore(task): 记录 s2 PR 链接 |
| `031ce9e` | chore(task): 切换阶段分支 s3 |
| `e3680c4` | feat(settings): 账号切换补 busy 态 |
| `43f4333` | feat(settings): 凭据清除加确认并接 dirty 守卫 |
| `928ed5b` | feat(settings): 翻译页凭据入口显示配置状态 |
| `fe5ccd6` | fix(settings): 命名模板无效时禁用保存并接草稿守卫 |
| `eec6906` | feat(settings): 保存位置主视图人类可读名 |
| `afb7e0b` | chore(task): 记录 s3 PR 链接 |
| `91e3d09` | chore(task): 切换阶段分支 s4 |
| `3e89ff6` | feat(settings): 网络页摘要优先级与可达性说明 |
| `77532a1` | feat(settings): 网络高级页统一草稿与重置确认 |
| `33e037c` | feat(settings): 网络探测先摘要后细节 |
| `f3d4143` | feat(settings): 帧探针录制生命周期与控制页分离 |
| `232ab03` | chore(task): 记录 s4 PR 链接 |
| `593b116` | chore(task): 切换阶段分支 s5 |
| `996abff` | feat(settings): 仓库链接可打开与复制 |
| `6dbbaeb` | fix(settings): 更新失败按可行动原因区分文案 |
| `d660fbb` | fix(settings): 备份导入先选策略再同权确认 |
| `dfe41fb` | chore(settings): 术语表全量核对 |
| `7615901` | fix(settings): 备份导入弹窗横屏可滚动并补宽度矩阵验收 |
| `cb8b020` | chore(task): 记录 s5 PR 链接 |

### Testing

- [OK] flutter analyze --no-pub 0 issues；聚焦 122 全绿+rebase 后 83 全绿；全量 flutter test 1505 通过；git diff --check 净；task.py validate 过

### Status

[OK] **Completed**

### Next Steps

- PR body 已标未验证：真机 launchUrl/剪贴板 OEM 差异、TalkBack/Narrator 弹窗朗读、桌面实机宽度矩阵、俄语长标题观感


## Session 67: W6 实体管理一致性:下载任务页对齐管理页规范(downloads stage)
<!-- trellis-session: v=2 fp=935be2df6b136600 -->

**Date**: 2026-09-24
**Task**: W6 实体管理一致性:下载任务页对齐管理页规范(downloads stage)
**Branch**: `task/09-22-entity-management-consistency-downloads`

### Summary

DownloadManager 新增终态清理 API;任务页组聚合、选择模式批量动作、九态动作映射、Snackbar 直达任务页、840 限宽与触觉分级

### Main Changes

- DownloadManager: dismiss(taskId)/clearTerminal() 终态清理 API(任务项+组槽+恢复记录+changes 通知)
- 下载任务页:组聚合父子层级渲染;选择模式(长摁/动作进入)支持全选、批量取消与批量移除(showAppDialog 确认)
- 九态动作映射对齐术语契约;新增 downloadViewResult/downloadRemoveRecord/downloadProcessing 等四语言 l10n 键
- 下载提交 SnackBar 增加『查看』直达 /settings/tasks;页面限宽 600→840(management);ListView restorationId 补齐
- 触觉分级:进入选择模式与确认弹窗 confirm()(heavyImpact),选择切换 select()(selectionClick)

### Git Commits

| Hash | Message |
|------|---------|
| `bddd1d2` | feat(download): 新增终态任务移除与清空 API |
| `51fd476` | feat(settings): 下载任务按组聚合父子层级 |
| `ddde95d` | feat(settings): 下载九态动作映射对齐术语契约 |
| `f63dbed` | feat(download): 提交反馈可直达任务页 |
| `4072841` | feat(settings): 下载任务选择模式与限宽接入 |

### Testing

- [OK] flutter analyze --no-pub 干净;dart format --set-exit-if-changed 与 git diff --check 通过
- [OK] download_tasks_page_test 7 项全过(批量取消/移除、840 限宽、平台通道触觉断言);local_novel/watchlist 回归全过
- [OK] 全量 flutter test 1546 项全过(EXIT 0)

### Status

[OK] **Completed**

### Next Steps

- 真机验证:触觉反馈、TalkBack、宽度矩阵、1.3x 字体缩放


## Session 68: W10 动效集成与最终验收：reduced-motion 三注入、跨包契约闸口与验收台账关闭
<!-- trellis-session: v=2 fp=c94e221ee9e5b15b -->

**Date**: 2026-09-24
**Task**: W10 动效集成与最终验收：reduced-motion 三注入、跨包契约闸口与验收台账关闭
**Branch**: `task/09-22-motion-integration-acceptance`

### Summary

路线图收尾：W1–W9 合入核对后按证据做 8 项窄修复（平台 reduceMotion 并入 MotionTokens.enabled、DragToDismiss/详情分页器/小说阅读器/SnackBar 过闸、motion 别名删除、收藏 tagInput 残文入脏判定、小说偏好读取失败落错误态）；C1–C10 逐契约回归全部落地；产出 acceptance-ledger.md 并关闭父追踪矩阵

### Main Changes

- 阶段1 rebaseline：9/9 leaf completed+archived（origin/main@5dbef81），在途 PR #79/#82/#85 记 in-flight
- 阶段2 窄修复 8 项：MotionTokens.enabled 并入 accessibilityFeatures.reduceMotion；DragToDismiss 取消/结束返回过闸；详情分页器时长走 tokens；小说 chrome/翻页过闸；FuncSemanticTokens.motion* 别名删除；SnackBar 动画样式过闸（design 二审撤销豁免）；收藏 _isDirty 并入 tagInput 残文（W3 已知项）；小说 _loadPrefs 失败落 FeedError+retry（W9 已知项）
- 阶段3 回归 C1–C10：反馈通道/动效常量静态闸口、SnackBar margin+样式、re-tap 回顶不刷新、恢复等级逐页核对、hero 单一空间链、分类 tap/drag 双注入一致、键盘不位移主控件、§5.7 术语表冻结测试、FormState/BackAndCancel 三处抽查（补 novel reader PopScope 第三腿）、三优先页语义树断言
- 阶段4：acceptance-ledger.md 全量台账（implemented/unit-widget-tested/未验证/in-flight 分级）；父 astra-review-traceability 44 行状态列刷新+非 Astra 契约项标注+§8 关闭注记；父 implement.md W10 汇总勾选

### Git Commits

| Hash | Message |
|------|---------|
| `3836f04` | chore(task): 启动 09-22-motion-integration-acceptance |
| `7eb3efe` | docs(trellis): W10 验收基线与 W1–W9 合入核对 |
| `a0fc077` | fix(motion): MotionTokens.enabled 覆盖平台 reduceMotion |
| `29dc188` | fix(motion): DragToDismiss 返回动画接入 reduced-motion 闸 |
| `95eceb3` | fix(illust): 详情分页器时长走 MotionTokens 闸口 |
| `7db3596` | fix(novel): 阅读器 chrome 与翻页动画接入 reduced-motion 闸 |
| `46d1d0c` | chore(theme): 删除未使用的 motion 别名字段 |
| `d237870` | fix(app): SnackBar 动画样式接入 reduced-motion 闸（design 二审撤销豁免） |
| `734f943` | fix(bookmark): tagInput 残文并入草稿脏判定 |
| `6cd01b4` | fix(novel): 阅读器偏好读取失败落错误态 |
| `785f70d` | test(architecture): 反馈通道与动效常量单一来源静态闸口 |
| `01f34e7` | test(app): SnackBar 分支根 margin 与动画样式回归 |
| `910198b` | test(nav): re-tap 当前标签回顶且不触发刷新 |
| `c06af21` | test(nav): 查询上下文恢复等级声明逐页核对 |
| `4af922b` | test(motion): feed→detail→viewer 单一空间链回归 |
| `4352fd6` | test(nav): 同级分类点击/拖动双注入一致性回归 |
| `89cb800` | test(form): 键盘展开不位移主控件 |
| `3f73032` | test(l10n): §5.7 动作术语表对照回归 |
| `d2c464c` | test(regression): FormState/BackAndCancel 三处抽查 |
| `de886f1` | test(a11y): 三优先页语义树代表路径断言 |
| `37fa065` | docs(trellis): W10 最终验收矩阵与追踪矩阵关闭 |

### Testing

- [OK] dart format 0 changed；flutter analyze --no-pub 0 issues；git diff --check clean；task.py validate ✓
- [OK] 全量 flutter test：149 文件 +1529 全绿；3 个 loopback 文件（oauth_service/download_manager/tls_sni）环境缺陷噪声——自标 environment-flaky/loopback workaround，全量中 oauth 记录 loopback still failing after retries，与本分支零重叠
- [OK] 冲突标记精确扫描 lib/test/.trellis 零命中；rebase 至 origin/main@d798027 零冲突

### Status

[OK] **Completed**

### Next Steps

- PR body 同步台账 §9 未验证面清单（设备/桌面 runtime/AT/性能/体感/进程死亡/长翻译/视觉等价人工复核）；CI 绿后 merge
