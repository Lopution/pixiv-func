# 09-19-network-image-l0 — 网络与图片 L0 修复

## Goal

修复用户正在踩的三类真实缺陷：切换图片源后加载失败、图片流传输中途僵死永不恢复、`decodeWidthFor` 注释与实现不符。附带 pixiv.cat 可达性提示。

## In Scope

1. **镜像路由组隔离**：非 pximg 图片 host（预设/自定义镜像）不再与官方 CDN 共享 `image` 组路由偏好——阻断 pximg 学到的 `noSni`/`ech` 组偏好泄漏到镜像导致证书终态失败。
2. **镜像档位补全**：镜像 fallback 档补 `noSni`（空 SNI + 证书校验保持开启）。空 SNI 下的证书不匹配语义是"默认 vhost 不承载该域"而非 MITM → 允许梯子继续下一档（仅 `noSni` 档放宽，其余档维持终态）。
3. **流式空闲超时（B1）**：image purpose 池化客户端包空闲守卫——send() 的 headers 预算（超时 → 梯子换档）+ body 逐块空闲超时（超时 → 流报错 → 图片可见错误态 / 下载走 Range 续传重试）。覆盖图片缓存与下载两条路径。
4. **`decodeWidthFor` 注释修正**：注释声称的 1.5x cap 不存在且是被故意移除的（高 DPR 下半分辨率发糊）；改注释为真实语义（解码到显示像素宽度、源宽封顶）。
5. **pixiv.cat 警示**：图片源预设项标注大陆网络通常不可达。

## Out of Scope

- 冷启动竞速、路由持久化、connectivity 失效（任务 2）
- `auto` 图片源、viewer 渐进显示、下载缓存复用（任务 2）
- 设置页语义收敛（任务 2）
- insecureNoSni 对镜像开放（任意用户自定义域不关证书校验，维持现状）

## Acceptance Criteria

- 切换镜像后首个请求不再因组偏好泄漏走非法档位而直接失败
- 镜像 host 的 fallback 含 `noSni`，且 `noSni` 档证书不匹配时梯子继续而非终态
- 图片流 headers 超时 → 梯子换档；body 空闲超时 → 可见错误（不再无限转圈）
- 下载中途僵死 → 报错后可 Range 续传
- pixiv.cat 预设项有不可达提示
- 新增/更新单测覆盖以上行为；`flutter analyze` 零 issue
