# 技术设计：正式签名、更新清单验签与 GitHub 资产下载

设计基线：当前规划基线 `409df51`；发布阻塞现状与 G1 补充见本 child `prd.md` 和上级
`research/audit-verification.md`。本设计只建立最小可发布链路，不增加密钥轮换、多公钥、
回滚保护或新的分发渠道。

## 1. 开工 gate 与决策

以下事项已由用户于 2026-09-02 一次性确认：

- D-1：agent 在仓库外生成 APK keystore，用户负责离线加密备份。
- D-2：agent 生成独立 manifest signing key，不能与 APK keystore 混用。
- D-3：采用 `SHA256withECDSA` P-256，并用 API 29 fixture 验证。
- D-4：建立 GitHub Actions，release workflow 手工触发；本轮允许 Secrets 和 draft Release。
- D-5：保留 `fdroid` flavor，维持 store-managed/关闭 updater，不公开 F-Droid release。

密钥仍只在仓库外生成，所有远程发布动作限定在 L2 的 draft Release 范围内。

## 2. 两条签名链路

### 2.1 APK signing

`github` release 使用项目正式 keystore，密码、alias、store file 通过 Gradle property 或环境
变量注入；缺少材料时明确失败并给出配置名，绝不静默退回 debug。`fdroid` 依据 D-5：若由
F-Droid 构建服务签名则不把本地 release key 注入其 flavor；若不发布则保留现有禁用 updater
能力，并记录维护决定。

### 2.2 Update manifest signing

生成侧产出与 `update_manifest.dart` 严格兼容的 `update-manifest.json` 和二进制
`update-manifest.sig`。manifest 内容只包含版本、APK URL、SHA-256、长度等已有字段；签名
覆盖 canonical UTF-8 bytes（换行、编码和 JSON 序列化规则写入脚本帮助与 fixture）。私钥通过
受控环境变量/secret file 输入，脚本默认拒绝将 key 或签名私材打印到 stdout/stderr。

消费侧 `DistributionUpdaterChannel`：

- 采用 D-3 确认的 API 29 标准算法和公钥编码；去掉 Ed25519-only 与固定 64 字节判断。
- 返回可诊断错误分类：`public_key_missing`、`algorithm_unavailable`、`signature_mismatch`
  （以及输入格式/IO 错误），Dart 层展示安全的 code，不暴露 key 或 manifest 私密内容。
- 保留 `fdroid` flavor 的 no-op/store-managed 能力；不要让 GitHub verifier 代码进入 F-Droid
  的网络或安装权限边界。

## 3. Release 资产与 URL 合同

- release 流程先构建并检查 APK 证书指纹，再生成 manifest、签名并上传三个资产。
- `kUpdateDownloadHosts` 精确加入 `release-assets.githubusercontent.com`，不改为通配符。
- 通过真实 GitHub release 资产记录完整 redirect chain；最终 URL 必须同时满足精确 host、HTTPS、
  `.apk` path、无 userinfo/fragment 和现有大小/哈希校验。若 GitHub 最终 path 不以 `.apk` 结尾，
  必须调整资产 URL/解析合同，不能只放宽后缀检查。

## 4. 文件责任

| 责任 | 主要文件 |
|---|---|
| Gradle signing/flavor | `android/app/build.gradle.kts`、`android/app/src/{github,fdroid}` |
| verifier | `android/app/src/github/kotlin/io/github/lopution/pixivfunc/DistributionUpdaterChannel.kt`、`lib/core/updater/update_platform.dart` |
| manifest parser/service | `lib/core/updater/update_manifest.dart`、`update_service.dart`、`update_providers.dart` |
| signer/generator | `tool/` 下最小可复现脚本及使用说明；不保存 key |
| release wiring | `.github/workflows/`（仅 D-4 选择 CI 时新建）或手工 release SOP |
| download allowlist | `lib/core/download/download_request.dart` |
| tests | updater manifest/download/flavor tests、Kotlin verifier tests、signer fixture tests |
| research | `research/api29-verifier.md`、`research/github-redirect.md`（不写密钥） |

## 5. 测试与安全不变量

- 同一 manifest 的签名/验签 fixture 在本地和 API 29 verifier 上结果一致；篡改正文或签名均
  返回 `signature_mismatch`。
- 缺公钥、算法不可用、签名不匹配三类错误不能合并为 `false` 后静默跳过。
- 私钥不进入 git、APK、manifest、日志、CI artifact；公钥只以现有 DER base64 注入。
- GitHub asset redirect 只允许精确 host 集合；非 `.apk` 或非 HTTPS 被拒绝。
- updater 只作用于 `github` flavor；F-Droid 继续走 store-managed 路径。

## 6. 风险与回滚

- 先完成 verifier/signer 离线 fixture，再接 Gradle signing，最后做真实 release/设备安装；每
  阶段独立可回滚。
- 算法 provider 或签名格式在 API 29 不一致时停在 research，不通过换回 Ed25519 逃避基线兼容。
- 真实 key 泄漏、redirect 越界或 F-Droid 分发策略未定时，立即停止发布链路，不生成可被用户
  安装的“临时正式包”。
