# 一次性产品与发布决策（2026-09-02）

用户确认：全部采用本轮推荐方案，并授权远程级别 `L2`。以下决策对 09-01 child 及其后
的 09-02 重构生效；实现阶段不得再次就同一事项停下来询问。

## 产品

- 下载位置：统一“保存位置”入口，先选相册或文件夹。相册默认 `Pictures/PixivFunc`，
  自定义相册名只允许规范化名称；文件夹只用 Android SAF 系统目录选择器并持久化 tree URI
  权限，不接受裸路径、不自制文件浏览器。
- 网络设置：普通页只保留网络模式（Automatic / Direct only）、网络诊断、高级设置。高级
  只保留 DoH endpoint、ECH front host、恢复默认值；不暴露实现名词、全局 insecure no-SNI
  或 native login intercept。
- 质量：预览 medium / large / original，默认 medium；查看 medium / large / original，默认
  original。旧 bool 按既定迁移规则读取。
- 文件命名：预设优先，自定义模板只允许 `{artist}`、`{title}`、`{id}`、`{page}`、`{ext}`、
  可选 `{date}`；实时预览、非法字符清理、长度裁剪和多页编号；不允许脚本、正则、条件或
  模板目录语法。
- 本地过滤：接入 Recommended、Ranking、Search、用户作品列表和 widget；不接入收藏、历史、
  详情；过滤后有限续拉直到填满一屏，设置变更即时生效。
- 翻译：四条路径为关闭、百度、通用 OpenAI-compatible LLM、Google 兼容项。新安装默认关闭，
  旧版本已明确保存 Google 的用户继续兼容 Google；不自动 provider fallback。LLM 使用固定
  翻译 prompt，不开放 prompt/模型高级参数。百度实名认证与额度提示写进设置引导。
- 翻译凭据：新增独立安全存储 namespace，不复用 Pixiv 账号 `CredentialStore`；百度 AppID/
  secret、LLM API key 不进入 `AppSettings` JSON、账号迁移剪贴板、日志或崩溃字段。
- 反向搜图：采用 PixEz 式零配置 SauceNAO WebView/HTML 路径；用户确认搜索后才上传，Pixiv
  链接回 Func 原生页面，其他来源交给浏览器；不建结构化 provider registry、mirror 或要求
  用户填写 key。匿名服务不可用时显示明确失败，不伪装成空成功。
- 网络性能：测量能力放入网络诊断的高级/深藏入口，不对普通用户展示；只有实测数据支持时
  才改 GET/image 路线或公开图片 CDN 内部 transport，API/OAuth 始终 strict。

## 网络性能修订（2026-09-03，用户追加确认）

- 撤销「只有实测数据支持才改 GET/image 路线或公开图片 CDN 内部 transport，API/OAuth 始终
  strict」的限制：直接按 PixEz 方案落地，无需真机数据。
- 具体：业务请求即路由尝试（attempt-first，取消独立 probe）；PixEz 兼容档（空 SNI + 无证书
  校验 + 持久化/内置地址）为所有已知 Pixiv 主机的第一候选；POST/一次性 exchange 仅在
  「能证明请求未送达」的失败（DNS/connect/reset/TLS 握手）下才前进到下一档，已送达结果
  永不重发；ECH config 查询改走境内可达的阿里 DoH（`lookup_alidns_https_ech` 同款）；
  `insecureNoSniEnabled` 在生产恒开启（不再作为用户显式开关暴露）。

## 发布与远程操作

- APK release keystore 与 updater manifest signing key 为两把独立密钥，由 agent 在仓库外
  生成并以 `0700/0600` 权限保存；用户负责离线加密备份。任何密钥不进 git、APK、artifact、
  日志或聊天内容。
- updater 使用 `SHA256withECDSA` P-256，兼容 API 29；删除 Ed25519-only 与固定 64 字节假设。
- 建 GitHub Actions：常规检查可自动运行，release workflow 只手工触发；本轮 `L2` 允许配置
  GitHub Secrets、创建 draft Release、验证真实 CDN redirect 和 updater，不自动发布公开正式
  Release。
- 保留 `fdroid` flavor 并继续构建，维持 store-managed、关闭应用内 updater；当前不申请或发布
  F-Droid 上架版本。

## 时序与范围

- 先完成所有 09-01 child，再进入 09-02 全项目性能/体积/可维护性重构。
- 09-02 不做前置性能/体积基线，不写未经测量的绝对指标；只做有调用链、消费者和构建证据的
  可回滚阶段。
- `L2` 不扩大到删除用户数据、重写 git 历史、强制 push 或公开正式发布。
