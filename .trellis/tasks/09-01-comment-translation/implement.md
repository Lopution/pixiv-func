# 执行计划：百度与通用 LLM 评论翻译

## 开工条件

- [x] 事实复核完成（2026-09-02 决策记录）：百度标准版 5 万字符/月 + QPS1（个人认证）、
      高级版 100 万/月 + QPS10（实名认证）；腾讯 TMT 在售（不改变 D2 路径集合）；
      Google gtx 大陆被阻断（保留为显式兼容路径）。
- [x] 用户确认 D2 仍成立：新安装默认关闭、旧 Google 配置兼容，不自动 provider fallback。
- [x] 用户确认独立安全 namespace 存储凭据，LLM 使用固定 prompt，不开放高级参数。
- [ ] `task.py start` 前 `design.md`、`implement.md` 无阻塞 open question。

## 阶段 0：事实与安全契约

- [x] 事实摘要写入 parent decision-record（外部事实复核章节），代码注释引用额度常量。
- [x] provider enum：disabled(1)/baidu(2)/llm(3)/google(0)；secret namespace
      `replica.translation.v1.`（独立于账号 CredentialStore）；toString 不含 secret；
      AppSettings/迁移剪贴板/日志无凭据。
- [x] 脱敏 fixture：MockClient 构造，业务错误码/401/429/超限/超时/malformed 全覆盖。

## 阶段 1：安全凭据与设置模型

- [x] `TranslationCredentialStore`（SecureTranslationCredentialStore + 内存 fake）：
      read/write/delete 与账号 Credential schema 完全隔离。
- [x] 设置页：4 provider 选择；`TranslationCredentialsPage` 百度 AppID/Secret、
      LLM baseUrl/key/model 输入与清除（HTTPS 校验）；额度/实名提示文案。
- [x] `AppSettings` 仅保存 `translateIndex`；secret 只存在 secure storage。
- [x] 新装默认 disabled（`translateIndex` 默认 1）；迁移剪贴板不包含翻译凭据。
- [x] 阶段门：settings_test（默认/迁移/网络）+ comment_translation_test 通过。

## 阶段 2：Provider 实现

- [x] 百度：MD5(appid+q+salt+secret) 签名、x-www-form-urlencoded、业务错误码分类
      （52003/54001/90107 → invalidCredentials；54003/54005/54000/54004 → rateLimited；
      58002 → other）；凭据只在请求作用域。
- [x] LLM：HTTPS only（http:// 拒绝）、固定 system prompt、temperature 0.2、
      仅传文本与目标语言；401/403 → invalidCredentials、429 → rateLimited、
      响应 >256KB → malformed。
- [x] Google 与 Disabled 保留；显式 switch 分发（`ConfiguredCommentTranslationService`），
      无 registry/plugin discovery。
- [x] 五类错误映射：未配置（notConfigured）/凭据无效/限额频率/网络/格式。
- [x] 阶段门：13 项单测覆盖成功/空文本/错误码/429/超时/响应大小/分发。

## 阶段 3：评论 UI 与 i18n

- [x] 设置引导：百度实名/额度与 LLM HTTPS/固定 prompt 文案，四语言齐全。
- [x] overlay 分类：disabled/notConfigured → 设置提示；invalidCredentials → 凭据无效；
      rateLimited → 超限；其余 → 翻译失败；正文/译文不持久化。
- [x] 凭据按请求时读取（store.readXxx per translate），清除后下一次点击即未配置。
- [x] 阶段门：comments_replies_test（Google 回归/UI）+ settings 相关测试通过。

## 最终验证

- [x] `flutter analyze`（No issues）
- [x] `flutter test`（全量 570+）
- [ ] `git diff --check`（提交前统一执行）
- [ ] 真机：大陆网络百度真实翻译；LLM 自定义 endpoint；Google 回归；关闭无网络请求。
- [x] 静态复核：日志/设置 JSON/迁移剪贴板路径不含 secret（toString 断言覆盖）。

## 回滚点

1. secure store/schema；2. provider transport；3. UI/error/i18n。任何外部事实变化只停在
研究阶段，不以 fallback 掩盖产品决策变化。
