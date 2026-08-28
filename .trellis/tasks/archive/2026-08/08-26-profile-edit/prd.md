# 实现本人资料编辑

## Goal

在不读取用户密码或绕过Web安全的前提下，复刻beta56当前账号资料编辑表单/图片流程，并在成功后同步AccountStore和UserStore；资料图片、API/Web 提交必须复用大陆无外部代理目标的 shared `NetworkAccessPolicy`，不得自行添加网络 fallback。

## Confirmed Facts

- beta56在lib/pages/user/me_settings/profile及web/workspace下包含资料编辑、图片选择和网页辅助实现。
- Profile API或Web表单具有时效性，必须在实现开始时核验当前可支持字段和提交方式。
- User/Profile leaf已提供当前账号UserStore和header状态。

## Dependencies

- 08-26-user-profile-follow、08-26-secure-account-store、08-26-android-platform-parity 与 P0 `08-26-restricted-compat-network` 完成。

## Requirements

- R1: 从beta56固定commit提取可见字段、顺序、说明、校验、头像/背景选择和成功/失败反馈；不主动新增字段。
- R2: 开始实现时核验当前官方API或受支持Web编辑流程；不得抓取密码、注入读取凭据、绕过证书或伪造成功。
- R3: 用 typed `ProfileCapabilities` 描述服务端当前允许字段、是否需当前密码和验证流程；用 `ProfileDraft` + `ProfilePatch` 区分 base、dirty fields、图片变更和服务端错误，只发送真实变化字段。
- R4: 文本字段做长度、格式、枚举和服务端约束验证，错误定位到对应控件且保留用户输入。
- R5: 头像/背景通过受控picker、MIME/像素/大小检查和有界处理，上传/裁剪可取消并清理临时数据。
- R6: 提交结果区分 `confirmed`、`verificationPending` 与 field errors；只有 confirmed authoritative metadata 才原子更新 UserStore/AccountStore，等待邮件验证的字段不得提前显示为已生效。
- R7: Web helper若必要只能单向填写用户已明确输入的非秘密表单值，不读取密码/cookie DOM，且TLS错误失败。
- R8: 账号切换或页面dispose取消当前draft/upload，禁止把A账号编辑提交给B账号。

## Acceptance Criteria

- [x] 表单字段、顺序、默认值和图片入口与beta56对应，unsupported字段明确处理而非空操作。
- [x] 客户端/服务端校验、未保存离开、提交loading/cancel/error/retry均可测试。
- [x] 图片处理内存有界，恶意/超大格式拒绝，临时文件在所有终态清理。
- [x] 成功后Profile/header/account card同步新资料；失败/取消保持旧confirmed状态（confirmed store bridge 已由单元测试覆盖）。
- [x] 账号切换竞态不能错账号提交；日志/DOM bridge/普通存储不含密码、token、cookie或原图bytes。
- [x] capability/dirty-field 组合矩阵验证只发送预期字段；当前密码只存在于一次 submit 的短生命周期内，不进入 draft persistence、prefill 或错误对象。
- [ ] API/Web 图片与提交均消费 shared route policy；系统 proxy/VPN 关闭时的真实受控资料修改记录 route/failure，不把第三方 Web helper 或固定 host 当兼容路径。
- [ ] analyze、全量test、debug build和真实账号受控资料修改/恢复验证通过。

### Evidence boundary

最新 MuMu API 35 运行已确认真实登录账号的个人资料读取和编辑页入口，
但当前没有已核验的官方资料修改 API 或允许的 Web adapter。编辑页因此展示
明确的 unavailable 原因并禁用 Save；这是真实产品能力 blocker，不是账号登录
链路 blocker。没有在真实账号上修改资料，因为本任务没有指定无害测试值，也没有
得到针对该账号的实时变更授权。详见 `research/implementation-evidence.md`。

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
