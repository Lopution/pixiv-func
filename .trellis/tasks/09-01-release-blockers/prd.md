# 发布阻塞：签名、updater 验签、GitHub CDN

## Goal

解决三项使 1.0 无法真正发布的阻塞项。三项都不是「代码写得不好」，而是**发布链路缺失**：
release 用调试密钥签名、自更新验签算法在基线设备上必然失败、下载 host 白名单不含
GitHub 实际重定向到的域。

覆盖审计编号：R1, R2, R3（含复核文档的 G1）。

## 现状核实（HEAD `409df51`）

### R1 现状：release 用 debug 签名

`android/app/build.gradle.kts:58-64`：

```kotlin
buildTypes {
    release {
        // TODO: Add your own signing config for the release build.
        // Signing with the debug keys for now, so `flutter run --release` works.
        signingConfig = signingConfigs.getByName("debug")
    }
}
```

产品后果不止「不正规」：debug keystore 每台机器不同、且 Android Studio 的默认 debug key
有效期与内容都不由项目控制。用它签的包一旦分发出去，**后续任何用正式密钥签的版本都无法
覆盖安装**，只能让用户卸载重装（丢全部本地数据）。所以这一项必须在有任何真实用户之前完成。

### R2 现状：验签用 Ed25519，基线设备上必然失败

`android/app/src/github/kotlin/.../DistributionUpdaterChannel.kt:50-71`：

```kotlin
val key = KeyFactory.getInstance("Ed25519")
    .generatePublic(X509EncodedKeySpec(keyBytes))
Signature.getInstance("Ed25519").run { ... }
```

- AOSP 的 Conscrypt 从 **API 33** 才提供 `Ed25519`。项目基线是 **API 29**（`minSdk = 29`，
  且 API 29 明确在 parent 的验收矩阵里）。
- 代码里**没有 API 级别判断**，异常被 `catch (_: Exception) { false }` 吞掉。
  结果是在 API 29–32 上验签静默返回 `false`，自更新**永远失败且没有可诊断的原因**。
- 签名长度被硬编码为 `signature.size != 64`（Ed25519 raw 长度）。换算法时这条也要改
  ——ECDSA 是 DER 编码、长度可变。

### G1 现状：签名生成侧完全不存在

复核文档 G1 已指出，本次核实再次确认：

- `tool/` 是**空目录**。
- `.trellis/scripts/` 只有 Trellis 自身的 Python 脚本，没有 manifest 签名脚本。
- **仓库根本没有 `.github/workflows/`——CI 完全不存在。**

manifest 与签名的消费侧已经写好了（`lib/core/updater/update_manifest.dart:17,21`）：

```
https://github.com/Lopution/Pixiv-func/releases/latest/download/update-manifest.json
https://github.com/Lopution/Pixiv-func/releases/latest/download/update-manifest.sig
```

也就是说：**app 已经在等两个从来没有人生产过的文件**。R2 的真实工作量是从零建立生成侧
（密钥生成 → manifest 生成 → 签名 → 随 release 上传），不是「换个算法名」。

公钥注入路径已就位：`build.gradle.kts:43-49` 从 gradle property
`PIXIV_UPDATE_PUBLIC_KEY_DER_B64` 读取，`fdroid` flavor 固定为空且自更新关闭。

### R3 现状：host 白名单缺 GitHub 实际下载域

`lib/core/download/download_request.dart:142-147`：

```dart
const Set<String> kUpdateDownloadHosts = <String>{
  'github.com',
  'objects.githubusercontent.com',
  'github-releases.githubusercontent.com',
};
```

不含 `release-assets.githubusercontent.com`。

## Requirements

### R1. 正式签名

- release 构建使用项目自己的 keystore，不再复用 debug 签名配置。
- 密钥材料与密码**不进仓库**：走 gradle property / 环境变量 / CI secret，
  与已有的 `PIXIV_UPDATE_PUBLIC_KEY_DER_B64` 注入方式保持一致的风格。
- 本地无密钥时的行为必须**明确**：要么构建失败并说明原因，要么显式落到一个标记为
  「非发布」的配置。不允许静默退回 debug 签名——那正是当前这个 bug 的成因。
- `github` 与 `fdroid` 两个 flavor 的签名策略分别确认（F-Droid 通常由其构建服务器
  自行签名，策略取决于是否真的上架，见 Open Questions）。

### R2. Updater 验签在 API 29 可用

**消费侧**：

- 换用 API 29 上确实可用的签名算法。`SHA256withECDSA`（P-256）与 `SHA256withRSA`
  在 API 29 的标准 provider 中均可用；具体选型在 design 阶段定并写明依据。
- 去掉硬编码的 64 字节长度判断，改为与所选算法一致的校验。
- **验签失败必须可诊断**：区分「未配置公钥」「算法不可用」「签名不匹配」三种情况，
  不再全部塌缩成 `false`。当前这种吞异常写法本身就是 parent R1「错误 guard 阻塞正常
  行为」的实例。

**生成侧（G1，工作量主体）**：

- 密钥生成步骤有可复现的记录（命令、参数、产物格式）。
- 建立 manifest 生成 + 签名脚本，产出 `update-manifest.json` 与 `update-manifest.sig`，
  格式与 `lib/core/updater/update_manifest.dart` 的解析实现一致。
- 与 release 流程接上：发布时这两个文件与 APK 一起出现在 GitHub release assets。
- 私钥不进仓库、不进构建产物、不出现在日志。

### R3. 下载 host 白名单补齐

- `kUpdateDownloadHosts` 增加 `release-assets.githubusercontent.com`。
- **必须实测确认**：GitHub release asset 下载的完整重定向链，以及重定向后的 URL 是否
  仍满足 `isStrictUpdateAssetUrl` 的其余条件——特别是 `url.path.endsWith('.apk')`
  这一条。若重定向后的路径不以 `.apk` 结尾，只加 host 是不够的。
- 不放宽为通配符或后缀匹配。当前是精确 host 集合，保持这个强度。

### R4. 不扩范围

- 不引入应用内更新以外的分发渠道。
- 不为「以后可能」增加密钥轮换机制、多公钥支持、回滚保护等基础设施（parent R6）。
  单公钥 + 单算法即可，需要轮换时再做。

### R5. per-ABI 多资产 manifest（2026-09-07 由 09-02 的 D-1～D-3 决策并入）

背景：`09-02-performance-size-maintainability-refactor` 实测 release APK 88.6 MB 中 85.9 MB 是三套
ABI 的 native 库，用户已决定 release **只发布 per-ABI 拆分 APK（arm64-v8a + armeabi-v7a）**、
**保留 Flutter 的 `1000 × ABI` versionCode 偏移**（arm64 = `2000 + n`，armeabi-v7a = `1000 + n`），
并要求 manifest 从一开始就按多资产实现，避免发布过单资产 schema 后再迁移。设计草案见
`../09-02-performance-size-maintainability-refactor/design.md` §4.1–4.2。

- manifest `schema` 为 2：顶层 `version`（semver）、`versionCode`（不含 ABI 偏移的基数）、
  `packageName`、`signingCertificateSha256`；`assets` 为数组，每项 `abi`、`url`、`size`、`sha256`、
  `versionCode`（含偏移的实际值）。签名对象仍是整个 manifest 文件字节。
- `tool/update_release.py generate` 接受多个 APK（每个带 ABI），为每个资产计算 `size`/`sha256`，
  URL 命名 `pixiv-func-v<version>-github-<abi>.apk`；`isStrictUpdateManifestAssetUrl` 与 200 MiB 上限不变。
- Kotlin `platformInfo` 增加 `supportedAbis`（`Build.SUPPORTED_ABIS`）；Dart 端选择第一个匹配 ABI 的
  资产，无匹配时给出可诊断的失败原因（不静默 `up_to_date`）。
- 版本判定：先比 semver `version`，相等时比 `manifest.versionCode` 与
  `installed.longVersionCode % 1000`；不使用 `-Pforce-version-code-ignoring-abi`。
- `release.yml` 构建 `--split-per-abi --target-platform android-arm64,android-arm`，上传两个 APK +
  manifest + sig；`ci.yml` 的 release 检查遍历两个 APK 验签。`--obfuscate --split-debug-info` 的
  符号目录归档由 09-02 child B 负责。
- 验收补充：API 29 与高版本真机各完成一次**按 ABI 选择资产**的真实自更新；manifest 缺少本机 ABI
  时的负向用例可见原因。

## 已确认的发布决策

用户于 2026-09-02 已一次性确认以下方案：

| # | 决策 | 说明 |
|---|---|---|
| D-1 | APK 签名 keystore | agent 在仓库外生成并以 `0700/0600` 保存；用户负责离线加密备份。 |
| D-2 | updater manifest 签名密钥 | 与 APK keystore 独立，由 agent 在同一外部 release 目录生成和保护。 |
| D-3 | 验签算法 | `SHA256withECDSA` P-256，API 29 provider fixture 必须通过。 |
| D-4 | CI/release | 建 GitHub Actions；release workflow 手工触发，本轮允许配置 Secrets 和 draft Release。 |
| D-5 | `fdroid` flavor | 保留并继续构建，维持 store-managed/关闭 updater；当前不公开发布 F-Droid。 |

## Acceptance Criteria

- [ ] release 构建使用正式 keystore；构建产物的签名证书指纹可查且与 debug 不同。
- [ ] 缺少密钥时构建行为明确（失败或显式标记），不会静默退回 debug 签名。
- [ ] 存在可复现的 manifest 生成 + 签名脚本，产物格式与 `update_manifest.dart` 的解析一致。
- [ ] 私钥不出现在仓库、构建产物与日志中。
- [ ] **API 29 真机上完成一次真实的自更新**（下载 → 验签 → 安装），这是 R2 的核心验收。
- [ ] 高版本 Android 真机上同样完成一次。
- [ ] 验签失败时能区分未配置 / 算法不可用 / 签名不匹配三种原因。
- [ ] 篡改 manifest 或签名后，更新被拒绝且原因可见（负向用例）。
- [ ] GitHub release asset 的完整重定向链在白名单内可通过；实测记录写进 research。
- [ ] `flutter analyze` 与 `flutter test` 通过。

## Open Questions

- 只剩实现阶段的 API 29 provider、签名 fixture 和真实 GitHub redirect 事实复核；产品决策已确认。
- `fdroid` flavor 的空实现按已确认的 store-managed 契约复核，不接入 GitHub updater。

## Notes

- 上级需求与跨 child 约束见 `../09-01-func-1-0-hardening/prd.md`。
- 复核文档的 G1 是本 child 的必读项（`research/audit-verification.md`）。
- 排序：本 child 的**代码改动**与其它 child 不冲突，可随时插入；密钥准备与 CI/draft Release
  验证按用户授权的 `L2` 执行，但不转为公开正式 Release。
