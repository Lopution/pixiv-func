# 当前 HEAD 复核（network-perf-ab，2026-09-03）

基线 `9d1cb1b`；以下为实施时定位。

## P-NET-1（默认图片体积/质量档）
- D4 已由 settings-productization 交付：`PreviewQuality`（默认 medium）、
  `ViewQuality`（默认 original）；`illust_entity.previewUrl/viewerUrls` 消费新档位，
  `recommended_illust_page` 与 `illust_detail_page` 已迁移。本 task 不重复设计。

## P-NET-2（冷 direct-first vs ECH-first）
- 入口：`network_access_policy_provider`（network_providers.dart）→ `NetworkAccessPolicy`；
  路线内存 `_routeMemory`/`_groupMemory`（network_policy.dart）+ 网络诊断页
  `network_probe_page.dart`（policy 的 runExplicit）。测量点现成；冷/热对比需真机。

## P-NET-3（GET/HEAD/image 跳过独立 preflight）
- 当前：`runLadder` 统一 preflight（probe）→ 一次业务请求；`_safeReplayFactory`
  只放行空 GET/HEAD 的**单次**重放。实验开关未实现（默认路径保持 preflight）；
  若有数据支持，只允许覆盖 GET/HEAD/公开图片，mutation 永不纳入。

## P-NET-4（图片 route selection 惊群）
- 并发图片请求共用 `networkAccessPolicyProvider` 与 per-host route memory
  （`_HostRouteMemory`），无 per-host single-flight；未证实重复成本前不新增协调层。

## P-NET-5（图片 CDN relaxed transport 候选）
- 现状无 relaxed 路径；`insecureNoSniEnabled` 设置已随 settings D3 删除（C17 收口）。
  候选实现若做，仅限 i.pximg.net/s.pximg.net 且不得成为全局开关（本 task 未实现）。

## N-FIXED-1~9 回归证据
- N-FIXED 行为由 `restricted_compat_network_test.dart`（46 项）、`network_probe_test.dart`
  （19 项）、`tls_sni_behaviour_test.dart` 覆盖；本 task 阶段 1 只增加 timing 字段，
  未触碰可达性分类、route memory、preflight 顺序或证书校验。

## 阶段 1 交付
- `NetworkProbeStep.duration`、`NetworkProbeReport.totalDuration`、copyable text 输出
  `(Nms)`；探测本身不新增请求（沿用既有 I/O 注入）。
