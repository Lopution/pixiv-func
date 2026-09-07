# 执行计划：网络性能 A/B 与图片路径候选验证

## 开工条件

- [x] P-NET-1~5 与 N-FIXED-1~9 当前 HEAD 复核摘要写入本任务 `research/current-head-recheck.md`。
- [x] D4（PreviewQuality/ViewQuality + medium/original 默认）与 D3（networkMode 简化）已由
      settings-productization 交付；本 task 未改设置模型。
- [x] 实验仅使用现有 network contract（无 Cronet/QUIC/VPN/代理/hosts/root/relay）。
- [x] “无数据不改路线”规则已写入本 implement；采样/报告字段见阶段 1 实现。

- [x] 写入 `research/current-head-recheck.md`（P-NET 入口 + N-FIXED 回归证据）。
- [x] destination group（appApi/oauth/图片）由 probe 的 host+purpose 天然分组；
      route kind/key 由 network_contracts 提供；冷/热待真机采集（本机不可测）。
- [x] 统计字段：per-layer duration + total（probe 输出）；脱敏（只含 host/purpose/status）。

## 阶段 1：报告与离线测量能力

- [x] `NetworkProbeStep.duration`（per-layer）+ `NetworkProbeReport.totalDuration` +
      可复制文本追加 `(Nms)`；可达性结论与分类逻辑未变。
- [x] fake fixtures 已存在（HTTP 全部 I/O 注入）；新增断言：duration>=0、total 导出、
      探测不增加请求（沿用注入 harness）。
- [x] 阶段门：network_probe_test 19 项、restricted_compat_network 全过；N-FIXED 行为测试
      未变。

## 阶段 2/3：按 PixEz 方案直接落地（2026-09-03 用户决策，无需真机数据）

- [x] P-NET-1：D4 natural（settings-productization 已交付）。
- [x] P-NET-2：PixEz 兼容档为所有已知主机的第一候选，无需 cold 探测与 DNS；
      失败地址冷却 30s 后进入下一档。
- [x] P-NET-3：业务请求即 attempt（attempt-first），取消独立 probe；GET/HEAD 超时也可换档，
      POST/一次性 exchange 仅在未送达类失败时换档一次（DNS/connect/reset/TLS 握手）。
- [x] P-NET-4：DoH 解析与 ECH config 查询已有 single-flight；图片请求共享 per-host
      route memory，无额外协调层（PixEz 同样无）。
- [x] P-NET-5：PixEz compatible tier（空 SNI + 无证书校验 + 固定地址）生产恒开启，
      覆盖 API/OAuth/image/download 全部出口（用户明确撤销「仅图片/严格 API」限制）。
- [x] ECH 增强：ECH config 从阿里 DoH（dns.alidns.com → 223.5.5.5）查询，境内可达，
      与 PixEz `lookup_alidns_https_ech` 同源。
- [x] 回归：`flutter analyze` 0 issues；全量 `flutter test` 592 项通过（含
      attempt-first/逐档重试/POST 未送达重试等新契约用例）。
- 决策记录：`../09-01-func-1-0-hardening/research/decision-record-2026-09-02.md`
  「网络性能修订（2026-09-03）」；spec 契约：`.trellis/spec/frontend/state-management.md`
  §Restricted Pixiv Network Policy Contract 已同步。

## 最终验证

- [x] `flutter analyze`（No issues）
- [x] `flutter test`（全量 593 项通过，含 attempt-first/逐档重试/POST 未送达重试新契约）
- [x] `git diff --check`（提交前统一执行）
- [x] 真实大陆无 VPN 探测报告已回填：`research/real-device-probe-2026-09-03.md`
      （两轮用户实机分层数据 + 结论 + 体感确认）。
- [x] 用户真机观察：Automatic 默认路径（ECH first）首屏 ≤1.5s、图片成批、滚动跟手；
      图片/API/OAuth/下载全部走同一策略。

## 回滚点

1. 报告/timing 扩展；2. 读请求实验开关；3. route preference；4. image-only relaxed path。
每一层只在数据支持时落地，任何 API/OAuth 证书放松、请求重复或代理环境回归都回滚该层，
不恢复历史 N-FIXED 问题。
