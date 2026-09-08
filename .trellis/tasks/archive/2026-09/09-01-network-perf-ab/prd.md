# 网络性能 A/B 实测

## Goal

用实测数据决定网络路线，而不是靠对标推测。当前网络栈已有 5 档路线与完整的**可达性**
诊断，但**没有任何性能测量**——「哪档更快」这个问题目前无法回答，因此也就无法有依据地
调整默认路线或阶梯顺序。

覆盖审计编号：P-NET-1 ~ P-NET-5。

审计原文已由用户补回并归档到
`../09-01-func-1-0-hardening/research/audit-source.md`。原文基线是 `9d1cb1b`；实现前仍
需对当前 HEAD 做最小复核，不能把旧基线行号直接当作当前证据。下文 R1–R5 是方法论要求，
R6 收录五个具体实验项。`task.py start` 前必须完成当前 HEAD 复核、settings D4 交付和
本文件的 design/implement review。

## 现状核实（HEAD `409df51`）

网络层规模：`lib/core/network/` 共 4436 行，其中 `compat/` 子层 3873 行。

**已有 5 档路线**（`compat/network_contracts.dart:106`）：

| 档 | 组成 | 说明 |
|---|---|---|
| `direct` | 系统 DNS + 真实 SNI + 完整校验 | 基线 |
| `ech` | DoH 地址 + ECH（真实 SNI 加密）+ 完整校验 | |
| `dohRealSni` | DoH 地址 + 真实 SNI + 完整校验 | |
| `noSni` | DoH/兜底地址 + 空 SNI + 完整校验 | 仅源站主机 |
| `insecureNoSni` | 同上但**关闭证书校验** | 仅用户显式开启，从不自动 |

阶梯顺序是 **per-destination-group** 而非全局（API 主机在 Cloudflare anycast，
图片主机在源站）。

**已有可达性诊断**（`compat/network_probe.dart`, 584 行 + `features/settings/network_probe_page.dart`）：

- 分层探测：`system-dns` / `doh` / `tcp` / `tls` / `http` / `ech` / `no-sni`。
- 8 种结论：`dnsPolluted` / `ipBlackholed` / `sniBlocked` / `echAvailable` /
  `noSniAvailable` / `appLayer` / `allReachable` / `inconclusive`。
- 报告可复制（`NetworkProbeReport.toCopyableText()`）。

**缺口**：以上全部回答的是「**通不通**」。没有任何代码测量「**多快**」——
无延迟分布、无吞吐、无握手耗时、无首字节时间、无失败率统计。因此：

- 阶梯顺序目前是**设计时按可达性排的**，没有性能依据。
- 「Automatic 会不会比直连慢」这个问题无法回答。
- parent R2 允许的「图片 CDN 内部放松传输策略」需要 A/B 证明才能启用——
  **而做这个证明的工具还不存在**。

## Requirements（方法论部分，已可执行）

### R1. 先建测量能力，再谈路线调整

- 本 child 的**第一交付物是测量能力**，不是路线改动。
- 在拿到数据之前不调整任何默认路线、阶梯顺序或超时参数。
- 若测量结论是「现状已是最优」，本 child 的正确结局就是**不改代码**并留下数据。
  这不算失败。

### R2. 测量什么

至少覆盖能区分 5 档路线优劣的维度：

- DNS 解析耗时（系统 DNS vs DoH）
- TCP 建连耗时
- TLS 握手耗时（各档分别）
- 首字节时间（TTFB）
- 失败率与失败类型分布（复用已有的 `NetworkFailureKind` 分类）

按 **destination group** 分别统计（API 主机与图片主机的最优档很可能不同——
现有阶梯就是按这个前提设计的）。

### R3. A/B 的可信度要求

- 同一网络环境下对比，不能拿不同时间、不同网络的数据互比。
- 样本量与离群值处理方式必须写明；单次请求的耗时差异不构成结论。
- 报告可导出、可复制（沿用 `NetworkProbeReport` 已确立的模式）。
- **测量代码不得改变被测路径的行为**——不能为了测量而插入额外的请求或缓存旁路。

### R4. 硬边界（parent R2 / R5.2）

- **不引入 Cronet / QUIC / HTTP-3 / VPNService / 系统代理修改 / Private DNS 修改 /
  hosts / root 依赖 / 自建远端 relay**。「更像 PixEz」不构成引入理由。
- API / OAuth / 账号接口**必须保持真实服务器身份验证**，不做 MITM、不装自定义 CA。
- 放松传输策略若被证明有价值，**只允许限制在公开图片 CDN**（`i.pximg.net` /
  `s.pximg.net`）内部使用，**不得暴露为全局普通用户开关**
  （settings 的 C17 正在删除那个开关，本 child 不得把它加回来）。
- 审计第五节的 9 项网络历史问题（N-FIXED-1 ~ N-FIXED-9）在 HEAD 已实质修复，
  **本 child 不得把它们当作待修项重新打开**。

### R5. 测量结果的处置

- 数据落到本 child 的 `research/`，包含测量环境（运营商、网络类型、时间、设备）。
- 每一条基于数据的路线改动，都要在提交信息或 design 中指向具体数据。
- 数据不支持的改动一律不做——这是本 child 存在的全部意义。

## R6. 审计原文补全：P-NET-1 ~ P-NET-5

这五项都是性能候选或测量要求，不自动等价于已证实的 bug。实现前对当前 HEAD 复核后，按
下列顺序执行；每项必须留下可复现数据，数据不足时结论只能是“继续观测”，不能改路线。

### R6.1 P-NET-1：默认图片体积与质量档位

验证 PixEz 信息流 medium 与 Func 当前 preview 默认 large 的体积、TTFB、首屏和滚动影响。
D4 的 typed quality 与默认值由 settings child 实现；本 child 只复测实际请求与渲染效果，
不重复设计质量设置。

### R6.2 P-NET-2：冷状态 direct-first 与 ECH-first A/B

在同一真实网络环境分别测量无 host/group memory 的冷启动，以及热状态下 API/OAuth 组的
direct-first 与 ECH-first。不得按国家硬编码；至少保留大陆已知环境与外部网络环境两个样本。

### R6.3 P-NET-3：安全 GET/HEAD/image 是否可跳过独立 preflight

只对 GET/HEAD/公开图片做实验：将业务请求本身作为 route attempt，transport failure 后最多
换路一次。POST/PATCH/DELETE 继续 preflight 后只发送一次；不得把 mutation 纳入重试实验。

### R6.4 P-NET-4：图片 route selection 惊群

在第一屏多张 `i.pximg.net` 请求并发时 trace DoH、TCP/TLS 和 route selection 是否重复。只有
确认存在重复成本后，才评估 per-host single-flight；未证实前不新增并发协调层。

### R6.5 P-NET-5：公开图片 CDN relaxed transport 候选

比较 strict 与 PixEz-style cached/fixed IP、no-SNI 或 relaxed certificate path 的图片 CDN
效果。即使有收益，也只能在 `i.pximg.net` / `s.pximg.net` 内部使用；API/OAuth 永远保持真实
身份验证和 strict transport。不得把候选实现成全局普通用户开关。

## Acceptance Criteria（方法论部分）

- [x] 存在可重复运行的测量能力，输出按路线档与 destination group 分组的性能数据。（设置 → 网络探测：每 host 每层 `NetworkProbeStep.duration` + `totalDuration`；`network_probe_test` 18 项）
- [x] 测量报告可导出/复制，含测量环境元信息。（`toCopyableText` 头部 `env / app-version / os / network-mode / doh / ech-front`，`5892a11`）
- [x] 至少在一种真实大陆网络环境下完成一轮完整测量，数据存入 `research/`。（用户真机 5G/4G 无 VPN 两轮：`research/real-device-probe-2026-09-03.md`）
- [~] 每一条路线改动都能指向支撑它的数据；没有数据支撑的改动为零。（**部分被 2026-09-03 用户决策取代**：attempt-first / 兼容档纳入 API-OAuth 按 PixEz 方案直接落地、无需数据；**最终档序**（ECH 优先、兼容档最后）则由 09-03/04 真机探测数据决定，见 research）
- [x] 未引入 Cronet / QUIC / VPN / 代理修改 / 自建 relay。（仅 `HttpVersionPref.all`，HTTP/3 未开）
- [~] API / OAuth 的证书校验未被放松。（**被 2026-09-03 用户决策取代**，决策记录 `../09-01-func-1-0-hardening/research/decision-record-2026-09-02.md`「网络性能修订」：PixEz 兼容档（空 SNI、不校验证书、固定地址）作为所有已知 Pixiv 主机的**最后**兜底，含 API/OAuth；ECH / dohRealSni / direct 仍完整校验；OAuth token 只发往 `https://oauth.secure.pixiv.net/auth/token`，地址表限白名单主机，URL/Host 不改写）
- [x] 未新增全局的「不安全传输」用户开关。（生产恒开、无设置项与 i18n key，`i18n_network_keys_test`）
- [x] N-FIXED-1 ~ N-FIXED-9 在最终 HEAD 上仍然成立。（2026-09-08 check 逐项对照 `restricted_compat_network_test` / `rhttp_client_factory_test`；N-FIXED-1「先探测再单发 POST」以 attempt-first + 仅未送达失败换档的新形态成立）
- [x] `flutter analyze` 与 `flutter test` 通过。（2026-09-08：651+ 通过，仅 WSL loopback 已知超时；网络相关文件单跑全绿）

## Open Questions

- **R6 的五项当前 HEAD 复核**：原文已补回，但旧基线结论需在实现前重新确认。
- 测量能力已确认扩展现有网络诊断页的高级/深藏区域；沿用其报告导出模式。
- 测量结果不在普通用户页面展示；只有有数据支撑的改动才进入生产路径。

## Notes

- 上级需求与跨 child 约束见 `../09-01-func-1-0-hardening/prd.md`（R2 产品边界、
  R5.2 网络禁令、R6 禁止新增清单）。
- 本 child 排在最后：它的结论可能影响图片加载路径，而那条路径同时被
  settings（D4 质量档位）与 download 消费，先让那些定稿。
