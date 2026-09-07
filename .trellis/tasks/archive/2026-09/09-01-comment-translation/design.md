# 技术设计：百度与通用 LLM 评论翻译

设计基线：HEAD `409df51`。现有接口、Google 实现和非持久化 overlay 保留；本 child 增加两种
实现、设置凭据入口和错误分类，不建立动态 provider 框架。

## 1. 前置事实与产品决策

- 实现前必须复核百度当前额度/实名认证、腾讯云 TextTranslate 是否仍可用、Google gtx 是否
  可用。外部事实若推翻 D2，必须回到用户重新选择，不得静默换服务商。
- 默认 provider 已确认：新安装默认关闭，已有配置继续兼容 google；用户主动切换后才使用
  百度/LLM。该策略不会自动 provider fallback。
- LLM prompt 固定为短指令 + 评论文本，不开放模板编辑或模型高级参数，避免本 child 变成
  通用 LLM 客户端。

## 2. 凭据边界

当前 `CredentialStore` 是按 accountId 存 Pixiv `Credential` 的强类型存储，直接复用会把账号
token schema 与翻译 secret 混在一起。推荐新增同级 `SecureSettingsSecretStore`（底层仍使用
`flutter_secure_storage`，独立 key namespace），`SecretSettingRef` 只保存非机密 key；同步
修正文档注释，保持 `AppSettings.toJson()` 和 `accountTransferService` payload 不含 secret。

凭据记录按 provider 保存：百度 AppID/secret，LLM base URL/model/API key。写入、读取、清除均
注入接口，测试使用 fake；日志、异常和诊断只输出 provider/code，不输出值或请求正文。

## 3. Provider 与错误合同

沿用 `CommentTranslationService` + `_ConfiguredCommentTranslationService` 的显式 switch，
新增 `BaiduCommentTranslationService` 与 `OpenAiCompatibleCommentTranslationService`。
transport 只发送用户点击的单条文本与目标语言。

将 `CommentTranslationError` 的字符串 reason 收敛为可枚举 code（未配置/凭据无效/限额或频率/
网络/响应格式/空文本），UI 文案在 i18n 层映射；Google 的现有行为通过适配映射保持兼容。

- 百度：按官方签名规则生成请求，凭据仅在内存中组装；HTTP/业务错误按 code 映射，实名认证
  与额度提示在设置引导中出现。
- LLM：校验 base URL 为 HTTPS、host/path 有界且无 userinfo/fragment；只调用兼容的
  chat-completions 形状，限制响应大小并检查文本字段，禁止把账号/作品 ID 放进 body。
- 关闭：继续抛 `CommentTranslationUnavailable`，UI 不发请求。

## 4. 文件责任

| 责任 | 主要文件 |
|---|---|
| provider/错误 | `lib/core/comments/comment_translation.dart` 及同目录实现 |
| secret 存储 | `lib/core/auth/` 或 `lib/core/settings/` 新增独立 store；不改账号 Credential JSON |
| 设置模型/UI | `app_settings.dart`、`settings_controller.dart`、`features/settings/settings_page.dart` |
| 文案 | `lib/core/i18n/replica_strings.dart` |
| 迁移边界 | `lib/core/auth/account_transfer_service.dart`（只补负向测试，不加 secret） |
| 测试 | `test/comment_translation_test.dart`、`settings_test.dart`、account transfer 测试 |
| 研究 | `research/provider-facts.md`（来源、日期、结论，不写密钥） |

## 5. 兼容与回滚

- 旧 `translateIndex` 数值必须稳定迁移；新枚举只能追加明确 code，不重排现有值。
- provider 切换不持久化评论原文/译文；清除凭据后立即回到未配置态。
- 每阶段可单独回滚：secret store、provider、UI/错误文案；失败时不自动降级到另一 provider。
- 真实百度/LLM 请求与凭据只在真机验收使用，不把任何测试 key 写入仓库或日志。

## 6. 已确认的产品选择

用户于 2026-09-02 确认：使用独立安全 namespace 存储翻译 secret；新安装默认关闭、旧 Google
配置兼容；固定 prompt；额度/凭据/网络错误不自动切换 provider。完整决策记录见 parent 的
`research/decision-record-2026-09-02.md`。
