# 建立 Pixiv 网络客户端与 Token 刷新

## Goal

提供所有业务 feature 共用的严格、可取消、可观测 Pixiv 网络层，并保证并发认证失败只触发一次按账号刷新。

## Confirmed Facts

- 当前仓库没有 Dio/HTTP 业务客户端、统一错误或 refresh gate。
- beta56 通过 ApiClient/AuthClient 复用旧 API 包，同时全局 badCertificateCallback=true；后者明确禁止沿用。
- 父 PRD 要求重新核验时效客户端身份，并限制 API/OAuth/next_url host。

## Dependencies

- 08-26-secure-account-store 与 08-26-oauth-pkce-webview-login 完成。

## Requirements

- R1: 集中实现 PixivClientIdentity、动态请求头、API/OAuth base URI 与客户端身份；业务层不得持有常量副本。
- R2: 使用 Dio 或等价可测试 transport，默认系统 DNS、直接 HTTPS、严格系统证书校验；不得配置全局 cleartext、TLS bypass 或 MITM。
- R3: 定义稳定 ApiError 分类，覆盖网络、超时、取消、HTTP、认证、解析、限流和服务端错误，并保留安全诊断信息。
- R4: 按 account ID 维护 single-flight refresh；触发前比较请求旧 token 与当前 token，若已刷新则直接重试。
- R5: 每个原业务请求最多自动重试一次；invalid refresh 将该账号标记 reauthRequired，终止队列且不形成循环。
- R6: 支持请求取消、超时、受控 retry-after 和敏感字段日志脱敏。
- R7: 提供 allowlisted next-page URI 解析器，只接受 Pixiv API host、已知 endpoint 和 query 参数，不直接请求 arbitrary absolute URL。

## Acceptance Criteria

- [ ] 20 个并发请求使用同一失效 token 时恰好发起 1 次 refresh，其余等待同一 Future。
- [ ] 不同账号的 refresh 相互隔离；已有新 token 时不重复刷新；每个业务请求最多重试一次。
- [ ] invalid refresh 令正确账号进入 re-auth，等待请求得到统一错误且无死锁/无限循环。
- [ ] 恶意 next URL、非 HTTPS、未知 host/endpoint/query 被拒绝。
- [ ] 证书失败保持失败；日志与错误对象不含 access/refresh token、cookie 或授权头。
- [ ] flutter analyze、网络单元/并发测试、全量测试与 debug APK 构建通过。

## Out of Scope

- DoH/CONNECT 兼容网络模式。
- 具体推荐/搜索等 endpoint repository。
- 下载队列和媒体解码。

## Risks and Deferred Items

- 客户端身份和 header 算法会变化；必须集中且带 provenance，失效时只改一处并补契约测试。

## Source Anchors

- beta56 lib/app/api/api_client.dart、auth_client.dart、web_api_client.dart、lib/app/http.dart
- 父任务 R5 与账号/OAuth 子任务接口

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
