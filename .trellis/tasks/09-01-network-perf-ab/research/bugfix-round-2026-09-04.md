# 真机 Bugfix 决策记录 — 头像 / Hero / 多图占位（2026-09-04）

跟踪：09-01-network-perf-ab（真机验收阶段的修复轮）。

## 1. 头像加载动画 / 默认蓝圆

现象：个人页头像修复后，评论区作者头像、推荐用户头像、搜索结果头像、设置页账号头像仍显示
默认蓝色圆（CircleAvatar 默认背景），加载中无过渡。

根因：多个页面各自用 CircleAvatar（backgroundImage / child-ClipOval 两种写法），加载中或失败
时落在 Flutter 默认蓝色圆形背景上。

决策：新建公共组件 `lib/app/person_avatar.dart`（PersonAvatar），统一行为：
- 中性渐变占位（surfaceContainerHighest→High 渐变 + 低对比 person 图标），绝不使用蓝圆；
- 有图时经 PixivImage 淡入（350ms/200ms）；
- 可选白色环（个人页/评论区等深色背景场景）。
替换全部 6 处实现：profile_header_delegate._Avatar、user_page._ProfileAvatar、
search_result_page（用户卡）、recommended_home_page（UserRowCard）、illust_detail_page
（作者头像，key 保持 illust-author-avatar）、settings_page._AccountAvatar。

注意：reverse_image_search_page 的 leading CircleAvatar 是相似度数字圆（非头像），保留不动。

## 2. Hero 动画

现象（真机反馈两轮）：
- 早期：飞行中途被导航栏/头像区"截断"（自定义 clip 的 intersect 为空的后果）；
- 本轮：自定义全局矩形（Navigator 矩形）后影像盖在网格上方、遮住整页 UI（图：
  泳装大图悬浮在首页网格之上）。

根因：两条路径都在给 Hero 飞行加"自定义 viewport 裁剪"，任何 static 矩形都无法同时满足
push（源在滚动网格/头像顶部之下）与 pop（目标可能部分滚出视口）——用户看到的是被裁没或
盖住 UI 两个极端。

决策：**回到框架默认 shuttle**——`illustHeroFlightShuttleBuilder` 直接返回
`(heroContext.widget as Hero).child`，删除 `_GlobalRectClip` / `_heroViewportClip`。
影像尺寸由 Hero framework 在源→目标 Rect 间插值（标准行为，与其余页面一致）；
圆角自卡片内部的 ClipRRect 保留。测试同步改为断言“无 _GlobalRectClip”。

## 3. 多图多比例作品

现象：多图作品打开后先只显示第 1 张，detail 数据到达后其余页才划出位置（布局跳变）；
多比例页（横条/长条）在 detailReady 前按作品比例渲染，观感错误。

根因：渲染 childCount 被 detailReady 门控（`detailReady ? pageCount : 1`）。

决策：SliverList 恒为 `entity.pageCount`；detailReady 前的非首页渲染
`placeholderOnly` 占位（surfaceContainerHighest 底 + 页码数字 + 作品级比例，
`pageAspectRatioAt(0)` 在 detail 前回退到 work 宽高）。detail 到达后按真实比例/图替换，
页面结构稳定、无跳变。

## 测试与构建

- 全量 602 中本会话相关 52 项关键测试通过（hero_transition / pull_to_refresh /
  illust_detail / user_profile / settings / recommended_home）；
- 全量跑时 tls_sni_behaviour_test / oauth_service_test 偶发超时，独立重跑全过
  （并发真实 socket 环境 flaky，与本轮改动无关）；
- `flutter analyze` 0 issues。
- APK：`/root/PixivFunc-0904-fix5.apk`（sha256 b5b208cc…，debug 签名，
  `flutter build apk --release --flavor github -PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true`）。

## 待真机验证（fix5）

1. 评论区/推荐/搜索/设置页头像：无头像用户显示中性渐变占位（非蓝圆），加载淡入；
2. 个人页白环头像不回归；
3. Hero push/pop：影像按标准方式缩放飞行，不盖 UI、不裁没、不停驻；
4. 多图作品：打开即出现全部页（占位），detail 到达后替换为真实图与比例，无跳变。

---

## 4. Hero 飞行遮挡语义（修正轮，fix6）

现象（fix5 真机反馈）：飞行影像仍盖住 UI（大图悬浮在网格/底栏之上）；且此前"pop 时四角
突兀裁切圆角"的边界修复被上一轮删除。

用户明确的期望：**影像飞回时不能盖住 UI，应被 UI 优雅遮挡**。

根因（前两轮的两个极端）：intersect 裁切 → 影像被裁没；无裁剪默认 shuttle → 影像盖住
全部 UI。正确的第三条路径 = **稳定内容矩形**：
- 矩形来自 MediaQuery 静态计算（不依赖 route 转场中的 transform）：
  `top = statusBar + kToolbarHeight`，`bottom = screen - (kBottomNavHeight 45 + safeArea)`；
- shuttle 重挂 ClipRRect（12dp）——pop 方向（详情页 Hero child 无圆角）不再在目标处
  突兀裁切四角；
- 飞行末 15% shuttle 淡出（push 0.85→1.0、pop 0→0.15），与目标图交叉渐隐，
  消除"详情档大图 → 列表档缩略图"的硬切换闪烁（问题 2 的闪）。

测试：hero_transition_test 三用例更新为断言 `_GlobalRectClip.globalRect ==
Rect(0, 56, 400, 755)`（400x800 测试屏，安全区 0）+ 推拉同一边界。

## 5. 搜索推荐标签余 1 项

现象：热门标签 7 个（3 列网格 → 最后一行孤 1）。

修复：`itemCount = (tags.length ~/ 3) * 3`（>0 时），1-2 项小集合保持完整。

## 6. 底部导航触控反馈统一

现象：底栏 5 入口用 InkResponse（圆形波纹），与上方 TabBar（矩形水波）不一致。

修复：InkResponse → InkWell（borderRadius 8），与顶部 TabBar 同为矩形水波。

## fix6 验证要点

1. Hero push/pop：影像在 AppBar 与底栏之间的内容区飞行，被 UI 优雅遮挡；四角圆角保持；
   落点交叉渐隐（无缩略图闪烁）；
2. 搜索页热门标签 = 6 个（3x2 满行）；
3. 底部导航点击 = 与顶部 TabBar 相同的矩形水波。

---

## 7. 详情页底部"相关作品"（新功能，官方行为复现）

需求：作品详情页下方增加推荐作品，复现 Pixiv 官方客户端行为。

实现（复用 feed 同款分页机制）：
- 数据源：`GET /v1/illust/related?illust_id=&filter=for_ios`（app-api），
  `PixivRelatedIllustRepository`（新，`related_illust_repository.dart`）；
  `kNextPageEndpoints` 增加 `/v1/illust/related`（illust_id/filter/offset）。
- 分页：`RelatedIllustController extends PagedFeedController`（family by illustId），
  实体经 illustrate store generation commit（与推荐 feed 完全同构）。
- UI：`RelatedIllustsSlivers`（`related_illusts_section.dart`）——官方样式：
  双列网格卡（方图 + 标题 1 行 + 作者 1 行），区块标题「相关作品」，点击卡片
  push 新详情页（heroScope `related:<parentId>:<id>`，无 hero 飞行——官方无该动画）。
  滚动接近底部自动 loadMore（NotificationListener + 500px 阈值，仅 AsyncData 时
  触发——PagedFeedController.loadMore 内部 requireValue，AsyncLoading 会抛）。
- i18n：relatedWorks / relatedLoadFailed 4 语言。
- 测试：repo 3 项（URL 参数/解析/畸形 JSON）；widget 3 项（渲染标题+卡、
  点击开新详情页、空列表隐藏区块）。
- APK：`/root/PixivFunc-0904-related.apk`。

真机验证点：详情页底部出现相关作品区块（滚动加载），点击进入新详情页可逐层返回。

## 8. 相关作品卡加入 Hero 飞行（方案 A）

用户确认采用方案 A（体验连贯，优于官方 App 的普通 push）：

- `_RelatedIllustTile` 图片区包 `Hero(tag: illustHeroTag('related:<parentId>:<id>', id),
  flightShuttleBuilder: illustHeroFlightShuttleBuilder, child: ClipRRect(12, PixivImage(cover)))`
- 与详情页 push 时传入的 heroScope 完全一致 → push/pop 双向飞行；
- 目标卡滚出懒加载视口时框架自动降级为普通转场（不残留）；
- 复用既有 shuttle（圆角/稳定内容区 clip/末段淡出）零新机制；
- 唯一视觉差异：cover 方图在飞行中按 cover 重裁（官方缩放观感一致）。
- 测试：23 项（related + hero + detail）全过；APK `/root/PixivFunc-0904-related-hero.apk`。

## 9. fix7 — 闪白 / 相关区预取 / 底部指示器

1. **点开/退出闪白**：详情页主图 CachedNetworkImage 每次渲染都从透明淡入 350ms，
   半透明帧露出页面白底。PixivImage 加 `fade: false`（fadeIn/Out → Duration.zero），
   详情页 Hero 主图传 fade: false（URL 已被 flight/preload 预热，即时呈现）。
2. **相关区一直转圈**：section 位于懒加载 sliver，请求到滚动到底才开始 → 用户
   看到无限 spinner；且 AsyncError 时旧代码仍显示 spinner。修复：
   - 详情页 initState 即 `unawaited(ref.read(provider.future))` 并行预取；
   - section 增加 `async.hasError` 防御分支（显示错误 + 重试，不再死转圈）。
3. **底部五 UI 缺指示线**：当前 tab 图标下加 18x3 圆角短横线（primary 色），
   与顶部 TabBar indicator 视觉同步（透明占位保持布局稳定）。
- 全量 608 测试通过；APK `/root/PixivFunc-0904-fix7.apk`。
- 网络层：related 走 pixivHttpClientProvider（app-api policy 链路），与详情同 client，
  已接入网络层。

## 10. fix8 — 闪白根治 / related 错误可见 / 底部真 TabBar

1. **闪白仍复现（fix7 未根治）**：fade:false 只覆盖详情页主图；Hero 源/目标卡
   （feed 卡、相关卡）在 flight 结束后 image state 重建，350ms 交叉淡入半透明帧
   透出页面白底。→ 参与 Hero 的全部 PixivImage（feed 卡 fitWidth、相关卡 cover、
   详情主图）统一 `fade: false`。
2. **related 一直失败且无信息**：Provider 级解析异常（非 ApiError）旧分支永远
   spinner → 新增 `async.hasError` 防御分支；错误文本显示 `runtimeType + 摘要`
   （真机可诊断）；repo 逐条容错（坏元素跳过，不拖垮整页）。重试无效 = 每次同样
   解析/请求失败，错误详情下版可见。
3. **底部导航行为同步**：手写 InkWell+短横线 → **真 TabBar**（icon-only，
   TabController 驱动）：indicator 滑动动画、点击 ripple、选中颜色、懒构建
   （_visitedTabs 经 onTap 同步）全部与顶部 TabBar 同一机制。
- 43 项相关测试全过；APK `/root/PixivFunc-0904-fix8.apk`。

待真机回传：related 失败时错误详情文本（`runtimeType: ...`）。

## 11. fix9 — 白闪真因确认 / related 网络层准入修复

1. **白闪真因（fix7/fix8 方向性错误）**：白闪不是图片 crossfade——是 fix6 引入的
   **shuttle 末段 15% 淡出**：淡出期间 Hero 源/目标 child 尚未恢复显示，半透明
   shuttle 露出页面白底 → 点开/退出都闪白。**移除 shuttle 淡出**（瞬时交接 +
   参与 Hero 的全部 PixivImage fade:false，crossfade 帧为零，无白无跳变）。
2. **related 加载失败真因（截图定位）**：
   `PixivDestinationException: destination not allowed for PixivDestinationPurpose.appApi`
   ——NextPageParser 生成的是无 host 相对 URI；policy 层按目的地校验 host
   白名单时拒绝。推荐 repo 有"相对 URI 绑定 appApiBase"步骤，related repo 遗漏。
   → 照抄绑定：`request.uri.hasScheme ? request.uri : appApiBase.replace(path, query)`。
3. **related 原报错可见化**：错误分支显示 runtimeType + 摘要（本版真机已可读）。
- 25 项相关测试全过；APK `/root/PixivFunc-0904-fix9.apk`。

## 12. fix10 — related 端点实证修正 / 加载动画恢复 / 代理端口

1. **related 404 实证（拉取 pixivpy + pixez-flutter 源码确认）**：
   - pixivpy `aapi.py:328` → `https://app-api.pixiv.net/v2/illust/related`（**v2**，非 v1）；
   - pixez-flutter `api_client.dart:503` → `/v2/illust/related?filter=for_android`；
   - next_url 携带 `seed_illust_ids[]` / `viewed[]`（排除已浏览）→ 白名单放行；
   - filter 统一 `for_android`（与 pixez 及本 app search 一致）。
   - 404 兜底空页已按用户要求撤销（不得吞错/伪造"无相关作品"）。
2. **加载动画恢复**：撤销 3 处 fade:false（feed 卡/相关卡/详情主图）——白闪根源是
   shuttle 末段淡出（fix9 已移除），图片交叉淡入是占位灰↔图混合，不露白底。
3. **本机网络基建**：Clash Verge Rev（TUN 模式，Windows 侧），WSL 经镜像网络
   可达 `127.0.0.1:7897`；git 全局代理 7890（旧）→ 7897。
- 30 项相关测试全过；APK `/root/PixivFunc-0904-fix10.apk`。

## 13. fix11 — 返回闪白自适应 fade / 相关卡自适应比例 + hero 对齐

1. **返回闪白（用户定位命中）**：pop 落点图（previewUrl）已解码完成，但
   CachedNetworkImage 重挂载仍走 350ms 淡入 → 半透明帧透白底。
   **修复**：PixivImage 用 `PaintingBinding.imageCache.statusForKey` 同步判断
   图是否已解码完成（keepAlive/live）：**已完成 → fade 0（瞬时，无白闪）**；
   未完成/pending → 保留 350ms 灰占位↔图交叉淡入（加载动画不回退）。
2. **相关卡比例固定 + hero 瞬间裁切**：相关区 SliverGrid 固定 0.8 方图（cover）
   是"先原图缩小返回、再瞬间裁切"的根源。**修复**：相关区改
   `SliverMasonryGrid`（同主 feed）并**复用 IllustCard**（自适应作品比例
   fitWidth + 圆角 + 徽标 + 标题/作者/收藏）——卡片形状与详情页一致，
   pop 落点无裁切跳变；heroScope 传 'related:<parentId>:<id>' 与详情页精确对齐。
- 43 项相关测试全过；APK `/root/PixivFunc-0904-fix11.apk`。

## 14. fix12 — Hero 遮挡严格匹配着陆页 UI

1. **全局一刀切修正**：`_heroContentRect` 按**着陆路由**动态计算：
   - 顶部：读取着陆页 Scaffold AppBar 的 `preferredSize.height`——主页/详情页
     TabBar 嵌在 AppBar title 内（56）；个人页 AppBar + 独立 bottom TabBar
     （56+48=104）；无 AppBar 页 fallback 56——不再固定 56；
   - 底部：`route.isFirst`（主壳有底部导航才裁 45），推入页（详情/个人页/
     搜索/收藏等）无底栏 → 影像直达屏幕底——消除"下方 UI 框位置对不上/
     影像被凭空裁一截"。
2. 测试：新增"pop onto user page"三层导航用例（root → 用户页(104 chrome) →
   详情 → pop），断言 clip=(0,104,400,800)；push/pop 原用例保持。
- 40 项相关测试全过；APK `/root/PixivFunc-0904-fix12.apk`。

## 15. fix13 — Hero 遮挡规则定版（用户规则：严格被 UI 遮挡 + 不硬切）

用户定规矩：hero 飞入/飞出都要**严格被实际 UI 遮挡**，且**不能生硬裁切**（要优雅）。

实现（双端对称，不再只对着陆页）：
- 顶部 chrome = max(源 AppBar preferredSize, 目标 AppBar preferredSize)：
  主页/详情 56；个人页 56+48=104——飞行中影像绝不盖住任何一端的顶栏/TabBar；
- 底部 = 主壳参与（源或目标 route.isFirst）才裁 45——主页 push/pop 都被底栏
  遮挡；纯推入页间飞行（详情↔个人页）无底栏可达屏幕底；
- **优雅**：shuttle 外罩 `ShaderMask`（LinearGradient，顶部/底部 5% 渐隐）——
  影像接近 UI 边界时边缘淡出，不再是直线硬切。
- 测试：push 期望改回 (0,56,400,755)（源=主壳参与底部）；个人页用例
  (0,104,400,800)；全部通过。APK `/root/PixivFunc-0904-fix13.apk`。

## 16. fix14 — Hero 遮挡定版（用户规则：只匹配着陆页 UI）

用户规则（最终）：
- 去掉边缘渐隐（不搞"优雅"，好好遮挡）；四周圆角（ClipRRect 12）维持原样；
- 在哪里的 hero 就只匹配哪里的 UI（**着陆页**，不再取两端最大）。

关键修复：
- **个人页 TabBar 仍被盖的真因**：个人页用 **NestedScrollView**（pinned header 在
  headerSliverBuilder 中），此前只检测 CustomScrollView → 漏检 → TabBar 被飞行
  影像覆盖（用户截图）。`_pinnedHeaderChrome` 现在**两种容器都检测**
  （CustomScrollView.slivers / NestedScrollView.headerSliverBuilder），取 pinned
  header 收缩高度（minExtent − status inset）。
- landing 判定统一：push→详情页（AppBar 56、无底栏→屏幕底）；
  pop→主页（56 + 底栏 45 裁剪）或个人页（56+56 header、无底栏）。
- 测试：个人页用例改 NestedScrollView 真实结构 (0,112,400,800)；
  pop 主页带 pinned header (0,136,400,755)；push 详情 (0,56,400,800)。
- APK `/root/PixivFunc-0904-fix14.apk`；40 项相关测试全过。

## 17. fix15 — Hero 遮挡终版（双端 chrome + NestedScrollView 检测）

用户两次截图定位的遗漏（fix14 只匹配 landing 端的错误）：
- 个人页 **push**：飞行前半程在源页上方，越过其 pinned TabBar 行（截图 01:59:43）；
- 主页 **push**：下方越过底部导航（截图 02:00:56——fix14 landing=详情页无底栏 → 不裁）。

终版（综合全部规则）：
- **双端 chrome 取 max**：top = status + max(源(end appBar+pinnedHeader), 目标同)——
  push 不越过源页 TabBar/header、pop 不盖着陆页 TabBar；
- **底部**：主壳作为任一端（源或目标 route.isFirst）→ 裁底栏 45——
  主页 push/pop 均被底栏遮挡；纯推入页飞行（详情↔个人页）到底；
- **NestedScrollView 检测**（个人页 pinned header 真实容器）已包含在
  _pinnedHeaderChrome（CustomScrollView + NestedScrollView 双路径）；
- 无渐隐（纯裁剪）、ClipRRect 12 圆角保持。
- 测试：push nested feed 期望 (0,356,400,755)（源 profile header 300+56 参与）；
  4 用例全过，40 项相关测试通过。APK `/root/PixivFunc-0904-fix15.apk`。

## 18. fix16 — 复位瞬间底栏重叠（19px 修正）

现象：有底栏页面 pop 时，shuttle 即将复位（缩到卡片尺寸）前会超出底栏一点点，
完全复位后消失。

根因：`kHomeBottomNavHeight = 45` 是早期估算（BottomAppBar padding + icon），
但本项目 **useMaterial3: false（M2）→ BottomAppBar 默认高 64** ——clip 底沿比
底栏上沿低约 **19px**：飞行中 shuttle 底部总停在底栏重叠区，小尺寸（复位前）
时尤其明显；复位后 shuttle 被卡片替换，故"又正常了"。

修复：常量 45 → **64**（M2 BottomAppBar 默认高度，safe-area 另加）。
测试所有 755 期望同步更 736（800−64）。4 hero 用例 + 40 项相关测试全过。
APK `/root/PixivFunc-0904-fix16.apk`。

## 19. fix17 — Hero 全动态裁剪 + 上方 TabBar 均布

**Hero 两处静态裁剪根治（用户逐步定位）**：
1. **底部"少一截/超出一截"**：常量猜测（45→64）永远差几像素。改为
   **HomeShellMetrics 实测**：HomePage 给 BottomAppBar 挂 GlobalKey，
   postFrame 测量真实高度并发布；hero clip 读实测值（回退 64）。
   ——落点卡片底部在底栏上方的场景，shuttle 复位前不再被多裁。
2. **push 复位前"被 UI 错误遮挡"（详情页 56-112 被裁）**：顶部裁剪改为
   **随 shuttle 位置动态**：`_GlobalRectClip` 增加 srcTop/dstTop，
   paint 时 `shuttleTop > dstTop ? max(src,dst) : dst` ——飞行中源页高
   chrome 遮挡；shuttle 顶部越过目标页 AppBar 后只匹配目标页 chrome。

**上方 TabBar 均布**（用户指定两页）：
- 推荐页（插画/漫画/小说/用户）、新作页（关注/大家/好P友）：
  `isScrollable: true+TabAlignment.start` → `isScrollable: false`
  ——tab 平分宽度占满，与底部导航同一逻辑。

- 48 项相关测试全过；APK `/root/PixivFunc-0904-fix17.apk`。
