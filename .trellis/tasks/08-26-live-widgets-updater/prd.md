# 复刻 Live、Widgets 与 Updater

## Goal

作为中间父任务，协调Live播放器、Android Home Widgets和Updater flavors三个高时效叶子任务，并隔离它们的网络、后台和权限风险。

## Confirmed Facts

- 该范围已拆为live-player、android-home-widgets、updater-flavors。
- beta56 Live使用旧fixed-IP local proxy，Widgets原生读取旧Flutter账号数据，Updater基础Manifest全局申请安装权限；都不能直接迁移。
- 三项均在第一条链和主要内容功能后实施。

## Dependencies

- 08-26-android-platform-parity、profile-follow-social、downloads-ugoira-media和正常网络主链完成。

## Requirements

- R1: 本中间父任务不直接实现业务代码，维护三个叶子任务的实时research、权限/credential和lifecycle隔离。
- R2: Live不得恢复fixed-IP proxy或添加chat；Widgets不得明文复制token；Updater安装权限仅GitHub flavor。
- R3: 三个叶子分别规划审批、实现、验证、提交和归档；任何当前API/平台能力失效保持blocker。
- R4: 完成前审计后台Worker/player/updater不会互相持有无界资源或污染基础Manifest。
- R5: 运行账号切换、网络失败、进程重启、flavor build和深链/Widget点击集成回归。

## Acceptance Criteria

- [ ] 三个叶子任务均归档并有时效来源/设备证据。
- [ ] Live可用时符合beta56player UX且无fixed-IP/chat；不可用时诚实报告。
- [ ] Widgets在无账号/锁定/失效token时安全降级且不泄密。
- [ ] GitHub/F-Droid merged manifests权限严格隔离，F-Droid无self updater/安装权限。
- [ ] 全量analyze/test、各flavor build和真机后台/播放/更新集成通过。

## Out of Scope

- 实际GitHub/F-Droid发布。
- Live弹幕。
- 任意后台常驻服务。

## Risks and Deferred Items

- Live API和Android后台/updater政策变化最快，必须在各叶子开始当天核验。

## Source Anchors

- beta56 lib/pages/live、Android appwidget、lib/app/updater
- 三个叶子task artifacts与父PRD

## Open Questions

无阻塞性产品决策。时效性技术事实按Requirements中的research gate在实现开始时重新核验，不改变本任务行为边界。
