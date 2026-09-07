Pixiv Func 当前问题审计、产品决策与简要解决方案
================================================
审计日期：2026-09-01
审计基线：Lopution/pixiv-func
当前 HEAD：9d1cb1b809e20b67e2409970938b221e178d292e
主要代码提交：2eaf5aa38b17a413fcf64419475ec3467fe8cb4f

说明
----
这份清单合并了三类证据：
1. 用户在真实 Android 设备上的直接使用反馈；
2. 当前 HEAD 的代码审计；
3. PixEz、Pixiv-Shaft、SauceNAO、国内翻译服务、Android/GitHub 官方资料的对照调研。

本清单遵循 Func 已确定的工程原则：
- Modern by default, compatible by design, graceful by degradation.
- Minimum Necessary Defense.
- 只修有真实产品后果、真实协议要求或已复现证据的问题。
- 不因为“理论上可能”增加状态机、lease、epoch、journal、allowlist 层等基础设施。
- 错误 guard 阻塞正常行为时优先删除/放松 guard，而不是再包一层补丁。
- 测试必须覆盖正常用户 happy path；“测试全绿”不能证明一个臆造的限制就是正确产品行为。
- 不为“暂时不做某件事”建设基础设施。


================================================
一、已经确定的产品边界
================================================

1. Android 基线
- Android 10 / API 29。
- API 29 上核心功能必须完整可用。
- 旧平台允许合理体验降级，但不能为了低版本兼容增加大量双实现。

2. Live
- Live 已移出 Replica v1。
- 后续任务不要把 Live 重新作为 1.0 blocker。

3. 中国大陆网络边界
- 首次注册、首次 OAuth WebView 登录的网络环境由用户负责。
- Func 不承诺“大陆新用户零代理完成首次注册/首次登录”。
- 用户取得有效凭据以后，API、refresh、图片、下载、收藏、关注、评论等是 Func 的责任。
- 目标：登录后在中国大陆尽可能无需外部代理/VPN完成日常使用。
- 不做 VpnService、系统代理、Private DNS 修改、hosts、root 依赖、自建远端 relay/proxy。
- Cronet / QUIC / HTTP/3 没有新的实测证据前不扩 scope。

4. TLS
- API/OAuth/账号接口必须保持真实服务器身份验证。
- 不做 MITM，不安装自定义 CA。
- 如果后续真机 A/B 证明 Pixiv 公共图片 CDN 的 PixEz-style 宽松 transport 对兼容性/速度有明显价值，可以把风险严格限制到公开图片 CDN（例如 i.pximg.net / s.pximg.net）内部使用。
- 不应再暴露一个“对所有 Pixiv host 关闭证书验证”的全局普通用户开关。

5. 非 parity blocker
- 原项目本来就没完成的 DM、发布作品、pixivision 等不应为了“完整度”被临时加入 1.0。


================================================
二、用户已经做出的 D1–D7 决策
================================================

D1 反向搜图
--------------
决定：
- 使用零配置 SauceNAO。
- 结果不限制 Pixiv，可以展示 SauceNAO 返回的其它来源。

推荐首版实现：
- 参考 PixEz：选择图片 -> 本地压缩 -> multipart POST 到 SauceNAO search.php -> 展示 SauceNAO 返回结果。
- PixEz 当前做法是把较大图片缩至约 720px、JPEG quality 75，再 POST 到 SauceNAO，然后在 WebView 中显示返回 HTML。
- 为避免重复造一个脆弱的 HTML parser，Func 首版也可以直接使用结果 WebView：
  - Pixiv 链接：拦截并进入 Func 原生作品/作者页；
  - 其它来源：保留 SauceNAO 页面导航或交给外部浏览器。
- 如果未来确实需要原生结构化结果，再基于实际 SauceNAO HTML/API 做，不要现在继续扩 provider framework。

产品行为：
- 选择图片后先显示预览；
- 用户确认搜索后才上传；
- 失败/限流后清理临时文件；
- 不伪装成“Func 自己的离线识图”。

说明：
- SauceNAO 官方本身索引 Pixiv、Danbooru、Anime、MangaDex、ArtStation、Twitter、Skeb 等大量来源，所以 D1 的多来源需求天然满足。
- SauceNAO 服务条款明确不同登录/账户类型存在请求限制，服务也可能变化；零配置 HTML 路径应把“服务暂不可用/请求受限”当普通错误处理。
- 不要为了请求限制造多 IP/多账号绕限机制。

调研参考：
- SauceNAO 首页：https://saucenao.com/
- SauceNAO Terms：https://saucenao.com/legal.html
- PixEz：lib/page/webview/saucenao_webview_page.dart


D2 翻译
--------
决定：
- 找中国大陆可正常使用的 Provider，优先免费额度大。
- 增加“用户填写 API Key，通过 LLM 翻译”的能力。

2026-09-01 调研结果：
A. 腾讯云 TMT
- 腾讯云 2026-07-15 的计费页仍写“文本翻译每月 500 万字符免费”。
- 但腾讯云官方 API 更新历史显示：2026-07-08 第 17 次发布已经删除 TextTranslate。
- 当前 API 概览只列端到端图片翻译。
结论：
- 不能因为“500 万/月”看起来最大就选腾讯云 TMT 文本翻译；它当前已经不是可靠可接入的文本 API。

B. 百度翻译开放平台
- 通用文本翻译仍是当前公开 API。
- 标准版：5 万字符/月，QPS 1，无需认证。
- 高级版：个人认证后 100 万字符/月，QPS 10。
- 尊享版：企业认证后 200 万字符/月，QPS 100。
- 支持中/日/英等常见语种，满足 Pixiv 评论主要翻译方向。
- 百度另有“大模型文本翻译 API”，个人/企业认证用户有 100 万免费字符额度，并支持 Bearer API Key。
结论：
- 对普通个人用户，百度高级版的 100 万字符/月已经非常充足，且凭据是翻译平台自身的 AppID/密钥，比要求用户拿通用云账号主密钥更容易理解。

C. 阿里云机器翻译
- 通用版：100 万字符/月免费。
- 当前官方文档仍有通用版文本翻译接口和 50 QPS 配额。
结论：
- 与百度个人高级版的免费量相当，可以以后作为第二个传统 MT Provider，但首版没必要同时写两个几乎同用途的 Provider。

推荐 Func 首版翻译设置：
1. 关闭
2. 百度翻译（推荐，大陆）
3. LLM（OpenAI-compatible）
4. Google Translate（保留为兼容/海外可选项，不再默认）

“百度翻译”：
- 首版优先接通用文本翻译 API。
- 用户自己填写 AppID + 密钥。
- 凭据进入安全存储，不写普通 AppSettings。
- 目标语言默认跟随 Func 当前 UI 语言。
- 当前翻译功能只实际用于评论，所以设置名称建议改成“评论翻译”，不要假装已经是全局翻译系统。

“LLM 翻译”：
- 不针对 DeepSeek、GLM、Qwen、OpenAI 各写一套客户端。
- 首版只做一个通用 OpenAI-compatible Chat Completions 入口：
  - Base URL
  - API Key
  - Model
- API Key 安全存储。
- Base URL / Model 可以普通设置持久化。
- 固定一个小而明确的翻译提示词：
  - 翻译到当前 UI 语言；
  - 保留 URL、@用户名、emoji、换行；
  - 只返回译文，不解释。
- 不向普通用户暴露 temperature/top_p/reasoning 等模型参数。
- 未来如果遇到确实不兼容 OpenAI protocol 的热门 Provider，再根据真实需求加 adapter。

调研来源：
- 百度通用翻译：https://fanyi-api.baidu.com/product/11
- 百度通用翻译接入文档：https://fanyi-api.baidu.com/product/113
- 百度大模型文本翻译：https://fanyi-api.baidu.com/product/13
- 百度大模型 API 文档：https://fanyi-api.baidu.com/doc/21
- 阿里云机器翻译计费：https://help.aliyun.com/zh/machine-translation/product-overview/billing-overview
- 腾讯云 TMT API 更新历史：https://cloud.tencent.com/document/product/551/17231
- 腾讯云 TMT 计费：https://cloud.tencent.com/document/product/551/35017


D3 网络设置
------------
决定采用简化方案。

普通用户页只保留：
- 网络模式
  - 自动（推荐）
  - 仅直连
- 网络诊断 >
- 高级设置 >

Automatic：
- 新安装默认。
- 是 Func 的正常标准网络栈，不是“出问题后再开的兼容模式”。
- 用户不需要理解 ECH、DoH、SNI 等实现。
- 只有用户主动选择其它模式才覆盖 Automatic。

当前新发现：
- AppSettings 当前没有持久化 NetworkMode。
- networkAccessPolicyProvider 每次重建都默认 NetworkMode.automatic。
- 因此用户手动切成 Direct only 后，重启 App/重建 policy 可能又回 Automatic。
解决：
- 只持久化一个简单 networkMode = automatic/directOnly。
- 不持久化 route memory、探测结果等内部状态。

高级页：
- 可以给 power user 保留 DoH endpoint 和 ECH front host 编辑能力。
- 网络诊断尽量只读，展示当前 route / 最近探测结果。
- 删除普通用户可见的全局 insecureNoSni toggle。
- 删除已 descoped 的 native login WebView intercept 生产接线。
- 不在普通页面展示 “DoH / ECH / SNI / certificate verification / PlatformView” 教科书。

图片 CDN：
- 若之后 A/B 证实 PixEz 风格的 no-SNI / relaxed cert 对 i.pximg.net 明显更快、更稳，可作为 Automatic 内部 image-only route。
- 不允许扩展到 API/OAuth。


D4 图片质量
------------
决定：
- 预览质量：中图 / 大图 / 原图，默认“中图”。
- 查看质量：中图 / 大图 / 原图，默认“原图”。

旧设置迁移：
- previewQuality=true -> 大图
- previewQuality=false -> 中图
- scaleQuality=true -> 原图
- scaleQuality=false -> 大图

理由：
- 当前两个 bool 的语义不清。
- PixEz 默认信息流就是 medium，Func 当前默认 large；这是 PixEz 首屏/滚动体感更快的一个现实原因。
- 不需要为漫画/插画/放大查看分别再建四五套质量选项，先保持两组就够。


D5 下载保存位置
---------------
决定：
- 默认保存到 PixivFunc 相册。
- 用户可以改成自选目录或相册。

推荐 Android 设计：
A. 相册模式（默认）
- MediaStore。
- 默认 RELATIVE_PATH：Pictures/PixivFunc。
- 可以选择/输入其它相册目标。
- 对用户显示人类可读名称，例如“PixivFunc 相册”。

B. 文件夹模式
- 使用 Android 系统 Storage Access Framework 目录选择器。
- 持久化系统返回的 tree URI permission。
- 不自己写文件浏览器。

不要：
- 用一个普通字符串 TextField 让用户填磁盘路径；
- 自己管理 `/storage/emulated/0/...` 权限兼容表。


D6 文件命名
-----------
对比结果：

PixEz 当前：
- 高级模式实际上允许用户写 JavaScript `function eval(illust, index, mime) {...}` 来决定文件名。
- 功能极强，但对普通用户非常不友好。
- 不适合 Func 直接复制。

Shaft 当前：
- 做得更成熟：
  - 先给 4 个预设；
  - 每个预设直接展示“最终会存成什么”；
  - 高级用户再使用模板；
  - 支持 `{id}`、`{title}`、`{page}`、`{pages}`、`{ext}`、`{author}`、`{author_id}`、`{w}`、`{h}`、`{created}` 等；
  - 还能用条件表达式和 `/` 创建目录。
- Shaft 这套很强，但完整复制对 Func 仍然过度。

Func 推荐：
普通设置首先展示预设：
1. 作品 ID（默认）
   `123456789_p0.jpg`
2. 作者 - 标题 - ID
   `作者名 - 标题 - 123456789_p0.jpg`
3. 标题 - ID
   `标题 - 123456789_p0.jpg`
4. 自定义…

自定义模板只支持少量变量：
- `{artist}`
- `{title}`
- `{id}`
- `{page}`
- `{ext}`
- 可选 `{date}`

必须提供：
- 当前模板实时预览；
- 变量说明；
- 非法文件字符自动清理；
- 文件名过长自动裁剪；
- 多 P 页码明确。

不要：
- JavaScript eval；
- 正则表达式规则；
- Shaft 的 `[?R18:...]` 条件语言；
- 在“文件名模板”里再允许 `/` 控制目录。
目录由 D5 单独负责，避免把保存位置和文件名重新耦合。

调研来源：
- PixEz：lib/page/hello/setting/save_eval_page.dart
- Shaft：DOWNLOAD.md


D7 个人主页 Header
------------------
决定：
- 背景图随滚动 fade。
- 头像始终作为前景身份元素。
- 头像随滚动缩小并向 toolbar 移动。
- 用户名过渡成 toolbar title。
- 展开状态头像尺寸比现在缩小。

当前问题：
- 代码已经计算 avatarRadius 的 lerp，但真正 ExpandedProfile 没用它。
- 当前效果是 144dp 左右的大头像保持大小不变，整个 profile 一起变透明，最后突然换成小头像。
- 默认 “no image” 头像因此像巨大半透明背景水印。

建议：
- 展开头像直径约 96–112dp。
- 头像位置和半径连续插值。
- Header 背景与头像的透明度逻辑分开。


================================================
三、真实设备已经复现的用户体验问题
================================================

[U1 / P0] 下拉刷新手势错误
现象：
- 下拉呼出图标后反向上划，图标和列表一起上移。
- 预期应该先只收回刷新图标，页面不动；图标消失后剩余手势才滚动内容。
- 大力下甩、手已经离开后，惯性/overscroll 还能呼出刷新图标，甚至卡住。

代码原因：
- 当前是 RefreshIndicator.noSpinner + NotificationListener + 自绘 RefreshProgressIndicator。
- listener 监听反向滚动去收图标，却返回 false，所以同一滚动量继续交给列表。
- OverscrollNotification 可以在没有 active pointer 的情况下重新把 tracking 打开。

简要方案：
- 优先回到 Flutter 标准 RefreshIndicator 的触摸生命周期。
- 如果视觉必须自定义，只自定义 indicator，不自己重新发明完整 scroll lifecycle。
- 验收必须有两条真机/Widget test：
  1. 反向手势先只收 indicator；
  2. pointer up 后 ballistic overscroll 不得重新召唤 indicator。


[U2 / P1] 搜索热门标签没有代表图
现状：
- `/v1/trending-tags/illust` 已经返回 representative illust。
- Func 已解析并写入 IllustStore。
- UI 却只显示标签文字，代表作品主要只用于长按。

简要方案：
- 卡片显示 API 自带 representative image。
- 点击卡片：搜索该 tag。
- 长按/次要操作：进入代表作品（可保留）。
- 当天第一次进入搜索页拉一次；日期变化后下一次进入重新获取。
- 不为每个 tag 再搜索一次，不需要客户端随机算法。


[U3 / P1] 反向搜图 UI 存在但功能永远 unavailable
简要方案：
- 按 D1 接 SauceNAO 零配置路径。
- 首版不要继续扩 Provider abstraction。


[U4 / P1] 翻译设置与实际功能不一致
现状：
- 当前翻译只用于评论。
- Google Translate 是唯一实际 provider，且新设置默认选 Google。
- 大陆无代理用户无法可靠使用。
- UI 还写“翻译凭据由安全存储管理”，但当前 Google 路径不需要凭据。

简要方案：
- 设置改名“评论翻译”。
- 默认改成百度翻译（如果用户配置凭据）或关闭。
- 增加 generic LLM。
- Google 降为兼容选项。


[U5 / P1] 网络设置过于复杂
简要方案：按 D3。


[U6 / P1] 浏览质量只是两个开关
简要方案：按 D4。


[U7 / P1] 下载设置是假设置/表达差
现状：
- namingRule 可以保存，但真实 DownloadRequest 仍固定生成 `<id>_p<page>.<ext>`。
- 保存目录 ListTile 没有 onTap。
简要方案：按 D5/D6，接通真实消费；不要保留“能填但不生效”的 UI。


[U8 / P1] Profile 头像折叠视觉错误
简要方案：按 D7。


[U9 / P1] 作品详情作者区域不可点击
现状：
- 详情页已有 author ID、头像、用户名，并且代码已经能调用 showUserPage。
- 只是作者区域漏了 tap。

简要方案：
- 头像 + 作者名 + account 整个区域可点击 -> showUserPage(userId)。
- 不需要改路由架构。


================================================
四、当前 HEAD 仍存在的代码行为问题
================================================

[C1 / P1] credentialRevision 被当成过宽的全局“世界版本”
现状：
- AccountStore 的 credentialRevision 在这些动作都会增长：
  - token refresh 最终走 upsertAccount；
  - profile metadata update；
  - switch account；
  - reauth；
  - remove account。
- PagedFeedController 又把 `(accountId, credentialRevision)` 当 feed commit boundary。
- Download/Ugoira 也把 credentialRevision 放进 owner/recovery context。

真实后果：
- 同一个账号正常 refresh token 后，之前发起的合法 GET 响应可以被判 stale。
- 长下载/Ugoira 可能因为同账号 token 刷新被当成 owner change。
- profile metadata 变化也可能无意义地让读取任务失效。

简要方案：
- 不造 GlobalRevision2。
- Feed：稳定 accountId + feed generation + cancel/dispose。
- Mutation：account identity + operation identity。
- Download/Ugoira：stable accountId + job/output/destination。
- Credential refresh 自己处理并发，不要让 access token 版本成为全 App 所有权。


[C2 / P1] 收藏/关注/评论在 token 刚过期时第一次操作可被主动判失败
现状：
- PixivHttpClient 收到明确 401 / invalid_grant 后会成功 refresh。
- 但 Bookmark 等 repository 传 `allowAuthReplay:false`。
- refresh 成功后代码主动抛 `authentication refreshed; mutation replay suppressed`。

真实后果：
- 用户第一次点收藏失败，再点一次才成功。

简要方案：
- 明确 auth rejection（401 / 明确 invalid_grant）与“请求结果未知”分开。
- 明确认证拒绝 -> refresh -> 原操作只重发一次。
- timeout/reset 等不知道服务器是否已经执行 -> 不自动重放 mutation。
- 不需要幂等 token 服务、事务框架。


[C3 / P1] Entity Store 的“单调合并”会保留永久旧数据
IllustStore 当前例子：
- incoming caption 为空 -> 保留旧 caption；
- tags 为空 -> 保留旧 tags；
- visible=false 一旦观察到会 sticky；
- pageCount 不允许减少。

真实后果：
- 作者真的删除简介/标签、作品状态恢复、page count 合法变化时，Func 可能一直显示旧状态。

简要方案：
- 区分 payload 来源：
  - Feed/sparse：未提供的 rich 字段不覆盖。
  - Detail/authoritative：服务端返回的空值/false/减少也是真实新值。
- 一个简单 source 参数/两个 merge 方法就够，不要 AuthorityGraph。


[C4 / P1] Download recovery 与内存 credentialRevision 冲突
现状：
- Download recovery record 会持久化 credentialRevision。
- AccountStore 的 `_credentialRevision` 本身不持久化，进程重启从 0 开始。
真实后果：
- 真正发生进程死亡后，同账号合法下载记录可能因为 revision 不同被判 orphan。

简要方案：
- recovery owner 使用稳定 accountId + job/output + destination。
- 不要为了让现有设计成立去持久化 credentialRevision。


[C5 / P1] Download recovery 不是严格 startup recovery
现状：
- recovery 依赖 downloadManagerProvider 实际被创建。
- 用户 App 重启后如果只浏览首页，不保证马上执行下载恢复/MediaStore cleanup。

简要方案：
- App 启动后做一次轻量 recovery bootstrap。
- recovery 不自动重发下载；只恢复状态、清理明确拥有的 pending output，并让用户显式 retry。


[C6 / P1] Widget 无实例时仍可能在 App 启动支付网络成本
现状：
- App 侧 eager start WidgetCoordinator。
- Coordinator 在知道设备是否真的有 Widget 之前可能先拉 Recommended、下载封面。

简要方案：
- native `hasAnyWidget` gate。
- 没有 Widget -> 不启动任何 Widget feed/network work。


[C7 / P1/P2] Home 隐藏 Tab 可能冷启动一起拉数据
现状：
- Home 使用 IndexedStack。
- Ranking/New 等 child build 后会 watch 自己的 feed。

真实后果：
- 冷启动可能并发 Recommended + Ranking + New；再叠 Widget 更浪费。

简要方案：
- visited-tab lazy creation：
  - 首次只创建当前 tab；
  - 用户第一次访问后再纳入 IndexedStack 保状态。
- 不需要页面缓存框架。


[C8 / P1] User deep link 的消费链需要确认/补齐
此前同一 HEAD 审计发现：
- Pixiv user URL 可以解析成 UserRoute；
- Home consumer 只显式处理 IllustRoute。
简要方案：
- `UserRoute -> showUserPage(userId)`。
- 实施前对当前 HEAD 再做一次最小 recheck；如果本轮 Agent 已补，则直接划掉，不重构 router。


[C9 / P1] R18 / AI / blocked tags 设置有“能保存但不真正过滤”的风险
现状：
- `enableLocalBlockR18`、`enableLocalBlockAI` 设置和 UI 明确存在。
- BlockedTags 可以持久化/切换。
- 之前 repo-wide 审计没有看到 Feed 展示过滤 consumer。

简要方案：
- 一个小的内容可见性 predicate 放在共享 Feed presentation boundary：
  - blockR18 && work.isR18 -> hide
  - blockAI && work.isAI -> hide
  - blockedTags 与作品 tags 相交 -> hide
- 如果某开关暂时不准备实现，就删开关，不保留假设置。
- 实施前 recheck 最新 HEAD consumer，避免重复做已修功能。


[C10 / P1/P2] API Accept-Language 默认固定 zh-CN
现状：
- PixivHttpClient 构造参数默认 `languageTag='zh-CN'`。
- 当前 provider 没把 settings languageTag 注入。

真实后果：
- Func UI 设为日语/英语/俄语，Pixiv API 仍按中文请求。

简要方案：
- client provider watch settings.languageTag 并传给 PixivHttpClient。
- 不需要单独 LocaleService。


[C11 / P1] LoginPage 设置读取失败直接白屏
现状：
- loading -> 空 Scaffold；
- error -> 空 Scaffold。

简要方案：
- loading 正常 progress；
- error 显示错误 + retry。
- 这是应该有的真实错误处理，不是“防御性编程”。


[C12 / P2] Recommended base contract 不干净
现状：
- RecommendedFeedController 真正实现 `fetchPageForContext`。
- 为满足 base contract 的 `fetchPage()` 直接 `UnimplementedError('use fetchPageForContext')`。

简要方案：
- 改 base seam，让 context fetch 成为唯一必要 contract，或提供安全 adapter。
- 不要留一个“理论上永远不会调用”的 production throw。


[C13 / P2] 图片源设置只有一个选项
现状：
- ImageSourceMode 当前只有 normal / i.pximg.net。
- UI 仍像一个可选择的设置。

简要方案：
- 只有一个选项时直接隐藏这一设置。
- 未来真出现第二种产品级图片源再恢复。


[C14 / P2] Network Policy 还保留 legacy ladder
现状：
- production 已使用 preflight ladder。
- `_runLegacyLadder` 注释明确是 legacy/test caller compatibility。

简要方案：
- repo-wide 确认调用方；
- 能迁完就删。
- 不长期维护两套具有不同语义的网络策略。


[C15 / P1] NetworkMode 用户选择没有持久化
见 D3。
简要方案：
- AppSettings 增 `automatic/directOnly`。
- 默认 Automatic。
- 用户主动选择持久化。
- route choice/route memory 不持久化。


[C16 / P1/P2] native login WebView intercept 与已 descoped 产品边界冲突
现状：
- login_intercept_controller.dart / login_intercepted_webview.dart 仍在。
- LoginPage 会读取 `nativeWebViewInterceptProvider` 并传给 WebView。

简要方案：
- 首次登录仍走稳定普通 WebView。
- 删除生产设置和 wiring。
- 如果代码只为已取消目标存在，直接删除，不留“实验模式”。


[C17 / P1/P2] 全局 insecureNoSni 开关不应作为普通用户 fallback
现状：
- AppSettings 中存在 insecureNoSniEnabled。
- policy 会把 insecureNoSni 添加到 fallback tiers。

简要方案：
- 删除普通用户/全 host 级开关。
- 如果图片 CDN A/B 证明 relaxed path 有明确收益，做 image-only 内部策略，不扩大到 OAuth/API。


[C18 / P2] Reverse image 过度搭建外围但核心没接
简要方案：
- D1 先让 SauceNAO 真正工作。
- 工作后再删掉没有真实 provider 需求的多余 abstraction。


[C19 / P2] 翻译的 credential abstraction 曾经早于需求
现状：
- 已经有 SecretSettingRef / translationCredentialRef，但过去 Google 根本不需要 key。

现在 D2 已经产生真实凭据需求：
- 可以复用“安全存储”的思想；
- 但只为百度 AppID/Secret、LLM API key 建具体配置；
- 不建一个抽象 Secret Provider Registry。


[C20 / P2] 仍有生产 UI 硬编码中文
当前详情页仍可见例如：
- “投稿日期未知”
- “投稿日期：”
- “尺寸：”

简要方案：
- 进入现有 ReplicaStrings/i18n。
- 不额外换 i18n 框架。


[C21 / P2] Account 删除后的远端 History outbox
此前审计：
- History DB 的 account_id scoping 本身健康。
- 但 removeAccount 没明确清理“尚未同步到 Pixiv 的 history outbox”。
真实可能：
- 删除账号 A -> 未发送 outbox 留下；
- 以后重新加入同账号 A -> 旧 outbox 被补发。

简要方案：
- remove account 时删除该 account 的“未发送 remote outbox”。
- 本地 history 是否保留不要顺手全删；只有用户清历史时删。


[C22 / P2] Download/Ugoira orphan recovery 还有“状态存在但没有用户出口”
此前审计发现：
- unmatched MediaStore pending row 被故意保留；
- orphan report / tombstone 并没有清晰产品消费入口。

简要方案：
- 与 C4/C5 一起收口。
- 只清理能用 owner/job 明确证明是 Func 自己的 pending output。
- 不为了低概率 orphan 再造“Recovery Center”页面，除非真机证明用户需要。


================================================
五、网络：上一轮已修的问题，不要重新打开
================================================

以下是历史问题，当前 HEAD 已经有实质修复。后续任务必须把它们写成“不要按旧审计重做”，防止 Agent 再把旧架构复活。

[N-FIXED-1]
旧：业务 POST 先 direct 失败，因为不可 replay 就永远到不了 ECH。
现：production 先用 side-effect-free probe 选 route，再只发送一次业务请求。

[N-FIXED-2]
旧：把收藏/关注/OAuth POST 本身拿来发现网络路线。
现：生产 route selection 已与业务 mutation 分开。

[N-FIXED-3]
旧：rhttp 错误主要用字符串判断。
现：RhttpInvalidCertificate / Timeout / Cancel / Connection 等结构化异常已直接分类。

[N-FIXED-4]
旧：普通 handshake 被错误视为 certificate mismatch，直接终止 fallback。
现：只有结构化 invalid certificate 才是 certificateMismatch；handshake/reset 可进入下一安全路线。

[N-FIXED-5]
旧：成功 ECH route memory 被下一次成功自己删除。
现：成功请求会刷新/保留 memory。

[N-FIXED-6]
旧：一次操作可能重复尝试同一路由。
现：attempted kind/key 已阻止同一轮重复。

[N-FIXED-7]
旧：download client cache key 不含 canonical host。
现：当前 key 含 canonicalHost + route + purpose。

[N-FIXED-8]
旧：OAuth production wiring 可能绕过 network policy。
现：oauthServiceProvider 注入 pixivNetworkFactoryProvider 的 OAuth client。

[N-FIXED-9]
旧：rhttp 强制 HTTP/2 prior knowledge。
现：HttpVersionPref.all，让 ALPN 协商 HTTP/2/1.1。

结论：
- 不因为“更先进”重新引入 Cronet/QUIC/VPN。
- 当前网络工作重点已经是性能与真实设备验证，而不是再建设一个网络框架。


================================================
六、为什么 PixEz 无代理体感仍可能比 Func 快
================================================

这是“性能候选”，不是全部都已经证明的 bug。必须用真机 A/B/trace 决定是否改。

[P-NET-1 / 高概率]
默认图片体积不同。
- PixEz 信息流默认 medium。
- Func 当前默认 previewQuality=true -> large。
解决：
- D4 默认 medium。
这是最应该先做的无风险优化。

[P-NET-2 / 高概率]
Func 冷状态仍可能 direct-first。
当前没有 host/group memory 时 candidate 顺序里仍包含 direct，再进入 ECH/DoH。
代码注释自己记录大陆 direct 可能白白消耗数秒 connect budget。
解决方向：
- 对 Cloudflare API/OAuth 组做 A/B：
  Automatic 在大陆已知环境/已证明 group preference 下优先 ECH；
- 不要仅凭猜测硬编码国家判断。
- 可保留 outside-wall direct 速度优势。

[P-NET-3 / 值得 A/B]
当前生产所有 exit 都是：
route probe -> business request。
这对 POST 是合理的，因为不能让 mutation 探路；
但 GET/HEAD/image 本来可以安全重试，额外 probe 会多 RTT。
建议实验：
- GET/HEAD/image：业务 GET 本身作为 route attempt，transport failure 后换下一 route，最多重试一次。
- POST/PATCH/DELETE：继续独立 probe -> send once。
如果首屏/滚动有明显改善，再正式改。

[P-NET-4 / MEASURE ONLY]
第一屏多个 i.pximg.net 请求可能同时进入 route selection，形成选路惊群。
不要直接写 single-flight。
先 trace：
- 是否多张图真的同时重复 DoH/probe；
- 若存在，再加 per-host route-selection single-flight。

[P-NET-5]
PixEz 图片路径更激进：
- cached/fixed IP
- no SNI
- compatible path 可关闭 certificate verification
所以它天然比 Func 当前严格阶梯少很多选路成本。
Func 不应简单复制“所有 host 关闭证书”。
若真机 A/B 证明收益：
- 只对公开图片 CDN 内部采用；
- API/OAuth 继续 strict。


================================================
七、Release / Updater 当前确认的发布阻塞
================================================

[R1 / Release blocker] Release 仍用 debug signing
当前 android/app/build.gradle.kts：
- release signingConfig = debug。
解决：
- 正式 release keystore；
- GitHub/F-Droid 分发策略明确；
- update manifest 的 expected signer 必须对应正式 key。


[R2 / Release blocker / API29] updater manifest verifier 使用 Ed25519
当前 native：
- KeyFactory.getInstance("Ed25519")
- Signature.getInstance("Ed25519")
Android 官方 java.security.Signature 表：
- Ed25519：API 33+
- SHA256withECDSA：API 11+

Func minSdk=29。
因此 API29–32 上当前 verifier 没有平台 Ed25519 provider，catch 后返回 false。

简要方案：
- 最简单：manifest 签名改 `SHA256withECDSA` (P-256)。
- 更新生成签名脚本和 Android verifier。
- 当前代码假定 Ed25519 signature 恰好 64 bytes；ECDSA DER signature 长度可变，所以同时删掉 `signature.size == 64` 的协议假设，保留合理最大尺寸即可。
- 不为了保留 Ed25519 再引一个庞大 crypto provider。


[R3 / Release blocker] GitHub Release CDN redirect 被当前 guard 拒绝
当前 kUpdateDownloadHosts：
- github.com
- objects.githubusercontent.com
- github-releases.githubusercontent.com

但 2026 GitHub Release 实际常见链路：
github.com/.../releases/download/file.apk
-> 302
release-assets.githubusercontent.com/github-production-release-asset/<repo-id>/<opaque-uuid>?...

当前问题：
1. `release-assets.githubusercontent.com` 不在 allowlist。
2. 当前 `isStrictUpdateAssetUrl` 还要求每一跳 path 都 `.endsWith('.apk')`。
3. 真实 CDN path 是 opaque UUID，并不以 .apk 结尾。

简要方案：
- 初始 signed manifest 中的 asset URL 仍要求：
  - HTTPS
  - 指定 GitHub repo release URL
  - 预期 APK asset
- 重定向允许 GitHub 实际 release CDN host。
- CDN hop 不要求 path 带 `.apk`。
- 文件本体继续做：
  - signed manifest
  - exact size
  - SHA256
  - packageName
  - APK signer certificate
这些才是实际完整性/身份边界。

官方 GitHub 文档也明确 Release Asset 客户端需要处理 200 或 302。


================================================
八、其它产品/代码收口
================================================

[P1] 热门标签图片
- 见 U2。

[P1] 作者跳转
- 见 U9。

[P1] 用户页 header
- 见 D7/U8。

[P1] R18/AI/tag block
- 假设置要么接通要么删除。

[P1] 下载目的地/命名
- 按 D5/D6。

[P1] 浏览质量
- 按 D4。

[P1] 网络设置
- 按 D3，并持久化用户 networkMode。

[P2] 只剩一个图片源的“选择器”
- 隐藏。

[P2] 硬编码中文
- 进入现有 i18n。

[P2] legacy ladder
- 调用方迁完后删。

[P2] Trellis/spec
- 每次删除错误 guard 时同步删掉固化该错误行为的测试/spec。
- 不写“旧坏设计已经删除”这种永久实现细节到规范。
- spec 只保留真正产品 invariant：
  - account A 响应不能污染 account B；
  - mutation 在结果未知时不自动重复；
  - detail 可覆盖真实服务端更新；
  - API29 核心功能成立；
  - Automatic 是默认标准网络行为。


================================================
九、明确已经解决/不应再列为当前问题的旧项
================================================

这些旧审计项不要继续当 blocker：

1. `pixiv_image.dart` 错误 relative import
- 已修。

2. Account currentId 不在 accounts 时直接 fatal
- 当前 AccountStore 已有 `_validCurrentId`，无效 currentId 会返回 null。
- 不再为它建 transaction journal。

3. Profile Edit “完整编辑 UI + production 永远 unavailable”
- 当前 Settings 的 MePage 已明确 read-only。
- 不再按旧结论重复改。

4. 旧 route memory self-delete、rhttp classifier、OAuth bypass、h2-only、download host cache
- 见第五节，已修。


================================================
十、推荐任务拆分顺序
================================================

Task A：真实高频 UX correctness
--------------------------------
优先级最高。
- U1 下拉刷新；
- U9 作者区域跳作者；
- D7 Profile header；
- U2 trending tag 代表图；
- C11 Login settings error/retry；
- C20 硬编码中文顺手处理当前涉及页面。

原则：
- UI bug 不与 network/refactor 混做。
- 真机验收优先于漂亮的 unit test。


Task B：设置页产品化
--------------------
- D3 简化 network page；
- C15 持久化 Automatic/Direct only；
- 删除 native login intercept production wiring；
- 删除全局 insecure no-SNI 用户开关；
- D4 quality enum；
- D5 destination；
- D6 filename preset/template；
- C9 R18/AI/tag block 接通；
- C13 单一图片源设置隐藏；
- C10 API language 跟 UI language。

验收重点：
- 每一个普通设置必须真的改变产品行为；
- 没消费者的设置不能留。


Task C：真实功能补全
--------------------
C1. Reverse Image
- PixEz-style zero-config SauceNAO。
- 多来源。
- Pixiv URL 内部打开，其它 URL 外部打开。

C2. Comment Translation
- 百度翻译（推荐大陆）；
- Generic OpenAI-compatible LLM；
- Google 变成可选兼容 provider；
- 凭据安全存储；
- 目标语言跟随 UI。


Task D：行为正确性 cleanup
--------------------------
- C1 credentialRevision 降权；
- C2 auth rejection 后 mutation replay once；
- C3 source-aware entity merge；
- C4/C5 download recovery；
- C21 history outbox；
- C22 recovery orphan 收口；
- C12 base contract；
- C14 legacy ladder。

这是“删除错误边界”的任务，不是 architecture v2。


Task E：网络性能 A/B
--------------------
按顺序测：
1. preview default medium；
2. 冷启动 vs 热状态；
3. GET/image 去独立 preflight 的实验；
4. API ECH-first/route preference 实验；
5. trace 是否有 image route-selection 惊群；
6. 最后才讨论 image-only relaxed transport。

没有数据就不加新路线。


Task F：Release blockers
-----------------------
- 正式 signing；
- API29 updater signature；
- GitHub release-assets redirect；
- clean GitHub release 安装实测。


Task G：最终真机矩阵
--------------------
最低：
- API29 Android 10
- 当前主流/高版本 Android（项目 target 对应）
- 大陆移动网络 / Wi-Fi
- 无代理
- 有代理环境（Automatic 不应破坏用户已有网络）
- 首次登录（仅确认普通 OAuth 能完成，不要求 Func 绕墙）
- 已登录：
  - Recommended
  - Ranking
  - Search
  - User
  - Detail
  - token refresh
  - bookmark/follow/comment
  - image
  - download
  - Ugoira
  - history
  - widget（有/无实例）
- kill process -> restart -> download recovery
- GitHub updater on API29 and high API


================================================
十一、验收时重点观察的真实行为
================================================

1. 下拉刷新
- 慢慢拉；
- 拉出后反向；
- 快速甩；
- 松手后惯性；
- 连续第二次刷新。

2. token 到期
- 用户第一次点收藏就应该得到收藏结果；
- 不应该要求“再点一次”。
- 同时进行的 Feed/Download 不应该因为同账号 refresh 被莫名取消。

3. 网络
- Automatic 全新安装无需配置。
- 重启仍 Automatic。
- 用户手动 Direct only 后重启仍 Direct only。
- API/OAuth/image/download 都走同一产品策略，不要求用户理解 ECH/DoH。
- 网络诊断失败不能等于“真实业务一定失败”，但真实业务成功才是最终验收。

4. 下载
- 默认出现在 PixivFunc 相册。
- 自定义相册/SAF 目录真实生效。
- 文件名预设和自定义模板真实生效。
- 多页不会覆盖。
- kill app 后重启能看到可恢复状态。

5. 搜索
- 热门 tag 有图。
- 日期变化后更新。
- 不为每一个 tag 额外制造一批网络请求。

6. Reverse Image
- 多来源结果正常。
- Pixiv 命中回 Func。
- 外部来源不被强行解析成 Pixiv。
- SauceNAO 限流/失败有普通错误提示。

7. Translation
- 大陆无代理可调用百度。
- LLM Key/Base URL/Model 配置后可用。
- 失败不影响评论本身显示。
- API Key 不出现在普通设置导出/日志。


================================================
十二、明确禁止下一轮 Agent 为这些问题创造的东西
================================================

除非出现新的、具体的协议/平台需求，否则不要新增：

- GlobalWorldRevision / IdentityEpochManager
- OwnershipPolicyEngine
- EntityAuthorityGraph
- MutationTransactionManager
- RefreshGestureEpoch / BallisticLease
- NetworkFallbackStateMachine v2
- AccountTransactionJournal
- ReverseImageProviderRegistry（为还不存在的多个 provider）
- TranslationProviderFramework（首版只需百度 + generic LLM + legacy Google）
- 自制 Android 文件浏览器
- JavaScript 文件命名脚本
- Cronet/QUIC/VPN
- “为了防止未来可能发生”而新增的 allowlist/deny guard

每个新 guard 在写入之前必须回答：
“它对应的真实协议、信任边界、平台要求或已复现 bug 是什么？”

如果回答只是“以后可能”，就不要写。


================================================
十三、调研资料摘要
================================================

国内翻译（2026-09-01）
----------------------
百度：
- 通用文本翻译：个人高级版 100 万字符/月；企业尊享版 200 万/月。
- 大模型文本翻译：认证用户 100 万免费字符。
- 日语、中文均支持。
- 当前 API 文档存在且可调用。

阿里云：
- 通用机器翻译 100 万字符/月免费。
- 50 QPS，单次通用文本 <5000 字符。

腾讯云：
- 计费页仍显示“文本翻译 500 万字符/月免费”。
- 但 API 更新历史 2026-07-08 明确“删除接口 TextTranslate”。
- 因此不选作当前 Func 文本翻译主 Provider。

结论：
- 首版大陆传统 MT：百度翻译。
- 备用：阿里云以后按真实用户需求再加。
- LLM：generic OpenAI-compatible，不绑定厂商。


SauceNAO
--------
- 官方索引远不止 Pixiv，天然满足多来源。
- 零配置可以使用 Web search HTML 入口。
- PixEz 当前已验证：
  - 本地压缩；
  - 最大边附近约 720px；
  - JPEG quality 75；
  - POST /search.php；
  - WebView 显示 HTML；
  - Pixiv 链接回原生。
- 这比 Func 当前“完整 provider abstraction + 永远 unavailable”更符合 1.0。


文件命名
--------
PixEz：
- 允许 JavaScript eval 控制文件名。
- 功能极强但学习成本高。

Shaft：
- 预设 + 模板 + 预览。
- 模板甚至支持条件、目录、日期等复杂语法。

Func：
- 取 Shaft 的“预设优先、模板可选、实时预览”。
- 不取 PixEz JavaScript。
- 不取 Shaft 条件语法/目录语法。
- D5 管保存位置，D6 只管文件名。


Updater
-------
Android 官方：
- Ed25519 Signature：API 33+
- SHA256withECDSA：API 11+
因此 Func API29 基线不能依赖平台 Ed25519 provider。

GitHub 官方 Release Asset API：
- 客户端需要处理 200 或 302。
2026 实际 release download：
- 常跳 `release-assets.githubusercontent.com/.../<opaque uuid>`.
因此 current host/path guard 与真实 GitHub release chain 不兼容。


================================================
十四、最终方向一句话
================================================

Func 下一阶段不应该再“建设更完整的框架”。

应该做的是：

把已经存在的核心能力真正接通，
删除阻塞 happy path 的错误 guard，
把普通设置变成用户能理解且真的生效的产品设置，
让 Automatic 网络在真实大陆环境里默认又快又能用，
然后用 API29 + 当前主流 Android 的真机长期使用去发现剩下的问题。
