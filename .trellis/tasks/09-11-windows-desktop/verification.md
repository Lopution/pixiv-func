# Windows 桌面验收证据与遗留限制

## 已完成（Linux/WSL 侧可验证）

| 项 | 结果 | 证据 |
| --- | --- | --- |
| `dart format --set-exit-if-changed lib test` | 通过 | 阶段 4 质量门 |
| `flutter analyze --no-pub` | No issues found | 阶段 4 质量门 |
| `flutter test` 全量 | 762 通过（743 + oauth/tls 单独 19） | oauth/tls 在 loopback 环境下需单独跑，属已知环境性慢，非回归 |
| `git diff --check` | 干净 | 阶段 4 质量门 |
| 平台能力矩阵 | 16 测试通过 | `test/windows_capability_test.dart` |
| 登录拦截判定纯函数 | 5 测试通过 | `test/login_navigation_decision_test.dart` |
| Android 语义零回归 | 全量套件通过 | 上述 |

## Windows 侧人工验收清单（必须在真实 Windows 执行）

- [x] `flutter build windows --release` 成功（CI `windows-build` 通过并产出 zip）
- [x] 启动 `pixiv_func.exe`：标题 `Pixiv Func`，正常运行（已实机验证）
- [x] 登录：`/login/web` 走 `LoginWebViewDesktopPage`（WebView2），OAuth 授权链 → `pixiv://account?code=` → exchange → 登录成功（已实机验证：去掉 `useShouldOverrideUrlLoading` 的 CDP Fetch 拦截后 `CONNECTION_ABORTED` 消除，登录成功、feed/我的/画师页真实数据加载）
- [ ] 注册入口（`create=true`）同样完成
- [x] 代理/直连下真实数据：推荐/排行/我的/画师页在 Windows 本机加载（截图验证）
- [ ] 样板链路：详情 → 大图 → 收藏 → 下载
- [ ] 下载落盘：默认目录 `<Downloads>/PixivFunc`、`.part` 暂存、完成物化、取消清理
- [ ] 账号迁移：导出/导入二维码 + 剪贴板兜底在 Windows 端工作
- [ ] 窗口缩放、横宽比、NavigationRail 切换正常
- [ ] 卸载/缺失 WebView2 Runtime 时登录页出现安装指引（`getAvailableVersion()` 探测路径）
- [ ] 以图搜图：file_selector 选图 → SauceNAO 结果页（InAppWebView）→ 外链进系统浏览器

## 已知限制 / 不声明完成项

- **登录已实机验证**（见上表），注册链路、下载落盘、迁移导入、WebView2 缺失提示仍待 Windows 逐项验收。
- **rhttp 系统代理**：Clash 环境下 rhttp 是否跟随系统代理未实测（design §6 要求先实测再决定是否加设置项）。
- **`useShouldOverrideUrlLoading` 已在 Windows WebView 上禁用**：插件用 CDP `Fetch.requestPaused` 暂停每个文档请求，OAuth 302 链中间跳被 abort（`CONNECTION_ABORTED`）。现在登录页与 SauceNAO 页都靠 `onLoadStart`（NavigationStarting）+ `onUpdateVisitedHistory` 观察回调；`pixiv://account` 回调在 `onLoadStart` 拦截后 `stopLoading`。重新启用前必须有可复现理由 + 回归测试。
- **WebView2 缺失探测**：`getAvailableVersion()` 在 Windows 侧返回 `null` 的分支只到 UI 层；真实缺失环境的表现未验证。
- **后台 widget / 推送 / SAF**：桌面端均为显式降级（inert intent source、MissingPlugin 吞掉、desktop file sink），未做桌面等价功能。
