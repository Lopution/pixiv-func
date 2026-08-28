# 实现安全剪贴板账号迁移 — Design

## Objective

保留beta56复制账号数据→另一设备粘贴登录的入口，同时去除硬编码AES并诚实约束clipboard传输威胁。

## Architecture and Boundaries

- TransferEnvelope parser为纯Dart、大小受限、版本化；ReplayStore只保存nonce摘要/expiry且不含credential。
- ClipboardAdapter封装sensitive flag、conditional clear和当前content fingerprint。
- ImportService执行parse→validate→credential verify→atomic account write，任何失败都清理内存secret。
- ThreatModel文档区分 at-rest、accidental corruption、target-local replay、malicious clipboard reader 与 writer；checksum/replay store 的能力名称与真实保证一致。

## Data Flow

user long-press export → build expiring envelope → sensitive clipboard → destination explicit paste → parse/checksum/target-local replay → Pixiv credential verify → secure atomic import → conditional clear。

## Compatibility, Security, and Migration

- 可见入口保持beta56；内部格式不兼容旧硬编码AES payload，旧格式必须明确拒绝或单独受控一次性迁移并经审批。
- 二维码/公钥方案留Evolution。
- Envelope 不承担 authenticity；未来口令/配对方案必须使用新的 version/UX，不能让当前 parser 猜测两种威胁模型。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。
