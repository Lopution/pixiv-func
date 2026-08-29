# 完成 Replica v1 集成与发布验收 — Design

## Objective

在所有功能叶子任务完成后，对 Replica v1 做跨功能、真机、安全、性能、许可和可发布构建验收，形成诚实的完成结论。

## Architecture and Boundaries

- EvidenceMatrix 以父 AC 为键，记录自动化/模拟器/真机/真实 API 四层证据和日期/版本。
- Integration tests 只使用可控测试 fixture；真实账号验证脚本/步骤不记录凭据或私有响应。
- Build/flavor 配置隔离 GitHub updater 与 F-Droid 权限；release signing 通过环境/本地忽略配置注入。
- 许可和归属文件与资产 provenance 一并审查，任何不确定依赖在发布前解决。
- HardeningMatrix 以 `feed-generation / mutation / novel-markup / media-recovery / external-capability / widget / updater-trust / live-feasibility` 为键；其中前四项由 `08-27-replica-v1-hardening` 的五个 owning leaves（Android platform 负责 platform-boundary）提交证据，每项都有自动化证据、设备/API证据和 blocker 状态。
- GenerationToken 从 request 发起一直传到 parsed entity/cursor commit；repository 不能在 controller 判旧之前写共享 store。mutation 则保持 server-confirmed，不建立隐式离线 outbox。
- WidgetSnapshot 是版本化、account revision 绑定的非秘密 render model；UpdaterTrust 先验签 manifest，再解析版本与 asset，最后验证 APK signing certificate。
- MainlandAccessEvidence 以 `(network sample, destination purpose, route kind, failure kind)` 为键，分别记录 OAuth/API/image/download/Widget headless；它只接受 `NetworkAccessPolicy` 产出的路由，不把一个 API 请求或第三方反代结果传播成 App 级可用。

## Data Flow

all archived child evidence → integrated build/test matrix → device/API/security/performance/license audits → resolve failures → parent AC review → final report；remote publication remains separately authorized。

Open-source hardening data flow：fixed-commit evidence → current-code reproduction → hardening test → pass 或 owning fix task → rerun matrix；不存在“文档建议即实现完成”的路径。

Mainland access data flow：`system proxy/VPN off + clean device → 分层探测（系统 DNS / DoH / TCP / TLS / 真实请求）→ direct-first probe → eligible strict fallback (DoH + direct connector) → destination-specific evidence → carrier/date scoped report`；证书不匹配、认证、HTTP、解析和取消失败不进入 fallback success 统计。

## Compatibility, Security, and Migration

- 集成任务不重新设计功能；发现 feature defect 时回到 owning child 或创建明确修复任务。
- 版本/Flavor migration 提供升级和清理说明，不修改用户数据 schema 而无 migration 测试。
- 已归档 task 保持只读。新增 hardening task 的完成证据由本任务引用，不回填为旧 archive 当时的能力。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。
- 是否存在 SNI 阻断必须由分层探测页的实测结果判定（TCP 通但带真实 SNI 的 TLS 握手失败），不能由设计推导。大陆样本缺失仍保持 release blocker/限制可见。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。
