# 补强 Novel typed markup 与长文预算

## Goal

补齐 Novel typed markup 的页面、章节、ruby、跳转和图片 token，保留未知标记并为长文本解析/布局增加可取消的分块与工作预算，避免长文卡顿、错页和旧 reader 结果污染当前页面。

## Scope and current facts

- 目标历史任务：`08-26-novel-reader`；当前边界包括 `novel_entity.dart`、`novel_layout.dart`、`novel_reader.dart`、repository 及对应测试。
- 开源审查显示 `newpage`、`chapter`、`jump`、`pixivimage`、`uploadedimage` 等标记不能被普通文本替换；未知标记必须可保留并可观察。
- 本叶子不改变 beta56 的水平阅读交互和字体/主题选择，只补足 parser、layout 和取消边界。

## Requirements

- R1：使用 typed token/AST 表示 text、new page、chapter、ruby、jump（URI/page）、Pixiv image、uploaded image 和 unknown；token 携带原始属性以便诊断和未来兼容。
- R2：未知或格式错误标记不得静默吞掉正文；按显式 fallback token 渲染或展示可观察的 unsupported 状态，禁止把未经验证的 HTML 当默认执行环境。
- R3：长文按有界 chunk/budget 解析和布局，支持取消、yield 和 progress；单次布局不得无界分配 bitmap、widget 或字符串副本。
- R4：reader generation、章节/页面选择和 cancellation 必须贯穿 parse -> layout -> commit；旧 reader 结果不能覆盖当前章节。
- R5：jump/image URI 只接受允许的 scheme/host/标识符，图片加载走共享网络 policy，不把正文 token 变成任意导航或文件访问。

## Acceptance Criteria

- [ ] 固定 fixtures 覆盖 newpage、chapter、ruby、jump、pixivimage、uploadedimage、未知属性、嵌套/损坏标记和 Unicode 文本。
- [ ] parser 保留未知 token 的原始信息；layout 输出页边界与现有 beta56 规则一致，长文在 budget 内分块且可取消。
- [ ] 章节切换或 dispose 后，迟到 parse/layout 结果不会写当前 reader；图片/跳转输入非法时返回可见错误。
- [ ] mapper、token、layout budget、cancellation 和水平 reader 回归测试通过；真实 Novel API/设备证据单独记录。
- [ ] 归档任务无 diff，且 integration release 记录本叶子的 typed-markup evidence。

## Dependencies and Out of Scope

- 依赖：`08-26-novel-reader` 的现有模型和 beta56 交互契约；不依赖新增远程服务。
- 不负责 Novel UI 重做、全文 HTML/CSS 引擎、评论/收藏或任意 URL 浏览能力。
