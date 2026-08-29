# 完成 Replica v1 集成与发布验收 — Implementation Plan

## Start Gate

- 父任务规划已批准。
- 依赖任务已完成、提交并归档。
- 当前任务经最终 planning review 后由用户明确批准 task.py start。
- Codex 保持 inline 模式，不派发 subagent。

## Steps

1. 冻结候选 commit/依赖锁，建立父 AC、17 项开源审查与子任务 evidence matrix。
2. 等待并逐项消费 `08-27-replica-v1-hardening` 五个 leaves 的提交证据，再跑 hardening fixtures：feed generation/stale entity commit、mutation owner/revision、Novel typed markup、download/Ugoira group/restart/limits；失败即创建 owning fix task，不继续宣称 release candidate。
3. 先执行 MainlandAccessEvidence：系统 proxy/VPN off、无外部代理 App 的 API 36 干净设备，分 OAuth/API/refresh、pximg、download/Ugoira、accounts/WebView、Widget headless 记录 route/failure、日期、运营商、网络类型、IP family、Android/WebView；不足三运营商只保留已测范围。
4. 核验 Reverse Image provider capability、Profile capabilities/patch、Live endpoint/HLS feasibility；记录观测日期和真实 blocker，不用 fixture 替代外部接受。
5. 审计 Widget snapshot/headless auth/WorkManager/RemoteViews IPC/PendingIntent，以及 signed updater manifest、size/hash/package/signer 和 F-Droid compile-time isolation。
6. 运行静态、全量测试、CI、debug/release/flavor 构建和 merged-manifest 审计；确认所有 Pixiv 出口消费 shared `NetworkAccessPolicy`，Updater/翻译/反向搜图未进入兼容路径。
7. 执行 API 36 真机+真实账号功能、intent、lifecycle、failure、进程重启和升级矩阵。
8. 执行安全/秘密/网络/权限/clipboard/FileProvider 审计及媒体/长列表性能测试，特别检查 direct-first、证书链与 SAN 校验、`Host` 未被改写、DNS/route isolation 和 POST no-replay。
9. 修正 LICENSE/NOTICE/README/归属/版本/构建说明，核对第三方许可、资产 provenance 和本次仅设计参考的固定 commit 清单。
10. 搜索并清除业务 placeholder/no-op/mock，复跑所有受影响门禁。
11. 记录最终结果和 blocker；仅在全部 AC 真实满足时完成并归档父任务。

## Validation

通用门禁：

```bash
/opt/flutter-3.47.0/bin/flutter pub get
/opt/flutter-3.47.0/bin/flutter analyze
/opt/flutter-3.47.0/bin/flutter test
python3 ./.trellis/scripts/task.py validate .trellis/tasks/08-26-replica-v1-integration-release
git diff --check
```

涉及 Android、插件、资源或平台行为时同时运行：

```bash
/opt/flutter-3.47.0/bin/flutter build apk --debug
```

专项验证：

- 全量 flutter analyze/test/integration_test 与 CI。
- debug/release、GitHub/F-Droid flavor 和 merged manifest。
- 真实账号功能矩阵、API 36 back/deep-link/SEND/MediaStore/WebView。
- token/网络/进程/旋转/低存储/取消/恶意输入失败矩阵。
- 内存/CPU/后台活动与 secret/license/placeholder 审计。
- Feed old-generation response 在取消失效后返回也不能写 Illust/User store 或 cursor。
- Novel newpage/chapter/ruby/jump/image/unknown fixtures 和长文本单批预算。
- Download/Ugoira task-group terminal、restart/pending cleanup、archive/frame/pixel limit 与峰值内存。
- Widget account revision/last-good/IPC budget/headless proof；Updater invalid signature/hash/size/package/signer 与 F-Droid no-network。
- Mainland clean-device matrix：system proxy/VPN off、无外部代理 App；OAuth/API/refresh/pximg/download/Ugoira/Widget headless 分出口；automatic eligibility、DoH/direct TLS、分层探测结果、carrier/date/IP-family evidence。
- Reverse Image structured/WebView/unavailable capability 和 Live 当日 endpoint/HLS 证据。

## Risky Files and Rollback Points

- LICENSE、NOTICE/ATTRIBUTION、README、pubspec version、CI、Android flavors/signing、integration_test/、跨功能配置

## Completion Gate

- PRD 全部 acceptance criteria 有真实证据；自动测试、模拟器、真机和真实账号验证明确区分。
- 不残留本任务范围内的 placeholder、空操作、隐藏 mock、吞错或安全绕过。
- 运行 trellis-check，按需更新 spec，提交仅包含本任务文件，再执行 finish/archive。

---

## 任务关闭（2026-08-29）

许可/归属修正、依赖与资产 provenance 审计、静态审计、全量测试（405 tests）与四变体构建均已
完成，证据见 `research/integration-evidence.md` 与 `research/license-audit.md`。

**Replica v1 尚未达到可发布验收状态。** 该文件结论中的 5 条 blocker 全部转入
`08-29-replica-v1-completion` R5：

1. API 36 设备门禁（本仓库无 API 36 镜像；物理 RMX5200 当时开着 VPN，未采信）
2. Live 外部能力（当日真实 API 三个 filter 均 `lives=0`）
3. Profile 写入与反向搜图缺少审定的官方合约
4. 可分发签名与 updater 信任根（release 仍是 debug signing）
5. 真实写操作、fresh OAuth 交换/刷新、重启后 widget restore 未覆盖

归档状态为 `completed` 表示**本任务关闭、验收所有权移交**，
**不代表 Replica v1 已通过发布验收**。在上述 blocker 解决并重跑完整矩阵前，
不创建远程 release，不宣称大陆普遍可用。
