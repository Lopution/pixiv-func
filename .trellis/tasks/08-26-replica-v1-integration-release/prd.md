# 完成 Replica v1 集成与发布验收

## Goal

在所有功能叶子任务完成后，对 Replica v1 做跨功能、真机、安全、性能、许可和可发布构建验收，形成诚实的完成结论。

## Confirmed Facts

- 父 PRD定义 10 项跨任务 AC，且明确远程发布不属于自动授权。
- 当前 LICENSE 文件头是 GPL v3，而 README/需求要求 AGPL-3.0-only 和原作者归属，必须在发布前修正。
- 单个子任务通过不能证明账号、生命周期、Android intent、媒体和 flavor 的集成行为。

## Dependencies

- 父任务除本任务外的所有叶子任务均已完成、提交并归档。
- 没有未解释的产品占位、活跃迁移或未决 breaking decision。

## Requirements

- R1: 建立父 PRD AC1–AC10 的证据矩阵，逐项链接子任务、测试、设备/API 样本和未验证边界。
- R2: 运行全量 analyze、unit/widget/integration tests、debug APK 和 release 构建；CI 使用同一锁文件/SDK 契约。
- R3: 在 API 36 真机用真实账号执行 Startup、OAuth、Token refresh、Recommended、Detail、Bookmark、Search/Profile/Novel/Download/Ugoira/Live、deep links、SEND 和 back 主路径。
- R4: 执行账号切换、token 失效、网络失败、进程重启、前后台、旋转、低存储、取消和恶意外部输入的跨功能回归。
- R5: 审计 TLS/cleartext/Manifest 权限、secret/log/storage、URL allowlist、FileProvider、clipboard 和 updater flavor 边界。
- R6: 检查长列表、图片、下载、Ugoira、Novel layout、History 和 Live 的内存/CPU/生命周期，记录设备和阈值而非笼统声称。
- R7: 修正为 AGPL-3.0-only，加入原作者 git-xiaocao 归属和修改说明，核对第三方依赖/资产许可与 README。
- R8: 建立 GitHub/F-Droid flavor 的 release 配置和无密钥可复现说明；签名材料只从外部安全输入读取，不进入仓库。
- R9: 移除或关闭所有业务 placeholder/no-op/隐藏 mock；无法真实验收的功能保持 blocker，不计入 Replica v1 完成。
- R10: 生成最终状态报告、已知限制、安装/验证说明；不自动 push、创建 release 或发布商店。

## Acceptance Criteria

- [ ] 父 PRD AC1–AC10 每项均有可复查证据，所有子任务状态 completed/archived。
- [ ] flutter analyze 零问题；全量 unit/widget/integration tests 和 CI 通过，debug/release 构建成功。
- [ ] API 36 真机和真实账号主链及平台矩阵全部通过，失败项有复现和 blocker，不被隐藏。
- [ ] merged manifests/flavors、TLS、秘密存储、日志、外部输入和依赖/许可审计无未处理 P0/P1。
- [ ] 性能与生命周期测试没有无界缓存、重复完成、持续后台 timer 或可复现关键泄漏。
- [ ] LICENSE/NOTICE/README/版本/权限/构建说明相互一致，仓库不含签名密钥和凭据。
- [ ] 源码搜索与人工走查确认没有业务占位、空操作或伪成功。
- [ ] 最终报告明确 commit、命令、真实结果、device-tested/API-tested 边界和剩余限制；未执行远程发布。

## Out of Scope

- Play Store、F-Droid、GitHub Release 的实际发布。
- Replica v1 之后的 UX Evolution。
- 替用户创建或持久化签名/服务凭据。
- 以降低断言或禁用测试换取通过。

## Risks and Deferred Items

- 真实 Pixiv/Live API、账号、设备或签名材料不可用会成为明确 blocker；本任务不能用 mock 将其转为接受。

## Source Anchors

- 父任务 prd.md、design.md、implement.md 与全部子任务 artifacts
- 当前 README.md、LICENSE、Android manifests/flavors、CI/测试配置

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
