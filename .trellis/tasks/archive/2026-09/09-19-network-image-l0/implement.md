# Implementation plan: 09-19-network-image-l0

## Checklist

- [x] 镜像路由组隔离：`_RouteGroup` 增 `imageMirror`，`_routeGroupFor` 按 (purpose, host) 分流，组偏好读取/写入/失效全部带 host
  - `fix(network): 镜像 host 独立路由组——阻断 pximg 组偏好泄漏到镜像的证书终态失败`
- [x] 镜像档位补 `[noSni, dohRealSni]`；`_retryEligible` 允许 `noSni` 档的 certificateMismatch 继续走下一档（空 SNI 不匹配 = 默认 vhost 不承载，非 MITM 信号）
  - `fix(network): 镜像补 noSni 档——空 SNI 证书不匹配改为可降级信号`
- [x] 流式空闲守卫：image purpose 池化客户端包装 send() headers 预算 + body 空闲超时（`_StreamIdleGuardClient`）
  - `fix(network): 图片流式传输空闲超时——headers 超时换档，body 僵死报错不再无限挂起`
- [x] `decodeWidthFor` 注释改为真实语义（显示像素宽度、源宽封顶、1.5x cap 被故意移除的原因）
  - `docs(image): decodeWidthFor 注释对齐实现——像素级解码是刻意的，逻辑 1.5x cap 曾在高 DPR 上发糊`
- [x] pixiv.cat 预设项加大陆不可达 subtitle（四语言 arb + 重新生成）
  - `feat(settings): pixiv.cat 预设标注大陆网络不可达`
- [x] 回归测试：镜像组隔离、镜像 noSni+证书降级、流式空闲守卫、pixiv.cat 提示
  - `test(network): 镜像组隔离/noSni降级/流式守卫回归用例`
- [ ] analyze + 全量测试 + `git diff --check` + PR
  - `chore(task): 09-19-network-image-l0 验证与归档`

## Safe execution rules

- `insecureNoSni` 不对镜像开放；证书校验在所有镜像档位保持开启（noSni 校验仍开）
- certificateMismatch 可降级仅限 `noSni` 档；ech/dohRealSni/direct 维持终态
- 空闲守卫只作用 image purpose；API/OAuth 的非流式超时语义不动
