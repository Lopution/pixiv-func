# 第三方客户端对比记录

调研时间：2026-08-31。源码来自公开仓库的浅克隆快照；本机代理状态不作为可达性证据。本文用于本任务的设计取舍，不把第三方实现直接复制进本项目。

## 对比表

| 项目 | 可借鉴事实 | 本任务采用 | 明确不采用 |
|---|---|---|---|
| [Pixiv-Shaft](https://github.com/CeuiLiSA/Pixiv-Shaft) `42e0e2cfcc4d3cb5dceab3667aa30e7e868287dd` | API 与图片分离客户端；API 使用 Cronet/QUIC；图片使用独立高并发 HTTP/1.1、DoH/健康检查；网络测试包含每层状态和真实图片下载 | 保持 API/图片目的地分层；增强探测结论和原始层可见性；复用池和健康/失效思想 | Cronet/QUIC、全局 TrustAll、默认第三方图片镜像 |
| [pixez-flutter](https://github.com/Notsfsssf/pixez-flutter) `e45e6b3936df391f802fdcb33ef4fb818ef2e151` | 所有 API/图片共享 Rust/rhttp；ECH HTTPS-RR 配置按 TTL 缓存；质量相关 URL 选择和缓存独立 | 在 Dart resolver 做 ECH TTL + 同时查询合并；首图 URL 在来源/详情间显式传递 | 其兼容档的全局关闭证书校验、固定 IP 作为默认路径、未经本项目验证的 DoH 实现 |
| [Pix-EzViewer](https://github.com/ultranity/Pix-EzViewer) `96a4f4b42df82c679d43c5da9283537ce7531590` | DNS×SNI 正交选择；shared-element 在首图和头像加载成功/失败后启动；列表预加载和缓存策略 | 采用首图+头像预加载后再启动 Hero 的时序；维持本项目严格证书与目的地白名单 | Kotlin/OkHttp 特有 socket/SNI 替换和持久化 IP 表 |

## 直接证据摘要

- Shaft `docs/direct-connect.md` 与 `CronetInterceptor.java`：API 只对精确 Pixiv host allowlist 使用 Cronet，图片走不同的 OkHttp/Glide client；其测试页同时展示 DNS、TCP、TLS、HTTP 和图片下载结果。
- Shaft `docs/image-host.md`：镜像切换是用户显式选择，非官方镜像必须退出官方源站的 no-SNI/固定 IP 策略。
- PixEz `lib/network/network_mode.dart`、`lib/network/pixez_network_settings.dart`：`standard`/`ech`/`compat` 是明确档位；ECH 档保持证书校验，兼容档才使用旧的空 SNI 方案。
- PixEz `lib/er/hoster.dart`、`lib/network/onezero_client.dart`：ECH 前置主机 HTTPS RR 需要可达的 DNS 引导；其部分路径关闭证书校验，不能原样迁移。
- Pix-EzViewer `PictureActivity.kt`、`PictureXAdapter`、`ImageViewAttrAdapter`：调用 `supportPostponeEnterTransition()`，在首图和头像完成（包括失败）后 `supportStartPostponedEnterTransition()`；`PictureXAdapter` 明确使用 `data.meta[0].medium` 作为第一张预览/transition source。

## 对当前设备截图的解释

当前 `network_probe.dart` 在发现系统 DNS 与 DoH 地址无交集时先返回 `dnsPolluted`，即使 ECH 层已收到 HTTP 404；这与 Shaft/PixEz 的“报告可用出口”目标不一致。修复应把 ECH/空 SNI 的已验证传输放在 DNS 差异之前，同时保留 DNS 差异作为诊断字段。

当前详情 `_PageImage` 在多图且 `metaPages` 尚未到达时回退 `imageUrls.medium`，而 `IllustCard` 默认使用 `imageUrls.large`；这解释了灰色 Hero 和打开后低清图。修复应让来源 URL成为详情首图的显式输入，而不是在详情重新猜测。

## 设计边界

1. 不把“某个网络上实测可用”扩展为所有网络的永久结论；所有缓存都绑定 `NetworkRevision`/TTL。
2. HTTP 404/403 证明服务器已响应，但不证明业务授权成功；探测只把它们当作传输到达，业务请求仍按现有 HTTP/auth 错误处理。
3. Pix-EzViewer 的“等待图片完成再启动 shared-element”适合其原生页面，但本项目保留立即点击转场；这里改为启动预热而不等待，详情首帧先创建稳定的头像位置，命中缓存时直接显示真实头像。
