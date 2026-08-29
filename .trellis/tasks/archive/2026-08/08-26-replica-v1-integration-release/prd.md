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
- R11: 执行跨代 feed commit 审计：Recommended/Ranking/New/Search/Profile 的 refresh 必须取消旧 append，旧 generation 即使网络最终返回也不得 merge entity、bookmark snapshot 或 cursor；覆盖相同/新增/删除/重排/完全不相交 ID 与账号/筛选切换。
- R12: 执行 mutation ownership 审计：Bookmark/Follow/Comments/Profile edit 的 pending operation 绑定 account/revision，429/Retry-After 与 terminal failure 可区分；Replica v1 不持久化并自动重放失败写操作，非幂等 mutation 不隐式重试。
- R13: 补齐 Novel typed markup 与媒体恢复 Gate：Novel 覆盖 newpage/chapter/ruby/jump/image/unknown tokens 和长文预算；Download/Ugoira 覆盖 task-group exactly-once、submission snapshot、owned cleanup、进程重启/pending item 策略与 archive/frame/pixel 上限。
- R14: 执行 late-platform trust Gate：Reverse Image 记录 provider capability/观测日期，Profile edit 使用 capabilities+typed patch，Widget 使用 versioned secret-free account snapshot/受证 headless path，GitHub updater 只信任 signed manifest 并校验 size/hash/package/APK signer；F-Droid 编译期无 updater。
- R15: Live 必须以当日真实 endpoint/auth/schema/HLS 证据进入播放器验收；所选开源客户端未提供当前可复用 Live 证据，不能用 fixture、旧 fixed-IP proxy 或 mock 降低 blocker。开源参考代码/资源未直接复制，许可证与 provenance 审计有记录。
- R16: 以“中国大陆用户无需外部代理/VPN仍能尽可能使用 Pixiv”为独立 release Gate：在系统 proxy/VPN 关闭、无外部代理 App 的 API 36 真机上分出口验证 OAuth/API/refresh、pximg 图片、下载/Ugoira、accounts/WebView 与 Widget headless；记录日期、运营商/网络类型、IPv4/IPv6、Android/WebView、route/failure。三运营商样本不足时只能报告已测范围，不得宣称大陆普遍可用。
- R17: Mainland access 必须消费 `NetworkAccessPolicy` 的 exact-host registry、direct-first、failure taxonomy 与 DoH/original-host TLS；不得用关闭证书校验、改写 `Host`、第三方反代、全局 proxy 或单一 API 成功替代全出口验收。范围为登录后使用的四个 Pixiv 自有主机；省 SNI 在自验证前提下允许，固定 IP 仅作兜底。

## Acceptance Criteria

- [ ] 父 PRD AC1–AC10 每项均有可复查证据，所有子任务状态 completed/archived。
- [ ] flutter analyze 零问题；全量 unit/widget/integration tests 和 CI 通过，debug/release 构建成功。
- [ ] API 36 真机和真实账号主链及平台矩阵全部通过，失败项有复现和 blocker，不被隐藏。
- [ ] merged manifests/flavors、TLS、秘密存储、日志、外部输入和依赖/许可审计无未处理 P0/P1。
- [ ] 性能与生命周期测试没有无界缓存、重复完成、持续后台 timer 或可复现关键泄漏。
- [ ] LICENSE/NOTICE/README/版本/权限/构建说明相互一致，仓库不含签名密钥和凭据。
- [ ] 源码搜索与人工走查确认没有业务占位、空操作或伪成功。
- [ ] 最终报告明确 commit、命令、真实结果、device-tested/API-tested 边界和剩余限制；未执行远程发布。
- [ ] Feed generation、mutation ownership、Novel markup、media recovery、Reverse/Profile、Widget、signed updater 与 Live feasibility 矩阵全部关闭；任何未满足项已创建 owning fix task 或保持 release blocker。
- [ ] GitHub updater 对无公钥、无效签名、checksum/size/package/signer mismatch 全部 fail closed；F-Droid merged manifest、依赖与运行时网络均无 self updater。
- [ ] 第三方源码 provenance 审计确认未复制 GPL/MIT 实现、资源、秘密或网络常量；只保留固定 commit 链接和设计证据。
- [ ] Mainland no-external-proxy Gate 完成：每个 Pixiv transport 出口有真实或明确 blocker 证据，route/failure/网络样本可追溯；未达到样本覆盖时发布说明不会扩大为“大陆可用”。

## Out of Scope

- Play Store、F-Droid、GitHub Release 的实际发布。
- Replica v1 之后的 UX Evolution。
- 替用户创建或持久化签名/服务凭据。
- 以降低断言或禁用测试换取通过。

## Risks and Deferred Items

- 真实 Pixiv/Live API、账号、设备或签名材料不可用会成为明确 blocker；本任务不能用 mock 将其转为接受。
- 已归档任务中的新缺口不能通过改写 archive 记录消失；集成任务发现需要代码修复时，创建 owning hardening task 并在其通过后回到本任务复验。

## Source Anchors

- 父任务 prd.md、design.md、implement.md 与全部子任务 artifacts
- 当前 README.md、LICENSE、Android manifests/flavors、CI/测试配置
- `../08-27-open-source-pixiv-app-plan-audit/research/source-evidence.md` 与 `task-audit-matrix.md`

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
