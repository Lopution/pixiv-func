# GitHub release asset redirect chain (R3 实测)

测量日期：2026-09-07（UTC+8 2026-09-08 00:08）。仓库当时没有任何 release
（`gh release list` 为空），所以用一个临时 prerelease 做探针，测完即删：

```bash
# 161 字节的 zip，只含 README.txt；文件名按 R5 的资产命名规则
gh release create redirect-probe-2026-09-07 --prerelease --target main \
  pixiv-func-v0.0.0-github-arm64-v8a.apk
curl -sSL -A okhttp/4.12.0 -D - -o out.bin \
  https://github.com/Lopution/Pixiv-func/releases/download/redirect-probe-2026-09-07/pixiv-func-v0.0.0-github-arm64-v8a.apk
gh release delete redirect-probe-2026-09-07 --yes --cleanup-tag
```

`curl` 与 `okhttp/4.12.0` 两个 UA 结果相同，body 与上传文件逐字节一致。

## 链路

| 跳 | 状态 | host | path | 说明 |
|---|---|---|---|---|
| 0 | `302` | `github.com` | `/Lopution/Pixiv-func/releases/download/<tag>/<file>.apk` | `content-length: 0`；manifest 里写的就是这个 URL |
| 1 | `200` | `release-assets.githubusercontent.com` | `/github-production-release-asset/1347042509/<uuid>` | `content-type: application/vnd.android.package-archive`，`content-disposition: attachment; filename=<file>.apk` |

- 只有 **一跳**。没有经过 `objects.githubusercontent.com` 或
  `github-releases.githubusercontent.com`；这两个 host 留在
  `kUpdateCdnHosts`（`lib/core/download/download_request.dart`）作为历史/回退路径，
  不删。
- 跳 1 的 path 不以 `.apk` 结尾（`1347042509` 是仓库 id，后面是资产 uuid），与
  `isStrictUpdateRedirectUrl` 的设计一致：只有跳 0 要求 `.apk` 后缀，CDN 跳只校验
  精确 host + https。
- 跳 1 的 URL 是带签名的 Azure Blob SAS + JWT（`se=` 过期时间约 1 小时）。
  下载必须在拿到 302 后立刻跟进；不能把 CDN URL 落盘复用。
- 删除 release 后，同一 CDN URL 在边缘缓存有效期内仍返回 200；`github.com` 端对不存在的
  tag 直接 `404`（无重定向）。

## 大小写：仓库名与 path 校验

GitHub 上仓库的规范名是 `Lopution/pixiv-func`（`gh repo view --json nameWithOwner`），
而代码与 manifest 使用 `Lopution/Pixiv-func`。实测 `github.com` 对
`/Lopution/Pixiv-func/releases/download/...` **直接 302 到 CDN，不先规范化大小写**，
所以 `isStrictUpdateManifestAssetUrl` 中区分大小写的
`url.path.startsWith('/Lopution/Pixiv-func/releases/download/')` 成立。

约束：生成器（`tool/update_release.py`）与 Dart 校验必须使用**同一个字面量**
`Lopution/Pixiv-func`；任何一侧改成小写都会让另一侧拒绝。09-07-release-size-per-abi
的 B4 按 R5 由 version + abi 推导 URL 时沿用这个字面量。

## 结论

- R3 的 allowlist（`github.com` + 三个 `*.githubusercontent.com`）覆盖实测链路，且
  `release-assets.githubusercontent.com` 正是当前唯一的 CDN 跳。
- 这只证明 URL 合同；真机上「下载 → 验签 → 安装」仍是 R2 的待用户项。
