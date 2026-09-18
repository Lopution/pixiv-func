# APK 体积瘦身

## Goal

release 构建开启 R8/shrinkResources + rhttp 裁剪压缩特性 + resConfigs/dex 压缩打包，两 ABI 各省 ~1.4MB 回到上限内。

## Requirements

- 开启 release 的 `minifyEnabled` + `shrinkResources`，keep 规则覆盖 Flutter embedding、本 app Kotlin channel、插件包（flutter_inappwebview/sqflite/secure_storage 等）
- `resConfigs` 只保留 en/ja/ru/zh
- `dex.useLegacyPackaging = true`：dex 回到 deflate 存储（universal APK 默认 Stored 白丢 ~1.3MB）
- rhttp `Cargo.toml` 移除 `brotli`/`deflate`/`zstd` 只留 `gzip`（reqwest 只通告编译进的解码器，无手动 Accept-Encoding，安全）

## Acceptance Criteria

- [x] arm64-v8a split APK ≤ 29,985,336 B（实测 29,118,521，余 867KB）
- [x] armeabi-v7a split APK ≤ 25,891,502 B（实测 25,183,039，余 708KB）
- [x] 签名 github flavor APK 真机冒烟：登录 webview、图片保存/下载、反向搜图、token 读写正常（用户已验收）

## Notes

- 已在 `/root/pixivfunc-arm64-v8a-release.apk` 交付签名包供验收
