# 技术设计：网络性能 A/B 与图片路径候选验证

设计基线：当前规划基线 `409df51`；用户审计原文基线 `9d1cb1b`。五项实验的原始定义与
来源见上级 `research/audit-source.md`。本 child 的第一交付物是可重复测量和脱敏报告，
不是重建网络栈或预先改变默认路线。

## 1. 依赖与不可变边界

- `settings-productization` 先定稿 D4（preview medium / view original）和 D3 的 network
  mode；本 child 不重复实现设置模型。
- 当前 HEAD 先复核 N-FIXED-1~9 与五个 P-NET 项；历史已修复项不得因实验回退。
- 不引入 Cronet、QUIC/HTTP-3、VPNService、系统代理、Private DNS、hosts、root 或远端 relay。
- API/OAuth/账号接口始终真实 TLS 身份验证；任何 relaxed/no-SNI 候选只能在公开图片 CDN
  内部评估，不能变成全局普通用户开关。

## 2. 测量合同

沿用 `NetworkProbeReport` 的分层报告与复制模式，扩展结构化 timing，而不是另建网络框架。
每条样本至少包含：destination group（API/OAuth 或 image CDN）、route kind/key、DNS、TCP、
TLS、TTFB、总耗时、HTTP 结果、`NetworkFailureKind`（如有）和测量环境元数据。元数据包括
设备/API level、网络类型/运营商、代理状态、时间、应用版本、冷/热状态；不保存账号 token、
请求正文或图片。

建议的最小实验协议（实现前写入 research 并按真实条件调整）：每个比较臂在同一设备、同一
网络窗口内完成不少于 10 次冷/热样本，报告 median、p95、成功率和失败类型分布；丢弃样本的
理由必须记录，不能只挑最快结果。单次请求不形成结论。

测量不得添加额外业务请求或缓存旁路；如果必须探测，使用已有 side-effect-free probe，
并把 probe 成本单独计入报告。POST/PATCH/DELETE 的生产语义保持 preflight -> send once。

## 3. 五项实验设计

### 3.1 P-NET-1：图片质量与体积

在 D4 定稿后，对 medium/large/original 的真实图片请求记录字节数、TTFB、首屏完成和滚动
丢帧/卡顿观察。该项验证设置改动的收益，不据此新增 route。

### 3.2 P-NET-2：冷状态 route preference

对 API/OAuth destination group 比较无 memory 的 direct-first 与 ECH-first；分别记录大陆
已知网络与外部网络样本，避免按国家硬编码。热状态单独记录 route memory 命中后的成本，
不把冷状态结论直接推广到所有网络。

### 3.3 P-NET-3：安全读请求是否跳过 preflight

为 GET/HEAD/公开图片建立实验开关或测试注入，使业务请求本身成为 route attempt，transport
failure 后最多换下一 route 一次；对 mutation 保持现有 preflight。比较额外 RTT、成功率和
重复请求计数，未获数据支持时不进入 production。

### 3.4 P-NET-4：图片 route selection 惊群

在第一屏并发多个 `i.pximg.net` 请求时采集 DoH/TCP/TLS/probe 时间线与 host/route key，判断
是否重复执行同一选择。如果 trace 无重复成本，不增加 single-flight；若确认存在，再设计
per-host single-flight，并证明取消、失败和不同 destination 不会互相阻塞。

### 3.5 P-NET-5：图片 CDN relaxed 候选

只在公开 `i.pximg.net` / `s.pximg.net` 比较 strict 与 cached/fixed IP、no-SNI 或 relaxed
certificate path。报告兼容性、速度和证书风险；API/OAuth 仍 strict。没有显著且可重复收益
时保留现状，不能因“更像 PixEz”强行加入。

## 4. 文件责任

| 责任 | 主要文件 |
|---|---|
| timing/report | `lib/core/network/compat/network_probe.dart`、`network_probe_page.dart` |
| route 实验注入 | `lib/core/network/compat/` 现有 route/policy contract；不新增 v2 状态机 |
| 质量与图片观察 | settings D4 消费侧、image request/cache 测试 |
| 报告 | `research/measurement-*.md`（环境、原始汇总、结论与数据到改动的映射） |
| 回归 | network probe、route selection、GET/mutation retry、image concurrency 测试 |

## 5. 风险与回滚

- 先合并测量能力与离线 fixture，再运行真实设备实验；没有真实数据不改默认路线/超时。
- GET 实验若改变重试次数或缓存命中，单独回滚实验开关；mutation 路径不随实验改动。
- single-flight 只有在 trace 证明惊群后才允许进入设计 review；否则以研究结论关闭 P-NET-4。
- relaxed 图片路径若出现证书、跨 host 或 API 泄漏，立即回滚该阶段，保持 strict 网络栈。
