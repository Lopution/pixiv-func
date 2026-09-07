# 执行计划：正式签名、更新清单验签与 GitHub 资产下载

## 开工条件

- [x] D-1 APK keystore 生成与保管方案已确认，真实密码通过外部 secret 注入。（`tool/RELEASE.md` SOP：`~/.pixivfunc-release`；`release.yml` 从 Secrets 解码）
- [x] D-2 独立 manifest signing key 方案已确认；不与 APK keystore 共用。（`update_release.py genkey`，EC P-256 独立文件）
- [x] D-3 已在 API 29 provider 上确认算法与签名编码；保留可运行 fixture。（`SHA256withECDSA` / DER；离线 fixture 在 `updater_manifest_test.dart` + `update_release.py self-test`；API 29 真机 provider 运行待用户）
- [x] D-4 已决定手工 release 或 GitHub Actions；D-5 已决定 `fdroid` 是否发布。（D-4 GitHub Actions 手工触发 + draft；D-5 保留 fdroid、store-managed、不公开发布）
- [x] `design.md` 的 canonical manifest bytes、错误 code、host/path 合同完成 review。

## 阶段 0：离线合同与密钥边界

- [x] D-1~D-5、密钥输入方式、禁止写入位置记录于 `tool/RELEASE.md`（密钥不进入仓库）。
- [x] manifest canonicalization（UTF-8、无尾换行、fixed separators）、SHA256withECDSA 签名编码、
      公钥 DER base64 与错误 code fixture 已固定（`tool/update_release.py` 自测
      sign→verify→tamper 通过）。
- [x] `github`/`fdroid` capability、安装权限与 updater 边界已明确（fdroid store-managed）。

## 阶段 1：API 29 verifier

- [x] GitHub flavor verifier 改为 `SHA256withECDSA`（P-256），删除固定 64 字节判断。
- [x] 返回可诊断 code（`public_key_missing` / `algorithm_unavailable` /
      `signature_mismatch` / `message_missing` / `signature_missing`）；Dart 侧新增
      `UpdateManifestVerification` 模型透传，service 不落日志。
- [x] Flutter contract tests：valid / tampered signature / 签名不匹配（`updater_manifest_test`、
      `updater_download_test`）；API 29 provider 实测待真机阶段。
- [x] 阶段门：updater 相关 `flutter test` 通过。

## 阶段 2：manifest 生成与签名

- [x] `tool/update_release.py`（generate/genkey/pubkey）输出与 `update_manifest.dart` 解析
      严格一致的 manifest + DER 签名；`tool/RELEASE.md` 记录用法。
- [x] 私钥只从文件路径读取；脚本不打印私钥；输出半成品前先写签名、拒绝覆盖已有资产。
- [x] 自测通过：sign→verify roundtrip、tamper 负向、pubkey DER/emitted 一致、sha256/size 一致。
- [x] 阶段门：无私钥环境明确失败；构建产物与日志无私钥（CI 有扫描步骤）。

## 阶段 3：Gradle signing 与 release wiring

- [x] `github` release 使用 `PIXIV_RELEASE_KEYSTORE*` 正式 keystore；缺材料时显式
      debug 标记警告 + `RELEASE_SIGNED_OFFICIALLY=false`（不再静默）；CI release
      workflow 缺 secret 直接失败。
- [x] 已建 `.github/workflows/ci.yml`（analyze+test）与 `release.yml`（手工触发：keystore
      解码、构建、指纹检查、manifest 签名、draft Release 上传三资产、密钥泄漏扫描）。
- [x] `fdroid` 保持 store-managed/no updater，不共享 GitHub 私钥。
- [x] 阶段门：真实 release 证书指纹与两 flavor 构建行为待发布阶段验证。

## 阶段 4：GitHub redirect 与真实 updater

- [x] `release-assets.githubusercontent.com` 已加入精确 host allowlist（`download_request.dart`）。
- [x] 发布测试资产，记录完整 redirect chain 与 path/`.apk` 后缀实测到
      `research/github-redirect.md`（2026-09-07 用临时 prerelease 探针测完即删：
      `github.com` 302 → `release-assets.githubusercontent.com` 200，单跳，CDN path 无
      `.apk` 后缀；`isStrictUpdateAssetUrl` 默认拒绝非 `.apk`/非白名单 host，不放松）。
- [ ] API 29 与高版本真机各执行一次下载、验签、安装；负向用例（**用户真机项**：需正式 keystore + CI Secrets 出一份 draft release 后执行；离线负向用例已由 self-test / `signature_mismatch` 单测覆盖）。
- [x] 阶段门：allowlist 单测与 updater 相关测试通过（真实资产待发布阶段）。

## 最终验证

- [x] `flutter analyze`（2026-09-08 final check：No issues found）
- [x] `flutter test`（658 passed；`oauth_service_test` 一次 2 分钟 loopback 超时属已知 flake，单独重跑 16/16 通过）
- [x] `git diff --check`
- [x] `flutter build apk --release --flavor github`（split；无材料无 flag → `:app:verifyGithubReleaseSigning` 失败并给出四个属性名，不产 APK；`-PPIXIV_ALLOW_DEBUG_RELEASE_SIGNING=true` → 打印 `!!! RELEASE SIGNING: DEBUG KEYS`，`apksigner --print-certs` 为 `CN=Android Debug`；正式指纹待 Secrets）
- [x] `flutter build apk --release --flavor fdroid`（split，未签名 = store-managed 预期）
- [x] 搜索 git diff、APK、build 输出和 CI 日志，确认无私钥、密码、token。（分支 `git log -p` / `git diff` 正则、`rg PRIVATE KEY`、`git ls-files` 均干净；`release.yml` 「Scan for leaked secrets」步骤在；根 `.gitignore` 补 `*.jks/*.keystore/*.p12/*.pem`）
- [ ] 用户真机完成 API 29、高版本自更新和 F-Droid/store-managed 行为确认。（**用户项**）

## 回滚点

1. verifier/error code；2. signer/generator；3. Gradle signing/CI；4. allowlist/真实 updater。
任一密钥、算法、redirect 或 flavor 边界未满足时只回滚当前阶段，保持 updater 拒绝未知资产，
不恢复 debug signing 作为发布方案。
