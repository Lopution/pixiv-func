# 审计原文来源

## 来源与基线

- 提供者：用户在 2026-09-02 对话中粘贴的网页版 ChatGPT 审计全文。
- 审计日期：2026-09-01。
- 审计基线：`9d1cb1b809e20b67e2409970938b221e178d292e`。
- 附件 SHA-256：`bc5f20437a26571feaf6bc35c88674fdd70eedd0597c0dadef9c6f9aac5e00b6`。
- 原文归档：`research/audit-original-2026-09-01.md`（1168 行，原样复制）。

本文件不是重新编写的审计，而是来源与缺口索引；原文全文见上面的归档文件。原文中的代码
行号只适用于上述审计基线；实现前仍需对当前 HEAD 重新定位并确认行为。

## 此前缺失项已找回

此前 `audit-verification.md` 和两个 child PRD 标记为缺失的 11 项，原文定义如下；后果、
简要方案和验收顺序的完整上下文见 `audit-original-2026-09-01.md`。

### 行为正确性

- **C2**：收藏、关注、评论在 token 刚过期时，第一次操作可能因 `allowAuthReplay:false`
  被主动判败；明确 401/`invalid_grant` 后应 refresh 并只重放原 mutation 一次，超时/reset
  等结果未知的情况不得自动重放。
- **C3**：Entity Store 的单调合并会永久保留旧 caption、tags、`visible` 和 page count；
  sparse/feed payload 与 detail/authoritative payload 需要不同的 merge 语义。
- **C4**：download recovery 持久化 `credentialRevision`，而内存 revision 在进程重启后从 0
  开始，导致同账号合法任务被判 orphan；owner 应改为稳定 account/job/output/destination。
- **C14**：production 已使用 preflight ladder，但仍保留只为 legacy/test caller 的
  `_runLegacyLadder`；确认调用方迁移完成后删除第二套语义。
- **C21**：删除账号时未明确清理尚未同步到 Pixiv 的 history remote outbox，账号重新加入后
  可能补发旧操作；只清该账号未发送 outbox，不顺手删除本地 history。
- **C22**：Download/Ugoira orphan recovery 可能留下用户没有出口的状态；只清理能以 owner/job
  明确证明属于 Func 的 pending output，不为低概率 orphan 新建 Recovery Center。

### 网络性能

- **P-NET-1**：PixEz 信息流默认 medium，Func 当前默认 large；先验证 D4 的 preview medium
  对体积和首屏/滚动的收益。
- **P-NET-2**：无 host/group memory 时 Automatic 仍可能 direct-first；在已知大陆环境与
  外部网络环境分别 A/B API/OAuth 组的 ECH-first 与 direct-first，不能按国家硬编码。
- **P-NET-3**：GET/HEAD/image 可实验“业务请求即 route attempt，transport failure 后最多换路
  一次”，POST/PATCH/DELETE 继续 preflight 后只发送一次。
- **P-NET-4**：第一屏多张 `i.pximg.net` 图片可能同时触发 route selection；先 trace 是否真的
  重复 DoH/probe，再决定是否加入 per-host single-flight。
- **P-NET-5**：PixEz 使用 cached/fixed IP、no-SNI、可选 relaxed certificate path；若实测有
  明确收益，Func 只允许在公开图片 CDN 内部使用，API/OAuth 保持 strict。

## 规划影响

`09-01-behavior-correctness-cleanup` 与 `09-01-network-perf-ab` 的“缺原文”硬 gate 已解除，
但两个 child 仍须在 `task.py start` 前完成当前 HEAD 的断言复核、设计 review 和各自的真实
环境/数据前置条件。D1/D2 中的 SauceNAO、百度额度与实名认证仍是会变化的外部事实，不能只
凭审计日期的描述当作永久保证。
