# Replica v1 归档补强与集成修复

## Goal

承接开源 Pixiv 客户端审查和已归档 Replica v1 任务暴露的实现缺口，把跨代结果提交、写操作归属、Novel 标记、媒体恢复和 Android 平台边界拆成可独立验收的补强叶子。归档任务仍是历史事实源，不被改写；本父任务只协调补强与最终集成，不承载产品实现。

## Confirmed Facts

- `08-27-open-source-pixiv-app-plan-audit` 已固定第三方源码证据，并检查 Replica v1 的 17 个直接子任务。
- 原 Replica v1 的 17 个直接子任务和其实现叶子矩阵保持不变。本父任务是新的 top-level hardening coordination task，不计入原 17 项，也不替代任何原任务。
- 当前 `08-26-ugoira-player-export` 处于 `in_progress`，`08-26-restricted-compat-network` 仍是后续 P0 规划项；新叶子不能抢占或修改 Ugoira 在途实现。
- “大陆用户无需外部代理/VPN”是产品目标，范围为登录后使用：App 内可采用 Pixiv 域限定的 `Automatic` direct-first 与 DoH 候选 transport；省 SNI 需自行完成链验证与 SAN 核对，固定 IP 仅作兜底，不得用关闭证书校验、改写 `Host`、第三方反代或全局代理代替真实可用性。

## Requirements

### R1. 历史和归属

- 不修改 `.trellis/tasks/archive/2026-08/` 下任何历史任务、完成声明或实现文件。
- 每个补强叶子必须拥有自己的代码、测试、验证证据和回滚边界；父任务只维护依赖、顺序和集成 Gate。
- 所有任务保持 `planning`，只有各叶子再次完成 planning review 并获得明确批准后才可 `task.py start`。

### R2. 共享工程契约

- 需要网络的叶子统一依赖 `08-26-restricted-compat-network` 的 host-scoped `NetworkAccessPolicy`、direct-first、严格 TLS 与失败分类；不得自行复制 fallback 或引入代理。
- `Implemented`、`Compiled`、`Unit-tested`、`Device-tested` 必须分层记录；没有设备或真实 API 证据时保持 blocker，不得把 fixture/mock 当作产品成功。
- 所有异步提交都必须有 owner、generation/revision、取消和终态；旧请求、旧账号、旧 transport 和已 dispose 的对象不能写入当前状态。

### R3. 补强范围

- Feed：Recommended、Ranking、New、Search、Profile 的请求代际、共享实体和 cursor 一致提交。
- Mutation：Bookmark、Follow、Comments 及相关资料写操作的账号归属、去重、限流和失败恢复。
- Novel：typed markup token、未知标记保留、长文本预算和取消后的布局提交。
- Media：下载/Ugoira group、任务恢复、临时输出所有权、MediaStore pending 生命周期和有界资源。
- Android：WebView capability/route、intent/deep link、FileProvider、MediaStore 和后台生命周期边界。

### R4. 用户可见边界

- 不改变 beta56 已确认的导航、阅读、收藏、下载和账号体验，只修复竞态、恢复、安全边界和真实失败可观察性。
- 不把外部项目的 UI、客户端身份、密钥、固定 endpoint 或许可证受限源码直接复制进本项目。

## Acceptance Criteria

- [ ] 父任务和五个叶子均有完整 `prd.md`、`design.md`、`implement.md`，并在 task metadata 中声明 owner、依赖和最终集成阻塞关系。
- [ ] 五个叶子逐一建立“审查证据 -> 当前代码/归档缺口 -> 实现改动 -> 自动化/设备验证”的闭环；至少包含旧请求、旧账号、取消、重启或平台能力等负向用例。
- [ ] 原 Replica v1 父任务仍保留 17 个直接子任务；最终集成任务明确等待本父任务的叶子证据。
- [ ] 所有归档目录在补强规划和后续实现阶段保持无 diff；需要修复的代码只落在叶子所拥有的开放源码区域。
- [ ] 大陆无外部代理验收按 API/OAuth/pximg/download/WebView 出口、运营商、IP family、Android/WebView 版本和实际 route 分层记录；样本不足时不得作普遍可用声明。
- [ ] 本次规划阶段不启动、不提交、不推送，不声称产品代码、真实 API 或设备已经验证。

## Dependencies and Order

建议在当前 Ugoira 完成且兼容网络叶子完成基础契约后，按单 agent 顺序执行：Feed -> Novel -> Mutation -> Media -> Android；Media 必须等待 Ugoira 叶子完成，所有叶子完成后才进入 `08-26-replica-v1-integration-release`。

## Out of Scope

- 改写归档任务、重建原 17 项任务树或把补强伪装成历史已完成。
- 直接实现远程代理/中继、修改 Pixiv 服务端、增加 beta56 未定义的功能。
- 在规划阶段修改产品代码、数据库、Android manifest、网络配置或发布渠道。
