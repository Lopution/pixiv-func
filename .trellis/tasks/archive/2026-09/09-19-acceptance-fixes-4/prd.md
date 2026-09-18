# acceptance-fixes-4: TickerMode动效冻结族/资料编辑AppAPI/沉浸阅读器/翻页校验/分享

## Goal

第四轮验收反馈，五个批次一次做完：修复转场期动效冻结族（P1/P7/P8）、个人页返回按钮几何与转场消失（P3/P4）、资料编辑从 web cookie 通道换 App API、推荐小说翻页 filter 严格等值误杀、沉浸式阅读器壳层、系统分享统一接入。

## Requirements

### A 批 — TickerMode 冻结族（P1/P7/P8）

- `PressScale`/`StaggeredEntrance` 感知 `TickerMode.of(context)`：ticker 禁用（路由转场、页面被覆盖）时直接渲染终态（scale=1 / 不透明零位移），解冻后不再补播半路动画
- 入场动画 once 语义：feed 级会话内"已播"集合，划出去再回来不重放上浮动画（`addAutomaticKeepAlives:false` 下重建不重播）
- 个人页返回按钮：展开/折叠收敛为同一几何同一组件（同一颗 Positioned 常驻按钮），圆心与底色圆一致
- 返回动画中按钮不消失：`canPop` 首次求值后缓存，pop 进行时不因路由摘除翻假

### B 批 — 资料编辑换通道

- 资料编辑/预置从 web `/ajax/my_profile`（依赖 www.pixiv.net PHPSESSID）换成 App API `v1/user/profile/edit` + `v1/user/profile/presets`（OAuth token，与全站同通道）
- `_StatusBody` loading 态不再显示"加载失败"文案；真实失败才出图标+重试
- 保留已有表单字段与提交契约，字段对齐 Shaft 的 presets/edit 端点

### C 批 — 沉浸式阅读器

- `NovelPage` 阅读态全屏 Stack：`[阅读舞台, 顶栏chrome, 底栏chrome]`；默认沉浸隐藏 chrome
- 中央点击 toggle chrome；上下栏联动滑入滑出动画，隐藏播完才 IgnorePointer；chrome 显示时返回键先关 chrome
- 顶栏：返回/标题/收藏/更多（分享、作品信息）；底栏：上一章·页码进度·下一章 + 目录 + 设置
- 元数据块撤出阅读页（标题去重）；简介改 `CaptionRichText`/`parseIllustCaption` 渲染 HTML（`<br>` 不再裸露）
- 阅读设置弹层：字号/行距/段距/页边距/主题（纸张白/护眼黄/暗夜跟随）→ 映射 `NovelLayoutStyle` 持久化
- 页内页脚 tip：章节名/页码/进度小字（不随 chrome 隐藏）；charIndex 进度持久化下次恢复

### D 批 — 推荐小说翻页

- `_validateCursor` 只校验 feed 身份参数（`user_id`/`series_id`/`mode`/`word`），`filter` 等客户端身份参数放行
- 推荐首页请求对齐参考实现：`include_privacy_policy=true` + `include_ranking_novels=true`，endpoint 注册表同步

### E 批 — 分享统一

- 引 `share_plus`，统一 `SharePayload`：`{title} | {author} #Pixiv {url}`；URL 按类型 `artworks/{id}`/`novel/show.php?id={id}`/`users/{id}`
- 详情页/阅读器 chrome 更多菜单放系统分享；个人页分享弹窗改「系统分享+复制链接」；卡片快捷操作保留复制链接
- 分享 API 失败 fallback 剪贴板+snackbar；iPad/桌面传 `sharePositionOrigin`

## Acceptance Criteria

- [ ] 按压卡片进入详情返回后，卡片无"残留缩小→突然回弹"
- [ ] 快速往返滚动 feed 不重播入场动画；排行小说页转场中即显示完整列表
- [ ] 个人页返回按钮展开/折叠位置一致，返回转场中随页右滑消失
- [ ] 资料编辑可用（能加载+能提交），loading 态文案正确
- [ ] 阅读器默认全屏沉浸，点中央唤出/收起 chrome；标题不重复；`<br>` 正常换行；设置可调字号/主题且持久化；重进恢复进度
- [ ] 推荐小说能翻页加载更多
- [ ] 插画详情/小说/个人页均可触发系统 Sharesheet，文案含标题+作者+链接
- [ ] `flutter analyze` 零警告、`flutter test` 全绿、Kotlin 测试过

## Out of Scope

- 仿真/cover 翻页动画（Flutter 无现成轮子，单列后续）
- 阅读器搜索/标注/TTS/自动翻页/音量键翻页
- 分享首图（FileProvider 等价）、分享文案用户模板
- 九宫格点击区自定义（固定 30/40/30 即可）

## Notes

- 参考仓：`/root/pixiv-audit/Pixiv-Shaft`（v1/user/profile/edit、shareNovel 格式）、`/root/ereader-audit/legado`（ReadView 九宫格/ReadMenu/PageDelegate/ReadTipConfig）、`skana_pix/lib/utils/text_composition`（Flutter 分页引擎移植参照）
- 分页引擎 `NovelLayoutEngine` 不动，改造全在壳层
