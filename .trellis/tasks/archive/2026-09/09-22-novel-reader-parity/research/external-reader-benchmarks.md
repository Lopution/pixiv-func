# 外部对标：小说阅读器功能面

来源：Pixiv-Shaft（CeuiLiSA/Pixiv-Shaft）release notes 与 PR/commit 记录、legado（阅读）官方使用说明、PixEz-Flutter issues、排版可读性文献（WCAG/Baymard/Butterick/Wikipedia）。检索日期 2026-09-22。

## 1. Shaft `NovelReaderV3Fragment`（Android/Kotlin）

v4.5.9/v4.6.2 与 PR #948、commit 9f6621a 公开记录：

- **舞台**：沉浸正文 + 顶/底 chrome；`loadState`(Idle→Loading→Loaded) 与 `pagination` 双状态驱动 loading——分页未完成不撤 loading（PR #948 复盘：建议收敛为单一 `updateLoadingVisibility()` 入口 + 显式 `isPaginating` 状态）。
- **底栏**：进度条（slider）+「系列」按钮——有系列时一键换单篇；章节/系列/搜索结果**三个 bottom sheet** 统一。
- **正文搜索**：高亮 + 命中居中 + 结果列表 sheet；返回手势优先退搜索。
- **翻页**：横向/纵向模式可切（`applyFlipMode`），音量键翻页。
- **设置**：进入即应用主题色。
- **入口策略**：`novelListDirectToReader` 开关（默认关）让列表点击跳过详情直进正文；正文页回详情走「更多」菜单里的「作品详情」——即顶部动作收敛进 overflow，信息不另占图标位。

对本包的映射：Shaft 底栏 = 进度条 + 系列 + 目录/搜索 sheet 簇；我们 §4.5 底栏 = 目录 + 可操作进度 + 排版设置。搜索/朗读/音量键属 Shaft 有而我们 roadmap 未列的能力，**不要顺手扩进来**。

## 2. legado（阅读）阅读界面结构

官方使用说明（geekdaxue.co/read/legado@wiki）：

- 顶栏：书名、换源、刷新、离线缓存、更多。
- 底栏：亮度滑块、全文搜索、自动翻页、替换净化、深色模式、**章节跳转+本章进度滑块**、**目录**、朗读、界面、设置。
- 排版设置：字号/字距/行距/段距、字体、缩进、简繁、边距、翻页动画（覆盖/仿真/滑动/滚动）、文字颜色与背景。
- 系统设置项：屏幕方向、超时、音量键翻页、点击区域自定义。

本仓库注释已自承 "legado-style footer tip"（novel_page.dart:240）——页脚 `标题·页·%` 常驻即 legado 模式。可借鉴点：**章节跳转滑块 + 目录入口**是 e-reader 底栏的事实标准位；翻页动画/音量键属可选项（非 §4.5 范围）。

## 3. PixEz-Flutter（Notsfsssf/pixez-flutter）

- 正文同样来自 `app-api.pixiv.net/webview/v2/novel`（issue #688），与我们一致；pixiv 官方 webview 阅读器自带 `font/font_size/line_height/color/background_color/mode=horizontal/theme/margin_*` 参数——印证「字号/行距/配色/翻页模式」是官方认可的最小设置面。
- PixEz 阅读器为纵向滚动，issue #732 显示超长小说（10万字+）接近罢工——我们已是分页引擎+预算上限，横向 PageView 方案在这方面优于其滚动方案；无需改方向。
- 无本地 TXT 书架概念（Shaft/legado 才有本地书）——本地阅读器对标 legado 更贴切。

## 4. 行长限宽（宽屏正文）

排版可读性共识：

- **WCAG 1.4.8**：每行 ≤80 字符（拉丁）；**CJK ≤40 字符**。
- Baymard：正文 50–75 cpl；Butterick：45–90 cpl（2–3 个字母表）；印刷书 ≈ 20–40×字号。
- 含义：拉丁字符均宽 ~0.5em、CJK 字符宽 1em → `maxTextWidth = fontSize × 40` 同时满足两族（40em ≈ 80 拉丁字符 = 40 个 CJK 字）。
- 本库默认字号 17 → 上限 ~680dp；30 → 1200dp。**字号相对上限**随字号滑块自缩放，比固定 dp 常量更贴合；可再叠一个绝对兜底（如不超 viewport）即可。
- 工程接法：`NovelLayoutStyle.horizontalPadding` 固定 24（layout L16）；把限宽做进排版 viewport（`_maxWidth` L878-882）或在 reader 层把布局宽度与手势宽度解耦 —— 推荐后者：手势区保持全宽，文字列居中限宽（详见 implementation-draft §B）。

## 5. 进度跳转与「可取消」的通行做法

- legado：章节进度 slider + 上/下章按钮；Shaft：底部进度条 + 系列 sheet。
- 通行语义：拖动 slider = 预览目标（页码/%），松手/确认 = 提交跳转；取消 = 不离开原位。「预览不落进度」是本包「不把打开信息记作已阅读」条款的自然实现——只有真实落页才写持久化锚点。
- Flutter 侧注意：`PageController.animateToPage`（现 handle.goToPage 用它）逐页动画在跨百页跳转时会跑完全部动画帧；远距离跳转宜用 `jumpToPage` 或参数化 `animated` —— 内核 handle 需小幅扩展而非另起跳转通道。
