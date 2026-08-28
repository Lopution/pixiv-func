# 实现本人资料编辑 — Design

## Objective

在不读取用户密码或绕过Web安全的前提下，复刻beta56当前账号资料编辑表单/图片流程，并在成功后同步AccountStore和UserStore。

## Architecture and Boundaries

- ProfileEditRepository暴露 `loadCapabilities/loadDraft/submit(ProfilePatch)`；实现按当前安全通道选择 API 或受限 Web adapter。
- API/Web adapter 的 host、TLS、DNS 和 route 由 shared `NetworkAccessPolicy` 提供；Profile task 不接受自定义 proxy/host，资料图片与 submit 的错误必须保留 route/failure 分类。
- ProfileDraftController拥有baseRevision、capabilities、dirty-field diff、validation、upload states和accountId guard；密码使用单独 ephemeral submit input，不属于可序列化 draft。
- ImagePreprocessor复用ReverseImage的受控文件基础但使用Profile尺寸/裁剪策略。
- typed submit outcome 把 field error、verification pending 与 confirmed response 分开；只有 confirmed response 经过 UserMapper 后一次性 commit 到 UserStore/AccountStore。

## Data Flow

current account/profile → load capabilities+draft → edit/diff/validate/image preprocess → build minimal patch + ephemeral password → submit(account+baseRevision) → confirmed atomic update | verification pending | field error；error/cancel保持非秘密 draft。

## Compatibility, Security, and Migration

- beta56可见表单为准；当前不支持字段显示明确不可用只在用户批准后接受，否则任务blocker。
- 不迁移旧Web密码辅助或credential抓取。

## Important Trade-offs

- 可见行为以beta56为准；内部选择当前Flutter/Android的安全、可测试实现。
- 依赖通过明确接口连接，不为尚未完成的后续任务添加空操作或隐藏mock。
- 时效性外部事实失效时保持明确失败并回到research，不降低安全或验收标准。

## Rollback

- 本任务使用独立提交，只回滚本任务新增接口、实现、配置和测试。
- 不重写历史、不覆盖无关工作树改动；中间父任务不直接回滚独立叶子提交。
