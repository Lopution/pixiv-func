# 源码证据与采用决策

## 判定标签

- **采用**：不改变 beta56 可见体验，且能直接增强一致性、失败恢复或可验证性。
- **受限采用**：只采用边界、状态机或测试思想；具体实现需按 Flutter/API 36 和本项目契约重写。
- **暂缓**：价值存在，但会扩大 Replica v1 行为或成本，先放入最终集成的显式决策 Gate。
- **拒绝**：与严格 TLS、安全存储、有界内存、真实失败或体验冻结冲突。

## 1. 认证、账号与 Token 刷新

Pixiv-Shaft 的并发测试明确覆盖旋转 refresh token 下“N 个线程只刷新一次”、旧 token 快路径以及失败等待者不能依次再次刷新：[SingleFlightTokenRefresherTest.kt L12-L105](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/test/java/ceui/pixiv/session/SingleFlightTokenRefresherTest.kt#L12-L105)。Replica 当前 `TokenRefreshGate` 已覆盖这些主要情形，说明既有规划方向正确。

反例同样重要：pixez-flutter 把 client identity/secret 写在源码并在 debug 模式记录 request/response body（[oauth_client.dart L34-L101](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/lib/network/oauth_client.dart#L34-L101)），刷新拦截器还会打印新 Authorization 值（[refresh_token_interceptor.dart L69-L137](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/lib/network/refresh_token_interceptor.dart#L69-L137)）；Pix-EzViewer 的 PKCE 是进程全局缓存且没有消费/清除语义（[Pkce.kt L10-L39](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/networks/Pkce.kt#L10-L39)）。

决策：

- **采用**：保留现有 per-account single-flight、一次重试、一次性 PKCE 和秘密/metadata 分离；在最终集成中复核“同一失败代”的所有等待者共享同一失败、账号切换时旧请求不得用新账号凭据。
- **拒绝**：从任何第三方仓库复制 client secret、hash salt、固定 User-Agent、全局 PKCE、BODY 日志或凭据数据库结构。

## 2. 大陆无外部代理连通性

pixez-flutter 已把网络拆为 `standard / ech / compat`。ECH 路径启用且要求 ECH，同时继续使用 WebPKI 证书校验（[pixez_network_settings.dart L7-L34](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/lib/network/pixez_network_settings.dart#L7-L34)）；但其 compat 路径关闭证书校验和 SNI，并把 API/OAuth/pximg 解析到本地缓存或固定 IP（[L37-L64](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/lib/network/pixez_network_settings.dart#L37-L64)、[hoster.dart L11-L45](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/lib/er/hoster.dart#L11-L45)）。这证明“按 host 选择 transport、把 ECH 作为独立能力”值得研究，也同时证明不能把一个活跃项目的 compatibility 模式整体照搬。

Pixiv-Shaft 把 cancellation、timeout、DNS、TLS、connect、socket 与一般 I/O 分成稳定类别，并明确 TLS 不可重试（[NetworkFailureClassifier.kt L13-L49](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/lisa/http/NetworkFailureClassifier.kt#L13-L49)）。它还为“无代理环境”按 app-api、Pixiv Web、pximg 等真实目标分别执行 DNS、TCP、TLS、网页请求和图片下载诊断，并在 ViewModel 清理时取消在途 client（[NetworkTestViewModel.kt L149-L180](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/pixiv/ui/debug/NetworkTestViewModel.kt#L149-L180)、[L226-L235](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/pixiv/ui/debug/NetworkTestViewModel.kt#L226-L235)）。其同一诊断文件也明确包含无 SNI、trust-all、固定 IP 与第三方代理路径，所以这里只采用 failure taxonomy、分目标真实探测、取消和脱敏思路。

Pix-EzViewer 的 DoH 代码把查询限制到 Pixiv API host 并保存 TTL cache（[NetworkMode.kt L243-L333](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/networks/NetworkMode.kt#L243-L333)），但默认 SNI replacement、可关闭 certificate/hostname verification 与硬编码 direct IP（[L67-L120](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/networks/NetworkMode.kt#L67-L120)、[L252-L281](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/networks/NetworkMode.kt#L252-L281)）。其 WebView interceptor 还会在失败后返回 `null` 交回原生栈，并在部分配置中 trust all（[WebViewBypassInterceptor.kt L27-L90](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/networks/bypass/WebViewBypassInterceptor.kt#L27-L90)），不符合 Replica 的 fail-closed 与严格 TLS 边界。

当前 Replica 自身还有一个比“选哪种 DNS”更基础的集成缺口：`PixivHttpClient`、`OAuthService`、`HttpDownloadTransport`、`CachedNetworkImage` 与 WebView 分属不同 transport 出口。只修改 API client 不能证明登录、图片和下载均可用。另一个技术推论是：loopback `CONNECT` 在不终止 TLS 时只能控制 DNS/连接 IP，不会改写或加密 WebView 发出的 ClientHello/SNI，因此它不能单独解决 SNI 阻断；这个边界必须写入设备 Gate。

决策：

- **采用（P0 产品 Gate）**：把“大陆用户无需外部代理/VPN”提升为父 PRD 目标；兼容网络从后置可选项提升为当前 Ugoira 工作边界之后的第一个基础任务，并由最终集成记录大陆真实网络矩阵。
- **采用（P0）**：建立 exact-host destination registry、stable failure taxonomy、direct-first route、network-revision/TTL health、脱敏诊断和一个共享 transport factory；API/OAuth/refresh、图片、下载/Ugoira、WebView 与 headless Widget 必须逐出口关闭证据。
- **受限采用（P0 research）**：secure DNS + candidate-IP direct connector 始终保留原 hostname TLS；ECH 只有 endpoint/transport/API 36 spike 证明 `require ECH`、系统信任链、取消/流式/连接池后才进入依赖。
- **受用户决策约束**：默认自动 fallback 与 WebView loopback `CONNECT` 只有在用户确认“不使用代理”指无需外部代理/VPN时才采用；否则只保留纯直连/平台原生 ECH。
- **拒绝**：证书/hostname verification 关闭、SNI 替换/清空、Host 改写、domain fronting、全局 `HttpOverrides`、固定 IP 安全核心、第三方 API/图片反代、BODY/Authorization 日志和把单一网络样本宣传为大陆普遍可用。

## 3. Feed 分页、跨代写回与首屏缓存

Pixiv-Shaft 的 feed controller 明确规定 refresh 取消旧加载并始终获胜，append 游标只属于当前代（[FeedViewModel.kt L70-L185](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/feeds/src/main/java/ceui/pixiv/feeds/FeedViewModel.kt#L70-L185)、[L201-L285](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/feeds/src/main/java/ceui/pixiv/feeds/FeedViewModel.kt#L201-L285)。其测试专门覆盖相同、删除、新增、完全不相交和 survivor 重排的跨代替换（[CrossGenerationSwapTest.kt L26-L97](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/feeds/src/test/java/ceui/pixiv/feeds/CrossGenerationSwapTest.kt#L26-L97)）。首屏缓存则同时绑定 slot、账号和响应类型，带 schema、年龄与体积上限，并让损坏数据退化为 miss（[FeedFirstPageCache.kt L21-L132](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/feeds/src/main/java/ceui/pixiv/feeds/cache/FeedFirstPageCache.kt#L21-L132)）。

当前仓库的 `PagedFeedController` 已有活动请求 token，但 `RecommendedIllustRepository.fetchPage` 没有把 cancel token传到底层；多个 repository 还会在 controller 判断“是否仍是当前请求”之前先 merge 共享实体。因此旧代请求即使不再更新 ID 列表，仍可能把旧实体 snapshot 写入 `IllustStore`。

决策：

- **采用（P0）**：为 Recommended/Ranking/New/Search/Profile feeds 建立 generation/request context；网络、解析、共享 store commit 和 cursor commit 必须属于同一活动代。refresh 取消 append，旧代结果不得写实体或 cursor。
- **采用（P0）**：增加 same IDs、删除、新增、重排、完全不相交、账号/筛选切换与“取消后仍返回”的测试。
- **暂缓（P2）**：版本化、账号隔离的首屏 cache 只作为冷启动优化；先用设备数据证明收益，并设置 payload/age/LRU 上限，不能因此延长旧账号内容显示。

## 4. 收藏、关注、评论等写操作

Pixiv-Shaft 的 action queue 具有 owner 隔离、按 dedupe key 合并、严格入队顺序、进程恢复、Retry-After/整队冷却和 superseded failure 判定（[ActionQueue.kt L55-L172](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/actionqueue/src/main/java/ceui/pixiv/actionqueue/ActionQueue.kt#L55-L172)、[L347-L472](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/actionqueue/src/main/java/ceui/pixiv/actionqueue/ActionQueue.kt#L347-L472)）。这是很成熟的“用户意图”模型。

但 Replica v1 已冻结为请求期间显示 pending、服务端成功后才改变 confirmed 状态。默认把失败收藏持久化并在数分钟或重启后自动重放，会改变 beta56 可见失败语义，也可能在用户已切换意图后产生意外写入。

决策：

- **受限采用（P1）**：吸收 owner/account isolation、dedupe key、superseded revision、429/Retry-After 分类、终态失败可见和存储故障不吞动作的测试思想。
- **拒绝（Replica v1）**：不引入默认离线持久 outbox/后台自动重放；评论、资料编辑等非幂等操作尤其禁止隐式重试。
- **暂缓（Evolution）**：若未来产品明确需要离线意图队列，必须单独审批可见 pending/failed/retry UX、幂等键和账号退出策略。

## 5. 反向图片搜索

Pixiv-Shaft 记录了 headless multipart + HTML replay 已被 Cloudflare challenge 阻断，真实 WebView 能执行 challenge，但仍可能被反复质询；其准备步骤还强调 cloud `content://` 读取必须离主线程并受 15 MB 上限约束（[ReverseImage.java L14-L71](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/lisa/utils/ReverseImage.java#L14-L71)）。

Pix-EzViewer 则展示了应拒绝的旧路径：BODY 日志、固定 IP DNS fallback、整张 Bitmap decode、异步压缩竞态、HTML scraping 和废弃的 `MediaStore.DATA`（[SaucenaoActivity.kt L82-L181](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/ui/settings/SaucenaoActivity.kt#L82-L181)、[L207-L272](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/ui/settings/SaucenaoActivity.kt#L207-L272)）。

决策：

- **采用（P0）**：provider research 必须产出 `structuredApi / interactiveWebView / unavailable` 能力结果和观测日期；输入准备使用受限 stream/temp、真实格式与 pixel/file caps、所有终态 cleanup。
- **受限采用**：WebView 可以作为“服务现实边界”的候选，但不能被静默替换成 beta56 结果卡体验；若唯一可行路径只能让用户在第三方网页中交互，必须作为产品差异重新审批或保持 blocker。
- **拒绝**：HTML replay、DOM 抓凭据、伪造 UA 绕 challenge、固定 IP、全图内存解码和未知 scheme 自动打开。

## 6. Profile edit 与能力模型

Pixiv-Shaft 会先读取 `canChangePixivID`、`hasPassword` 等服务端能力，且已停止预填密码（[FragmentEditAccount.java L18-L54](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/lisa/fragments/FragmentEditAccount.java#L18-L54)）。同时，旧 Fragment 的大量字段组合分支和本地保存密码展示了维护风险（[L57-L233](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/lisa/fragments/FragmentEditAccount.java#L57-L233)）；同一 endpoint 其实可以用 nullable patch 字段统一表达（[SignApi.kt L9-L29](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/lisa/http/SignApi.kt#L9-L29)）。

决策：

- **采用（P0）**：Profile task 使用服务端 `ProfileCapabilities` + typed `ProfilePatch`，只发送 dirty 字段；提交结果区分 `confirmed`、`verificationPending` 和 field errors。
- **采用**：密码只作为当前提交的短生命周期输入，永不预填、持久化、进入 draft snapshot 或日志。
- **拒绝**：按字段组合复制分支矩阵、先改本地 store 后等待服务端、或保存明文密码。

## 7. Novel 标记与长文本

Pix-EzViewer 将 parser 做成零 Android 依赖的纯 Kotlin，并用 typed Text/Image chunks 与 Chapter/Ruby/JumpUri/JumpPage tokens 表达 Pixiv 标记；每块 3000 字符上限用于避免超长布局阻塞（[NovelMarkup.kt L3-L96](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/ui/novel/NovelMarkup.kt#L3-L96)）。其 WebView HTML 抽取 fallback 本身仍是脆弱反例（[NovelViewModel.kt L32-L66](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/ui/novel/NovelViewModel.kt#L32-L66)）。

当前 Replica Novel 已有 typed paragraph、ruby/jumpuri 基础和可取消布局，但尚未完整表达 `[newpage]`、`[chapter:]`、`[jump:]`、`[pixivimage:]`、`[uploadedimage:]`，而布局仍逐段在 UI isolate 上用 `TextPainter` 测量。

决策：

- **采用（P0）**：在最终集成前补齐纯 Dart typed parser、未知 token 保留、page/image blocks、chunk 上限与 fixture 测试；解析与图片解析不得依赖 Widget。
- **受限采用**：Flutter 字形测量仍需 UI isolate 时，使用预算切片、generation cancellation 和可测的最大单批工作量；不能声称 `Future.delayed(Duration.zero)` 等于真正离主线程。
- **拒绝**：把 HTML scraping 恢复为默认正文来源，或把未知标记静默删为空文本。

## 8. 下载、任务组与进程恢复

PixivBiu 把一个作品建模为多 Task 的 Job，提交时冻结 Ugoira format（[types.go L46-L105](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/download/types.go#L46-L105)）；Manager 使用有界队列、不可变配置快照，并在启动时把遗留 running 重置为 queued 后恢复（[manager.go L25-L60](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/download/manager.go#L25-L60)、[L161-L219](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/download/manager.go#L161-L219)）。进度节流不会吞掉 0%/100% 终态（[publisher.go L90-L153](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/download/publisher.go#L90-L153)）。

Pixeval 进一步证明多页/动图需要 task-group 聚合状态和 exactly-once 后处理（[DownloadTaskGroup.cs L70-L101](https://github.com/Pixeval/Pixeval/blob/d1b9cafe48ab6b9817e92feb8aa9f1238d698050/src/Pixeval/Models/Download/Tasks/DownloadTaskGroup.cs#L70-L101)、[L168-L213](https://github.com/Pixeval/Pixeval/blob/d1b9cafe48ab6b9817e92feb8aa9f1238d698050/src/Pixeval/Models/Download/Tasks/DownloadTaskGroup.cs#L168-L213)），单任务只在临时输出完整后 commit，并只删除自己拥有的文件（[ImageDownloadTask.cs L43-L63](https://github.com/Pixeval/Pixeval/blob/d1b9cafe48ab6b9817e92feb8aa9f1238d698050/src/Pixeval/Models/Download/Tasks/ImageDownloadTask.cs#L43-L63)）。

决策：

- **采用（P1）**：将多页作品/Ugoira 导出显式建模为 group/job，提交时冻结命名、目标和设置快照；group 后处理只执行一次，terminal event 绕过节流。
- **采用（P0）**：临时/pending output 只在完整成功后 commit，cleanup 只能删除任务拥有的输出；碰撞策略必须确定且可恢复。
- **暂缓（P1 Gate）**：Replica v1 是否承诺跨进程自动续传需结合 beta56 行为和 MediaStore 设备结果决定；至少必须持久化足以解释遗留 pending item 的状态，重启后不能显示仍在运行的幽灵任务。

## 9. Ugoira 安全与资源上限

PixivBiu 在解码前验证格式、frame 存在性、单帧压缩字节、header-only dimensions 和格式 allowlist，并在失败/取消时删除未完成输出（[ugoira.go L22-L117](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/download/ugoira.go#L22-L117)、[L119-L219](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/download/ugoira.go#L119-L219)）。但它随后把所有 decoded images 收进 slice；这对移动端仍不可接受。

pixez-flutter 同样展示了全 ZIP `readAsBytesSync`、同步解压、未 await 下载和全文件导出的风险（[ugoira_store.dart L58-L99](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/lib/page/picture/ugoira_store.dart#L58-L99)、[L145-L230](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/lib/page/picture/ugoira_store.dart#L145-L230)）。其 route-aware stop/resume 值得借鉴，但缓存淘汰没有明确 `ui.Image.dispose`（[ugoira_painter.dart L44-L113](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/lib/component/ugoira_painter.dart#L44-L113)）。

决策：

- **采用（P0）**：把 archive 总压缩/解压字节、entry 数、重复/路径、frame 数、单帧 bytes、width×height 和总 pixel budget 写成 typed limits；header 通过后才分配像素。
- **采用**：播放和导出继续使用滑动窗口/流式 encoder、route/app lifecycle cancellation、逐帧 cleanup 和输出 commit。
- **拒绝**：所有 frame bytes/images 常驻内存、同步 IO、损坏 delay 静默改成默认而仍声称精确播放。

## 10. Android Widgets

Pixiv-Shaft 使用 unique WorkManager、网络约束、移除最后 Widget 时取消周期工作，并避免 resize 事件造成 REPLACE thrash（[RecommendCardWidgetProvider.kt L20-L80](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/pixiv/widget/RecommendCardWidgetProvider.kt#L20-L80)）。其 renderer 根据 Widget size 计算 bitmap，并为 RemoteViews IPC 设置硬 pixel cap；瞬时失败保留 last-good 内容（[BaseStripWidgetWorker.kt L42-L70](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/pixiv/widget/BaseStripWidgetWorker.kt#L42-L70)、[L86-L159](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/pixiv/widget/BaseStripWidgetWorker.kt#L86-L159)）。

pixez-flutter 提供了更安全的数据边界方向：native 只读包含作品 ID、标题、作者和图片 URL 的本地数据库，不读取 token（[GlanceDBManager.kt L7-L57](https://github.com/Notsfsssf/pixez-flutter/blob/837f95836ee674ecfa996c2603e20bf9197ff9ee/android/app/src/main/kotlin/com/perol/pixez/glance/GlanceDBManager.kt#L7-L57)）。其原始表没有 schema envelope/事务，仍需重写。

决策：

- **采用（P0）**：优先使用 Flutter 生成的 versioned、account-revision keyed、无秘密 Widget snapshot；账号切换/reauth 立即清空旧 snapshot。
- **受限采用**：若要在 app 未运行时保持 beta56 的网络刷新，必须证明 headless Flutter 能复用同一 AccountStore/TokenRefreshGate；不另写 native token/refresh 栈。证明失败则任务保持 blocker，不能降级到普通 prefs token。
- **采用**：unique work、last-good vs account-invalid 的不同清理策略、bitmap/IPC budget、immutable + uniquely identified PendingIntent 和 resize 去抖。
- **拒绝**：Widget receiver 直接网络写收藏、native 读取 Flutter 明文账号 JSON、MainScope 常驻请求或 token 放入 WorkData/RemoteViews。

## 11. Updater 供应链

PixivBiu 将签名 manifest 作为 release source of truth，签名验证成功后才解析版本、URL、size 和 SHA-256；没有可信公钥或签名无效时 fail closed，并对 manifest/signature response 限长（[manifest.go L15-L87](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/update/manifest.go#L15-L87)、[L89-L143](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/update/manifest.go#L89-L143)）。版本选择还会拒绝 dev build、不适配平台的 asset，并明确 channel maturity（[checker.go L14-L60](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/update/checker.go#L14-L60)、[L111-L157](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/update/checker.go#L111-L157)）。测试覆盖并发 apply 与 checksum mismatch（[apply_test.go L22-L79](https://github.com/txperl/PixivBiu/blob/1628d9868d70a5e5934d7536f82331f9178d170f/internal/update/apply_test.go#L22-L79)）。

Pixiv-Shaft 的 GitHub updater 能恢复 DownloadManager ID，但版本解析和任意 APK asset fallback 都不构成真实性保证（[AppUpdateChecker.kt L12-L119](https://github.com/CeuiLiSA/Pixiv-Shaft/blob/7e3e1a8cf3322503f727f665d8a67762387ea69f/app/src/main/java/ceui/lisa/update/AppUpdateChecker.kt#L12-L119)）。

决策：

- **采用（P0）**：GitHub flavor 使用 CI 生成的 signed manifest；App 内固定公钥，只在验签后信任 repo/tag/asset URL/size/SHA-256/channel。私钥只存在外部 release 环境。
- **采用**：APK 下载后必须同时满足 exact size、SHA-256、包名和 signing certificate 与已安装应用匹配，再交给 Package Installer；check/apply single-flight，状态可恢复。
- **采用**：F-Droid flavor 在依赖、组件、权限和网络路径上编译期剔除 updater，而非运行时空操作。
- **拒绝**：把未签名 GitHub JSON 或“若有 checksum”当安装信任根、挑第一个 `.apk`、静默安装或签名不匹配时放宽校验。

## 12. History、Settings 与 Search form

Pixeval 的 History helper 集中恢复、所有权和批量提交，未被 queue 接受的 task 会被调用方释放（[HistoryPersistHelper.cs L20-L85](https://github.com/Pixeval/Pixeval/blob/d1b9cafe48ab6b9817e92feb8aa9f1238d698050/src/Pixeval/Models/Database/Managers/HistoryPersistHelper.cs#L20-L85)、[L156-L212](https://github.com/Pixeval/Pixeval/blob/d1b9cafe48ab6b9817e92feb8aa9f1238d698050/src/Pixeval/Models/Database/Managers/HistoryPersistHelper.cs#L156-L212)）。其 Search form 在构造请求前集中验证日期、premium sort 与数值范围（[SearchArgumentsFormViewModelBase.cs L13-L72](https://github.com/Pixeval/Pixeval/blob/d1b9cafe48ab6b9817e92feb8aa9f1238d698050/src/Pixeval/ViewModels/Search/SearchArgumentsFormViewModelBase.cs#L13-L72)、[IllustrationSearchFormViewModel.cs L37-L70](https://github.com/Pixeval/Pixeval/blob/d1b9cafe48ab6b9817e92feb8aa9f1238d698050/src/Pixeval/ViewModels/Search/IllustrationSearchFormViewModel.cs#L37-L70)）。

Pix-EzViewer 的 Room singleton/migration 是正向基础，但 downgrade 时 destructive fallback 不适合 Replica 用户数据（[HistoryDatabase.kt L38-L69](https://github.com/ultranity/Pix-EzViewer/blob/96a4f4b42df82c679d43c5da9283537ce7531590/app/src/main/java/com/perol/asdpl/pixivez/data/HistoryDatabase.kt#L38-L69)）。

决策：

- **采用（P1）**：最终集成复核 restore/cancel/dispose ownership、版本迁移与 downgrade 明确失败；禁止 destructive migration。
- **采用**：Search 参数继续使用 typed query 和一次性 validation，服务端 capability 不可用时控件明确禁用，不能发出随后必失败的请求。

## 13. Live

在五个固定版本的所选源码中，没有找到可作为当前 Pixiv Sketch Live list/detail/HLS 链路证据的完整实现。这个“未找到”不能证明服务不可用，但足以证明不能因为其他客户端功能丰富就降低 Live research gate。

决策：

- **采用（P0 Gate）**：先完成当天 endpoint/auth/schema/HLS 可用性证据，再引入 player 依赖和 UI 实现。
- **拒绝**：使用旧 fixed-IP/local proxy、以静态 fixture 宣称真实 Live 可用，或在 endpoint 不可用时留下 mock 播放器。

## 总结

参考项目证明 Replica 当前“共享实体、严格 TLS、安全凭据、single-flight、有界媒体”总体方向是对的。真正需要写回规划的不是更换状态管理框架，而是补齐大陆无外部代理的 shared transport/真实网络 Gate、跨代 commit 边界、typed Novel markup、任务组/恢复语义、Widget 无秘密快照、signed updater manifest，以及把反向搜图和 Live 的外部能力失败变成显式 Gate。
