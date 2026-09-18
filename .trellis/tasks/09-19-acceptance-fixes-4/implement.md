# implement — acceptance-fixes-4

一个勾一个 commit，提交信息即条目文本。每步后跑 `flutter analyze` + 相关测试。

- [x] 1. fix(motion): PressScale/StaggeredEntrance 感知 TickerMode，转场冻结时直渲终态；feed 入场动画 once 语义防重播（P1/P7/P8）
- [x] 2. fix(profile): 返回按钮收敛为单一常驻组件统一展开/折叠几何；canPop 首次求值后缓存，返回转场中随页滑出（P3/P4）
- [x] 3. fix(profile): 资料编辑换 App API v1/user/profile/edit+presets，脱离 www cookie 依赖；_StatusBody loading 文案修正（B）
- [x] 4. fix(novel): 推荐翻页 _validateCursor 放行 filter 等客户端身份参数；首页请求补 include_privacy_policy/include_ranking_novels（D）
- [ ] 5. feat(share): share_plus 统一 SharePayload 契约；详情/个人页接系统分享，失败 fallback 剪贴板（E）
- [ ] 6. feat(novel): 沉浸式阅读器壳——全屏 Stack+chrome toggle+back 优先级；元数据撤出+caption HTML 渲染（C1）
- [ ] 7. feat(novel): 阅读设置弹层（字号/行距/主题持久化）+页脚 tip 行+charIndex 进度恢复（C2）
- [ ] 8. chore(task): 收尾簿记（勾选、journal、归档）
