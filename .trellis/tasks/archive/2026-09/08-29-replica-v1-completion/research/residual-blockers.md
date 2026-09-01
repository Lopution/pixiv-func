# 残余阻塞清单

Replica v1 的 32 个实现叶子（27 个原树 + 5 个 hardening）已全部归档，逐项核对见文末。
其中 3 个在归档时仍是 `in_progress`，归档动作把状态强制写成 `completed`——**那只表示任务
关闭、验收所有权移交本任务，不表示验收通过**。它们的未完成项在此逐条展开。

按「什么才能解开」分类，因为三类的处置方式完全不同。

## A. 用户可解：需要设备或真实账号

本任务交付 debug APK，验证由用户执行。

| # | 项 | 来源 | 阻塞原因 | 解开需要 |
|---|---|---|---|---|
| A1 | API 36 设备门禁 | integration / widgets / updater 三方共有 | 本仓库无 API 36 MuMu 镜像；物理 RMX5200 当时开着 VPN，证据未采信 | 一台 API 36 设备，关闭 VPN。覆盖 WebView、MediaStore、headless Worker、intent 生命周期 |
| A2 | API 29 验收 | 本任务 R4 新增 | 现有证据全部是 API 35，API 29 从未跑过 | 一台 API 29 设备/镜像 |
| A3 | 系统重启后 widget restore | widgets | 未跑过重启矩阵 | 真机重启后观察小组件是否保留、是否重新拉取 |
| A4 | 真实写操作 + fresh OAuth | integration | 最终集成只重跑了已登录读链路 | 真实账号执行收藏/关注/评论/资料写入；一次全新 OAuth 交换与 refresh |
| A5 | 大陆分层探测报告 | 本任务 R2 新增 | 无实测数据 | 境内网络、系统 proxy/VPN 关闭、无外部代理 App；按日期/运营商/网络类型/IP family 记录 |

**A5 是唯一决定 Phase 2 是否存在的输入**，其余四项是验收补证。

## B. 需要用户提供密钥材料

| # | 项 | 来源 | 阻塞原因 | 解开需要 |
|---|---|---|---|---|
| B1 | updater 信任根 | updater | 生产 keystore、公钥、签名 manifest、匹配 signer 均不存在。因此验签成功 → 未知来源授权 → 安装 / 安装拒绝 / 用户取消 这整条系统分支**无从验证**，不是漏测 | 生成生产 keystore，把公钥编进 app，用同一 key 签 manifest 与 APK |
| B2 | 可分发签名 | integration | release 仍是 debug signing，不能创建远程 release | 同 B1 的 keystore |

B1/B2 是同一份密钥材料的两个用途。按全局规则，密钥不进仓库、不硬编码——走环境变量或
`key.properties`（已 gitignore）。**在密钥到位前，验签安装分支保持未验证，不用 mock 冒充。**

## C. 外部依赖，客户端侧无解

这两项不是「还没做」，是**当前没有可实现的正确形态**。按 feasibility blocker 保留可见的
unavailable 状态，不写 mock、不引依赖。

| # | 项 | 来源 | 阻塞原因 | 何时重估 |
|---|---|---|---|---|
| C1 | Profile 写入官方 route | profile-edit / integration | 没有审定过的官方写入合约 | 找到并核实官方 route 后 |
| C2 | 反向搜图 provider | reverse-image-search / integration | 没有可接受的 provider / credential / privacy 方案 | 有符合隐私边界的方案后 |

这两项的形态是对的：无证据就不实现，好过实现一个猜出来的合约。

### 已移出范围：Live

Live 曾是 C 类第三项（核验当日三个 filter 均 `lives=0`，取不到 valid id / detail schema /
HLS manifest）。**2026-08-29 决定直接移出 Replica v1 范围**，不再作为待解阻塞：受众极小，
而实现成本高且不可控——没有公开接口合约，必须先抓到真实 live object 才能确定 schema。
继续挂在清单里会让它看起来是「早晚要做」，实际不会做。理由记在父 PRD 的 Out of Scope；
归档叶子 `08-26-live-player` 保留为该判断的记录。

## 叶子归档核对

27 个原树实现叶子 + 5 个 hardening 叶子 = 32，全部在 `archive/2026-08/`，无遗漏。
另有 10 个非实现目录一并归档：1 个 trellis init 空壳、6 个只做范围协调的中间父任务、
1 个 hardening 协调父、1 个开源审查研究任务、1 个防御代码清理任务。

归档时状态为 `in_progress` 的 3 个叶子及其去向：

| 叶子 | 已交付且有证据 | 转入本任务 |
|---|---|---|
| `08-26-android-home-widgets` | API 35 真机 8 张截图 | A1、A3 |
| `08-26-updater-flavors` | 四变体构建、merged manifest 权限审计、F-Droid 无安装权限（API 35 真机确认） | A1、B1 |
| `08-26-replica-v1-integration-release` | 许可/归属修正、provenance 审计、静态审计、全量测试、四变体构建 | A1、A4、B2、C1、C2（Live 已移出范围） |

这三个叶子的 `implement.md` 末尾各有一段任务关闭说明，写明 `completed` 的口径与去向；
`task.json.notes` 同义。归档件按 `writeback_policy: archives_read_only` 不再回写。
