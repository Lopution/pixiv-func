# 动效体系补齐

## Goal

动效从"只有 hero 图片转场"补齐为统一语法：页面转场、列表进场、反馈动效、sheet/菜单动效，全部 token 化、可降级。

## Requirements

- 扩展 `app/motion/`：页面转场构建器（go_router pageBuilder 统一入口）、列表项 stagger 进场、卡片按压缩放/反馈、sheet/长按菜单弹出曲线、查看器页间过渡
- 所有时长/曲线/距离进 `motion_tokens`；页面/组件禁止动效字面量
- "减少动态效果"设置项 + 系统 disableAnimations 合并判定（单 owner）
- 存量动效审计：逐处改为 token 引用；不改动 hero 转场与 viewer 手势的已验证行为

## Acceptance Criteria

- [ ] 主路径（打开详情/进查看器/弹 sheet/长按菜单/列表刷新）有连贯一致的动效语法
- [ ] 降级开关生效即关闭非必要动效；无动画字面量残留
- [ ] widget 测试断言 token 使用与降级；不做逐帧断言

## References

- 本仓：`lib/app/motion/`（hero_transition/drag_to_dismiss/motion_tokens）
- Shaft/pixes/freepiv 的转场与按压反馈观感（手动体验对比）

## Dependencies

建议在 card-quick-actions/mute-system 之前落地语法，避免返工（见父 implement 软依赖）。
