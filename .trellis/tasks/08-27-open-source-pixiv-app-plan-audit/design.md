# 开源 Pixiv App 规划审查 — Design

## Objective

把固定 commit 的第三方源码证据转成 Replica v1 可执行、状态感知且不改变 beta56 体验的规划改进，而不是形成泛泛的项目推荐清单。

## Evidence hierarchy

1. 当前仓库代码、测试、Trellis task status 和 beta56 固定 commit 决定已经实现了什么、用户可见行为是什么。
2. 第三方固定 commit 的实际源码、测试和注释证明某个实现模式或失败现场。
3. GitHub 元数据只用于活跃度与许可证筛选，不证明代码质量。
4. 未找到实现只能维持或提高 research gate，不能推导服务一定不可用。

## Audit model

每条建议保存以下字段：`topic`、`external evidence`、`current repo fact`、`decision(adopt/guarded/defer/reject)`、`owner task`、`priority`、`validation`。17 个直接子任务是覆盖索引，中间父任务继续映射到实际叶子。

## Status-aware write-back

- `archived`：只读历史；新缺口进入 Replica 父契约或最终集成任务。
- `planning intermediate parent`：更新共享契约和跨叶子 Gate，不让父任务承载产品代码。
- `planning leaf`：可更新 PRD/design/implement，但保持独立 start approval。
- 任何需要新增实现任务的缺口先写为集成 blocker；本轮用户已允许创建叶子，已落地独立 `08-27-replica-v1-hardening` 父任务及五个 owning leaves。它们仍需各自 final planning review 和明确 start approval。

## Key cross-task contracts

- 大陆 access 是 shared transport outcome，不是 feature URL rewrite：exact-host registry、failure taxonomy、direct-first、secure resolver、original-host TLS 与 network revision 由兼容网络 leaf 统一拥有，API/OAuth/image/download/WebView/Widget逐出口验收。
- ECH 是 capability-gated optional adapter；loopback `CONNECT` 只解决 WebView DNS/route steering，不能宣称隐藏 SNI。Q1 已允许自动 fallback/loopback，但二者仍分别受 ECH/WebKit capability、exact-host 和严格 TLS Gate 约束，任何证书/SNI/Host/固定 IP/反代降级继续拒绝。
- Feed result 的网络、解析、实体 merge 和 cursor commit 同属一个 generation。
- mutation 继续 server-confirmed；只借鉴 owner/dedupe/retry 分类，不默认后台重放。
- 大媒体以 task-group、owned temporary output、bounded decode 和 exactly-one terminal 为边界。
- Widget native 层优先消费无秘密、版本化、账号隔离 snapshot；Updater 只信任固定公钥验证后的 manifest。
- 外部服务能力无法证明时保持 blocker，不用 WebView/fixture/mock 偷换 beta56 验收。

## License and security boundary

本任务只保存链接、事实摘要、状态机和测试思想。GPL 源码不复制；MIT 代码未来若复用也需单独 provenance。第三方秘密、固定 IP、TLS 绕过、BODY 日志、整文件 bytes 和 destructive migration 都列为拒绝证据。

## Data flow

GitHub metadata → pin commit → inspect source/tests → compare current plans/code → 17-task matrix + mainland access transport audit → status-aware task write-back → Trellis/cross-reference/diff validation → Q1 decision record/user final planning review。

## Rollback

本任务只修改 `.trellis/tasks/` 规划与研究文件。Q1 已形成决策记录；若未来产品范围改变或某建议被否决，可逐项回滚对应规划段落，不需要回滚产品代码、数据 schema 或远程状态。当前 Ugoira 产品改动属于独立在途工作，本任务不编辑或回滚。
