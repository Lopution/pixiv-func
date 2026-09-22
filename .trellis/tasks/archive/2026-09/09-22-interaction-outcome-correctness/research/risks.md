# W1 风险与决策点

## 需要用户/评审确认的决策点

1. **浏览设置「测试」**：方案 A（改名「应用并测试」，行为不变）vs 方案 B（真测试语义：只写 `customImageSource` + 放宽 `imageMirrorAllowlistProvider`）。B 扩大 image-purpose 直连信任面（未选中 host 也可被 client() 接受）且需新 UI 表达「已测未启用」；**推荐 A**，B 需要显式确认。
2. **login WebView 致命错误**：原地「重新登录」（beginSession 幂等重启）vs 保留 pop 只改名「返回」。推荐原地重启（名称=后果、少一步操作），但它让 WebView 页多一个「开始新会话」的职责。
3. **AppBar 是否加常驻 reload**：对齐 PixEz 模式，非 §4.1 硬性要求；不加则 reload 只在错误卡内可达。
4. **reverse-image ready 态「取消」是否改名**（清除图片 vs 取消）：改名更准但多一组 l10n key。
5. **本地小说越界 offset 落点**：夹取到末页（推荐，最接近用户意图）vs 判失效回开头。
6. **本地小说锚点函数放哪**：页面私有 vs `core/localnovel/` 公共 helper（跨包单测需要公共；选公共则命名/注释按 core 规范）。
7. **桌面 login 页测试策略**：共享 helper + 手机端测试 + 桌面标未验证，vs 建 InAppWebView platform fake（成本高）。

## 风险

- **PopScope/predictive-back**：`canPop` 必须预先可知；novel 修复依赖「`pop()` 绕过 `popDisposition`」这一 SDK 行为（3.47.2 源码已核实 + #163052），未来 Flutter 版本若改变命令式 pop 语义会破坏——测试用 `handlePopRoute` + 显式按钮 tap 双路径钉住。
- **嵌套路由**：novel/reverse-image 页在 root navigator GoRoute 上；显式 pop 直接作用于所在栈，预测性返回动画下的真实表现**未验证**（需 Android 真机/模拟器）。
- **`pick()` 新增 gen 校验**：picker 长时间挂起后返回会静默丢弃结果——这是 stopSearch 的目标语义，但意味着「选完图回来发现已 idle」；可接受但要在测试注释里写明。
- **stopSearch 与 cancel 竞态**：两者都 `++_generation`，交错调用安全（gen 单调）；但 `_releaseInput` 只在 cancel 路径——stopSearch 保图依赖「search() 迟到结果的 keepInput 分支」，该分支判定条件是 `error != null`（ApiCancelled 也是 error），实现时复核不被重构破坏。
- **WebView reload 语义**：`reload()` 重载当前 `_mainFrameUri` 页面（可能是 Pixiv 登录中页）vs `loadRequest(authorizeUrl)` 回到授权起点——授权页重载可能导致 PKCE state 已在 URL 里过期，推荐 recoverable 用 `reload()`、fatal 用 `beginSession` 新会话，**不要**混用。
- **凭据清除**：清 controller 后再点「保存」会写入空串——`_save` 已有非空校验需复核（L88–110 区域）；若无校验，清除后误点保存会写空凭据，实现时确认。
- **l10n 工作量**：预计新增 2–4 个 key ×4 语言 + `gen-l10n` + `gen_l10n_lookup.py` 重跑；漏跑 lookup 会编译过但动态 key 兜底成 raw key。
- **测试 finder 依赖文案**：profile_edit 返回按钮按 `cancel` tooltip 定位、reverse-image 按 `取消` 定位的既有用例，改 tooltip/key 后必须同步更新（已在 draft 标出）。

## 超出一个可评审 PR 的信号（stage 拆分建议）

若以下任一成立，按 implement.md §2 切成 stage 分支 `task/...-stage`：
- 浏览设置选方案 B（触 `settings_controller.dart` 安全策略 + 新 UI 状态）——此时 6 独立成 stage 2；
- login WebView 双端 + 共享 helper + 新 fake 超过 ~400 行 diff——9 独立成 stage；
- 总 diff > ~800 行或测试新增 > 15 个用例——按「页面组」切：stage1={1,2,3,5}（novel+profile，纯返回/锚点）、stage2={4,6,7,8,9}（search+settings+login，动作语义）。

## 运行时证据缺口（实现 PR 需标「未验证」）

- Android 预测性返回动画下 novel 页 chrome 拦截的真实手势链；
- 桌面 Windows WebView2 的 reload/重启路径（无 Linux 后端）；
- TalkBack/Narrator 下新按钮的语义朗读；
- 1.3x 大字体下错误卡双按钮是否溢出；
- 反向搜图真机 picking→stopSearch 的交互（模拟器 picker 行为）。
