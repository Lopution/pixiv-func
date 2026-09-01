# 技术设计：无代理网络与作品转场

## 1. 边界与不变量

本任务只改动现有 `NetworkAccessPolicy`/`DohResolver`/探测报告、作品卡片与详情首图、共享下拉刷新包装器以及对应测试。现有目的地主机注册表、严格证书校验、`NetworkRevision` 失效边界、图片 Referer/缓存管理器和 `ReplicaPageRoute` 的右侧滑入曲线保持不变。

网络出口仍然只有：

```text
业务请求
  └─ NetworkAccessPolicy.runLadder
       ├─ clean network: direct (首次测量)
       ├─ Cloudflare hosts: ECH → DoH + real SNI
       └─ image hosts: DoH + real SNI → no-SNI → explicit insecureNoSni
```

ECH 成功只改变当前 revision 内的候选顺序，不把失败、未经验证的 IP 或第三方域名写入持久化设置。

## 2. 网络路径与缓存

### 2.1 ECH HTTPS-RR 缓存与并发合并

在 `DohResolver` 内增加运行时缓存：

- key = `frontHost + revision.value + revision.networkIdentity`；resolver 实例本身已随 DoH/ECH 设置变化而重建，因此不再额外持久化设置指纹。
- value = 不可变的 ECH config bytes、front `ipv4hint`、解析后的 TTL 和写入时间。有效期取 RR TTL（沿用现有 5 秒～10 分钟 clamp）。
- 过期或 revision 不同立即丢弃；`dispose` 清除缓存和未完成查询。
- 对无取消信号的并发查询复用同一个 in-flight `Future`，使推荐/探测同时访问四个主机时只发一次 HTTPS-RR 查询。带取消信号的调用保留各自可取消语义，不让一个已取消请求污染共享结果。
- 返回缓存副本时继续执行现有字节范围、公共地址和类型校验；不把空 config 当作成功。

### 2.2 按目的地组记忆可用档位

在 `NetworkAccessPolicy` 现有按主机的 `_routeMemory` 之外增加轻量的运行时“组偏好”：

- Cloudflare 组覆盖 `app-api`、OAuth、accounts 和 Pixiv Web；image 组覆盖 `i.pximg.net`/`s.pximg.net`。
- 默认没有偏好时仍先探测 `direct`，保证干净网络不承担 ECH 开销。
- 成功业务请求或独立 probe 记录 `ech`/`dohRealSni`/`noSni` 后，在其 route TTL（缺省使用现有 10 分钟）内把该档位放在 direct 之前；构造 route 时仍按目标主机解析地址，不能跨主机复用目标 IP。
- 组偏好仅用于候选排序。探测失败会立即移除偏好并走现有下一档；一次操作最多仍允许一轮幂等重选。
- `setMode`、`advanceNetworkRevision`、resolver 重建和过期都会清除组偏好，与客户端池和按主机记忆同步。

这样在截图所示网络上，首个 Cloudflare 请求完成 ECH 测量后，后续 API/OAuth 请求不再先等待必败的直连，也不会重复读取同一份 ECH config；在普通网络上直连成功仍是最快路径。

### 2.3 探测结论与可观察性

`NetworkProbeReport` 增加 `dnsDisagrees`（或等价的附加诊断字段），但不新增大量结论枚举。分类顺序调整为：

1. `ech` 层成功（HTTP 404/403 也算传输到达）→ `echAvailable`；
2. `no-sni` 层成功且非 421 → `noSniAvailable`；
3. 两个 DNS 源无共同公共地址 → `dnsPolluted`；
4. 其余沿用 TCP 黑洞、SNI 被封、应用层和不确定结论。

`NetworkProbe` 当前只把 HTTP 421 视为空 SNI 的路由错误；ECH/真实 SNI 的任意 HTTP 状态仍记录为“已到达”，不把 404/403 当作传输失败。探测页在 badge 下增加简短的 DNS 差异说明，复制文本写入 `dns-disagrees: true/false`，并保留每层原始行。这样“直连失败但 ECH 可用”不会再被橙色 DNS badge 误导，同时污染证据仍可诊断。

## 3. 作品首图、头像与 Hero

### 3.1 统一首图输入

给 `IllustEntity`/作品卡片提供一个明确的首页预览选择器，质量参数与现有 `previewQualityProvider` 对齐：

- 来源卡片计算 `previewUrl` 后，将该 URL 作为 `heroImageUrl` 传给 `IllustDetailPage`；
- 详情 index 0 优先使用 `heroImageUrl`，没有来源（深链/历史）时使用实体的高质量首图（不再因 `pageCount > 1` 无 `metaPages` 回退到 medium）；
- 详情 index > 0 继续按 `metaPages` 的逐页 URL/尺寸回退规则渲染。

因此来源和目标对同一首图、同一质量命中同一缓存键；详情 API 后到达时可以补全其余页，但不能把首个 Hero 替换为灰色占位或更低清 URL。

### 3.2 立即导航与非阻塞图片预热

在共享 `PixivImage` 边界提供使用同一 `CacheManager`、headers 和 URL 的 `ImageProvider`/预热辅助函数。`IllustCard` 的 `onTapDown`（无该事件的入口则在 `onTap`）开始预热，点击手势确认后：

1. 读取当前首图 URL和作者头像 URL；
2. 立即启动首图/头像 `precacheImage`，但不 `await` 其 Future；
3. 立即 push `ReplicaPageRoute`，把 `initialEntity`、`heroScope` 和 `heroImageUrl` 原样传入；
4. 详情首帧直接创建标题、画师名、头像 provider 和首图 Hero。命中共享缓存时真实像素同步可见，未命中时头像位置由现有 placeholder 占位，provider 完成后自然更新。

预热只影响当前点击，不建立全局队列，也不改变点击响应时间。预热 Future 的错误被记录为图片自身的失败，不转成导航错误；详情仍使用现有 placeholder/errorWidget。该方案能保证首帧 provider、占位尺寸和文本同步，不能保证冷缓存的远端像素在网络响应前出现。若产品坚持“冷缓存第一帧必须是真实头像”，唯一可行的替代是列表阶段提前预取所有可见作者头像或等待网络，需另行评估网络/内存或点击延迟成本。

### 3.3 稳定作用域与飞行裁剪

抽取纯函数/约定生成作用域：

| 来源 | scope 示例 |
|---|---|
| 推荐独立页 | `recommended:illust` |
| 推荐存活 tab | `recommended:<contentType>` |
| 新作 | `new:<scope>:<type>` |
| 排行 | `ranking:<mode>` |
| 搜索插画 | `search:<query.cacheKey>` |
| 个人页 | 保留现有用户 + 筛选器 scope |

卡片和详情共享同一 scope；不同存活列表即使作品 ID 相同也不会产生同一 Hero tag。非卡片入口继续使用默认 scope，找不到来源时由路由自然滑入。

来源 Hero 已在内部 `ClipRRect`。详情 Hero 使用相同圆角的
`flightShuttleBuilder`，在 Navigator overlay 中将 shuttle 裁剪到源/目标
垂直 viewport 的全局交集；嵌套 sliver 使用其 approximate paint clip，因而
不会在返回时先越过底部导航或 pinned header 再被父布局截断。最终详情布局
仍可保持原有图片形状。

## 4. 反向下拉刷新

新增小型 `PullToRefresh`（名称可按现有命名调整），封装标准 `RefreshIndicator`，不复制 Flutter framework 源码：

- 对外只保留现有调用点需要的 `onRefresh` 和 `child`；
- 外层 `NotificationListener<ScrollNotification>` 记录 leading-edge 下拉和随后反向的滚动更新；视觉位移按指针增量线性回放，即使 Flutter 内部仍处于 armed 状态也能回拉到不可见区域；
- 释放前实际距离低于阈值时只取消视觉 indicator，不调用 `onRefresh`；
- `onRefresh` 开始后设置 `_refreshing`，此时忽略滚动重置，保证服务器请求和现有 feed 状态不被取消；完成/失败后恢复可再次下拉；
- 迁移全部现有 `RefreshIndicator` 使用点，保留每个页面已有的 load-more NotificationListener（共享组件只处理刷新方向并返回 false）。

## 5. 兼容性、回滚与风险

- 新字段均可选或由现有构造器默认值填充，深链/历史/测试入口不需要一次性改完即可编译。
- ECH 缓存是内存态，进程重启后重新测量；网络 revision 和设置 provider 的重建是强失效点。
- 非阻塞预热不会延迟点击，但未命中缓存时头像会在转场过程中补齐；若设备回归要求第一帧必须有真实头像，需要另开范围评估列表阶段预取的网络/内存成本，不在本任务中偷偷改成等待导航。
- 所有改动可按网络、刷新、Hero 三组独立回退；不涉及数据库、登录凭据或持久化迁移。
