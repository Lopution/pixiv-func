# 实现 Pixiv OAuth PKCE WebView 登录 — Design

## Objective

在保留 beta56 Login 可见体验的同时，用严格 TLS 和一次性 PKCE 完成真实 Pixiv 登录并安全写入 AccountStore。

## Architecture and Boundaries

- PkceSessionStore 管理单个 session ID、verifier、createdAt/expiry 和 consumed 状态；纯函数负责 challenge 与 redirect 解析。
- OAuthService 负责 authorize URI、token exchange 和结果映射；WebView page 只转发经验证的导航事件和用户取消。
- LoginController 暴露 idle/loading/web/authenticating/error 状态，并在成功时调用 AccountStore 原子写入。
- WebView 配置最小权限；任何 JS 注入只允许产品明确的单向表单辅助，默认不实现。

## Data Flow

Login tap → create PKCE session → WebView authorize → validate callback → atomically consume verifier → strict token exchange → CredentialStore/AccountStore → Home。

## Compatibility, Security, and Migration

- 普通网络模式优先；Compatibility 模式在独立任务接入相同 OAuthService，不复制登录逻辑。
- WebView/插件升级必须保持 callback parser 和 PKCE 核心为纯 Dart 可测试组件。

## Important Trade-offs

- 可见行为以 beta56 为准；内部选择当前 Flutter/Android 的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏 mock。
- 任何时效性外部事实失效时保持明确失败，并回到任务 research，不放宽 TLS、输入校验或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；涉及持久化或 Android 配置时先验证反向迁移/移除路径。

