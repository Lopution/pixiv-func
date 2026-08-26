# 实现本人资料编辑

## Goal

在不读取用户密码或绕过Web安全的前提下，复刻beta56当前账号资料编辑表单/图片流程，并在成功后同步AccountStore和UserStore。

## Confirmed Facts

- beta56在lib/pages/user/me_settings/profile及web/workspace下包含资料编辑、图片选择和网页辅助实现。
- Profile API或Web表单具有时效性，必须在实现开始时核验当前可支持字段和提交方式。
- User/Profile leaf已提供当前账号UserStore和header状态。

## Dependencies

- 08-26-user-profile-follow、08-26-secure-account-store 与 08-26-android-platform-parity 完成。

## Requirements

- R1: 从beta56固定commit提取可见字段、顺序、说明、校验、头像/背景选择和成功/失败反馈；不主动新增字段。
- R2: 开始实现时核验当前官方API或受支持Web编辑流程；不得抓取密码、注入读取凭据、绕过证书或伪造成功。
- R3: 用typed ProfileDraft区分原值、用户编辑、图片变更和服务端错误；未修改离开按原版提示。
- R4: 文本字段做长度、格式、枚举和服务端约束验证，错误定位到对应控件且保留用户输入。
- R5: 头像/背景通过受控picker、MIME/像素/大小检查和有界处理，上传/裁剪可取消并清理临时数据。
- R6: 提交成功后使用返回的authoritative user metadata原子更新UserStore及AccountStore非秘密metadata；失败不提前修改。
- R7: Web helper若必要只能单向填写用户已明确输入的非秘密表单值，不读取密码/cookie DOM，且TLS错误失败。
- R8: 账号切换或页面dispose取消当前draft/upload，禁止把A账号编辑提交给B账号。

## Acceptance Criteria

- [ ] 表单字段、顺序、默认值和图片入口与beta56对应，unsupported字段明确处理而非空操作。
- [ ] 客户端/服务端校验、未保存离开、提交loading/cancel/error/retry均可测试。
- [ ] 图片处理内存有界，恶意/超大格式拒绝，临时文件在所有终态清理。
- [ ] 成功后Profile/header/account card同步新资料；失败/取消保持旧confirmed状态。
- [ ] 账号切换竞态不能错账号提交；日志/DOM bridge/普通存储不含密码、token、cookie或原图bytes。
- [ ] analyze、全量test、debug build和真实账号受控资料修改/恢复验证通过。

## Out of Scope

- 修改密码/邮箱安全流程，除非beta56明确且当前官方流程支持。
- 发布作品、私信。
- 新增Profile字段或UX redesign。

## Risks and Deferred Items

- 当前Pixiv可能不再允许第三方API资料编辑；若无安全受支持路径，本任务必须标记blocker，不能恢复凭据抓取。

## Source Anchors

- beta56 lib/pages/user/me_settings/profile/*、web/*、workspace/*、lib/pages/image_selector/*
- UserStore/AccountStore与Android WebView/image contracts

## Open Questions

无阻塞性产品决策。时效性技术事实按 Requirements 中的 research gate 在实现开始时重新核验，不改变本任务行为边界。
