# 修复无代理网络与作品转场问题

## Goal

在大陆直连、无外部代理的真实手机环境中，让 Pixiv 请求能够更快地选到实际可用的安全路径，并让网络探测结果反映“哪条路径可用”；同时修复作品列表、作品详情和下拉刷新中的转场与时序问题。用户应能看到稳定、连续的首图/头像/详情内容，返回时不出现跨页面误飞、越界或随机降级为普通滑动的动画。

本任务延续已合入的 ECH/DoH 路由阶梯和作品 Hero 作用域修复，不改变严格证书校验、目的地主机白名单或“外部代理不是默认出口”的安全边界。

## Background and confirmed facts

- 真实设备探测中，`app-api.pixiv.net`、`oauth.secure.pixiv.net` 和 `i.pximg.net` 的真实 SNI TLS/直接最小请求失败，但 ECH 请求收到 HTTP 404（说明 ECH 传输已到达服务端）；当前 `NetworkProbe._classify` 在 DNS 地址集合不一致时提前返回 `dnsPolluted`，因此把可用的 ECH 路径显示成“DNS 污染”。相关代码：`lib/core/network/compat/network_probe.dart:163-168,325-378`、`lib/features/settings/network_probe_page.dart:64-177`。
- `DohResolver` 每次 ECH 配置查询都重新走 DoH，`NetworkAccessPolicy` 的路由记忆只在进程内保存 10 分钟；首次请求会先探测直连再探测 ECH，墙内网络因此会承担不必要的等待。相关代码：`lib/core/network/compat/secure_resolver.dart:138-471`、`lib/core/network/compat/network_policy.dart:309-407,623-677`。
- Shaft 的源码把 API（Cronet/QUIC）与图片（独立 OkHttp/HTTP1.1、DoH/源站地址）分开；PixEz Flutter 使用共享 Rust/rhttp 客户端、ECH 配置 TTL 缓存和按主机的路由模式；Pix-EzViewer 另外证明了“图片预加载 + 首张图/头像加载完成后再开始 shared-element 转场”可以避免半成品首帧。对比快照：`CeuiLiSA/Pixiv-Shaft@42e0e2cfcc4d3cb5dceab3667aa30e7e868287dd`（2026-08-31）、`Notsfsssf/pixez-flutter@e45e6b3936df391f802fdcb33ef4fb818ef2e151`（2026-08-28）、`ultranity/Pix-EzViewer@96a4f4b42df82c679d43c5da9283537ce7531590`（2026-07-04）。本任务主要采用 Shaft/PixEz 的分层与缓存思想，不引入其不适合当前安全边界的全局证书绕过或第三方镜像。
- 多图详情首图当前在 `metaPages` 缺失时回退到 `imageUrls.medium`，而信息流默认使用 `imageUrls.large`；来源和目标 URL 不一致会让 Hero 先飞灰色预占图或回落到低清图。相关代码：`lib/features/home/recommended/recommended_illust_page.dart:199-203,244-253`、`lib/features/illust/detail/illust_detail_page.dart:299-327`。
- 推荐/新作/排行/搜索等存活 tab 使用短的固定 Hero scope（如 `'recommended'`、`'new'`、`'ranking'`、`'search'`）；多个存活列表可能同时出现相同作品 ID，导致 Hero tag 重复或目标在异步重建时不稳定。相关调用点：`lib/features/home/recommended/recommended_home_page.dart:278-281`、`lib/features/new/new_page.dart:265-277`、`lib/features/ranking/ranking_page.dart:152-155`、`lib/features/search/search_result_page.dart:152-154`。
- Flutter `RefreshIndicator` 在进入 armed 状态后会保持最小指示器值，当前 14 个使用点没有在反向 overscroll 时重置；因此用户把手指向上拉回时刷新标志仍可见。使用点包括 `history_page.dart`、`comments_page.dart`、`profile/user_page.dart`、`new_page.dart`、`ranking_page.dart`、`search_result_page.dart`、`recommended_home_page.dart`、`recommended_illust_page.dart`。

## Requirements

### N1. 无代理网络路径的可靠性与首请求速度

- 在自动模式下，保留干净网络的直连优先；一旦当前 `NetworkRevision` 测得某个 Cloudflare 主机的 ECH 路径成功，后续同组请求应优先复用仍在 TTL 内的 ECH 配置/路由，避免每次先等待必败的直连和重复 ECH HTTPS-RR 查询。
- ECH 配置缓存必须受 DNS RR TTL、网络 revision 和设置变更约束；网络切换、DoH/ECH 设置变化或过期后不得复用旧配置。缓存只存运行时非敏感连接材料，不持久化未经验证的 IP。
- 失败仍需可观察并按现有阶梯降级到严格 DoH/真实 SNI、图片空 SNI（及用户显式开启的不校验证书兜底）；不得把超时静默转换为空数据、不得默认启用证书绕过、固定 IP、第三方代理或镜像。
- API 与图片继续使用各自的目的地策略和共享连接池；本次不引入 QUIC/HTTP3 或重写 rhttp 协议栈。

### N2. 网络探测结论必须表达可行动路径

- 如果 ECH 请求已经完成 TLS/HTTP 传输（包括 HTTP 404/403），结论应优先显示“应选 ECH”；如果 ECH 不可用但空 SNI 请求可达，则显示“应选空 SNI”。
- 系统 DNS 与 DoH 地址不一致只作为附加诊断信息，不得覆盖一个已验证可用的 ECH/空 SNI 结论；HTTP 421 仍表示该空 SNI 路由不可用。
- 探测页和复制文本需同时展示主结论、DNS 地址差异（若存在）及各层原始状态，让用户能区分“直连失败但替代路径可用”和“所有路径失败”。

### U1. 反向下拉可取消视觉刷新

- 用户下拉达到刷新阈值后，在松手前向上拉回，刷新标志必须收起，且不触发 `onRefresh`。
- 松手后真正开始的刷新请求不被 UI 重置取消；请求完成/失败的现有 feed 状态和错误尾部保持不变。
- 推荐、排行、新作、搜索、个人页、历史和评论等现有刷新入口行为一致，不复制每个页面的私有实现。

### U2. 首图与详情 Hero 使用同一可见资源

- 进入多图作品时，Hero 来源和详情首个目标都使用该作品第一张可见预览图，不使用灰色占位节点。
- 来源与目标沿用同一 URL/质量选择；默认质量不得低于信息流当前预览质量。详情加载到完整 `metaPages` 后，其余页面仍按各页尺寸和 URL 展示。
- 没有详情快照时仍可正常进入；有信息流快照时首帧必须先显示快照内容，后台详情刷新不得替换成空白或延迟插入 Hero。

### U3. Hero 作用域稳定且飞行范围受控

- 每个存活信息流实例（tab、模式、搜索词、个人页筛选）拥有稳定且互不冲突的 scope；来源卡片与它创建的详情路由传递同一 scope。
- 无匹配来源时允许普通右侧滑入/返回，但不得复用上一个作品或其它页面的 Hero 节点。
- 进入和返回的飞行只裁剪在作品卡片/详情作品容器内；不出现先把整张图片飞出页面再由父布局截断的越界画面，现有圆角视觉保持连续。

### U4. 详情首帧立即创建作者头像

- 点击作品后立即 push 详情路由；不得等待网络图片预加载完成才开始右侧滑入转场。
- 详情首帧与标题、画师名同时创建头像的 `ImageProvider`，并在按下/点击开始时以同一 `CacheManager`/headers 启动头像预热；命中缓存时首帧直接显示真实头像。
- 未命中缓存时保留头像位置和现有 placeholder/errorWidget，图片返回后在当前详情页补齐；不得因为头像请求失败阻塞导航或引入额外重试。
- 不改变图片请求的 Referer、缓存管理器或网络策略。

本条保证范围：保证 provider、头像容器尺寸和标题/画师名在首帧同时存在，保证缓存命中时首帧出现真实头像；不承诺在没有缓存且网络尚未返回时凭空显示远端头像像素。若产品要求后一项，必须改为列表阶段提前预取或等待网络，属于不同的交互/性能取舍。

## Constraints and user decisions

- 主要参考 Shaft 与 PixEz；Pix-EzViewer 仅用于补充 shared-element/预加载和 DNS×SNI 经验。
- 保留现有 Hero（否则页面会割裂），不改 `ReplicaPageRoute` 的 300ms 右侧滑入/返回节奏。
- 不写过度的防御性代码：只增加能直接对应上述时序、路由失效或可观察性问题的状态和测试。
- “立即转场”是既定交互：按下时先启动首图/头像预热但不等待，抬起后立即 push；标题、画师名和头像占位在首帧一起出现。若头像已经在共享缓存中，首帧显示真实头像；网络未命中时只允许头像内容稍后补齐，不得用等待换取导航响应。

## Acceptance Criteria

- [ ] AC-N1：在模拟“直连 TCP/TLS 超时、ECH HTTP 404”的网络矩阵中，首次请求只为当前 revision 建立一次 ECH 配置；后续 TTL 内请求跳过重复 ECH 查询并优先使用 ECH；revision/设置变更后缓存失效。严格证书校验和显式不校验开关测试继续通过。
- [ ] AC-N2：网络探测离线测试覆盖“DNS 不一致 + ECH 成功”“ECH 404/403 传输成功”“空 SNI 421”“所有路径失败”；前两者结论分别为 `echAvailable`/可行动替代路径，复制文本包含 DNS 差异和原始层状态。设置页不再把可用 ECH 路径只标成“DNS 污染”。
- [ ] AC-U1：共享刷新组件的 widget 测试证明 armed 后反向 overscroll 会移除指示器且不调用刷新；正常下拉松手仍调用一次。所有现有 RefreshIndicator 调用点迁移且行为一致。
- [ ] AC-U2：多图实体（有/无 `metaPages`）测试证明来源和详情首个 Hero 使用同一第一张预览 URL，首帧存在 Hero 目标且质量不低于来源；单图和下载/查看器行为不回归。
- [ ] AC-U3：tab/模式/搜索/个人页 scope 测试证明同 ID 在不同存活列表不会匹配 Hero；Hero flight 测试或 golden/布局断言证明飞行内容被作品容器裁剪，普通无来源入口仍可滑入。
- [ ] AC-U4：详情导航 widget 测试证明按下/点击后立即开始 push（不等待预热 Future），首帧同时创建标题、画师名、头像 provider 与 Hero 目标；缓存命中时头像首帧可见，缓存未命中/失败时位置稳定且随后可补齐，网络失败不会永久卡住。
- [ ] AC-Q：执行 `flutter analyze --no-pub`、相关 `flutter test --no-pub`、`git diff --check`；若环境提供 Android 工具则构建 debug APK。真实手机无代理复测单独记录为 Device-tested，不以离线测试代替。

## Out of Scope

- 引入 Cronet/QUIC/HTTP3、SNI 替换、第三方图片镜像、WebView 全面代理或新的证书绕过；这些属于独立网络方案评估。
- 把所有网络地址持久化、扩大 Pixiv 主机白名单、并行竞速所有 IP、重写 rhttp fork 或将业务 POST 当探测/自动重放。
- 重写全局 Navigator、`ReplicaPageRoute` 曲线或创建与本任务无关的通用缓存/动画框架。
- 在刷新已经释放后取消服务器请求；本任务只处理阈值前的视觉取消。

## Resolved product decision

- 已确认 U4 采用“立即转场、非阻塞头像预热”：不等待网络再 push；冷缓存头像允许在详情转场中补齐，但头像容器、标题和画师名必须首帧稳定存在。这样保留当前点击响应，不引入列表阶段批量预取或额外点击延迟。

## References

- [Pixiv-Shaft](https://github.com/CeuiLiSA/Pixiv-Shaft) — API/图片分层、DoH/健康检查和网络自检；快照 `42e0e2cfcc4d3cb5dceab3667aa30e7e868287dd`。
- [pixez-flutter](https://github.com/Notsfsssf/pixez-flutter) — rhttp/ECH、共享客户端、ECH/质量缓存；快照 `e45e6b3936df391f802fdcb33ef4fb818ef2e151`。
- [Pix-EzViewer](https://github.com/ultranity/Pix-EzViewer) — DNS×SNI 路由模型、首图和头像完成后启动 shared-element；快照 `96a4f4b42df82c679d43c5da9283537ce7531590`。
