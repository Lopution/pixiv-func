# 实现本人资料编辑 — Design

## Objective

在不读取用户密码或绕过Web安全的前提下，复刻beta56当前账号资料编辑表单/图片流程，并在成功后同步AccountStore和UserStore。

## Architecture and Boundaries

- ProfileEditRepository暴露capabilities/loadDraft/submit；实现按当前安全通道选择API或受限Web adapter。
- ProfileDraftController拥有baseRevision、dirty fields、validation、upload states和accountId guard。
- ImagePreprocessor复用ReverseImage的受控文件基础但使用Profile尺寸/裁剪策略。
- 成功response经过UserMapper后一次性commit到UserStore/AccountStore。

## Data Flow

current account/profile → load capabilities+draft → edit/validate/image preprocess → submit(account+baseRevision) → authoritative response → atomic stores update；error/cancel保持draft。

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

