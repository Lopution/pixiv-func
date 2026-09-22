# Codebase — 搜索页热门标签裁剪 + 反向搜图取消语义（§4.1 第 4、8 项）

## 4. 热门标签为整行丢内容

`lib/features/search/search_page.dart` L140–161：

```dart
gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
  crossAxisCount: 3, ...),
itemCount: tags.length >= 3
    ? tags.length - (tags.length % 3)   // L153–155
    : tags.length,
```

注释自述「避免悬空的最后一行」：≥3 个标签时把余数（1–2 个）直接丢弃。这是为视觉对称丢弃服务端内容——**Astra 断言仍成立**，且与「tag 可导航」目标（`openSearchResults(NovelSearchQuery)`，L132 `_TrendingTagTile` 点击行为）直接冲突：被丢的标签恰恰是可操作的。

修复选项（leaf 决策，见 implementation-draft）：
- a) `itemCount: tags.length` —— 保留 3 列网格，末行不满。最小 diff，直接满足「不丢内容」。
- b) `SliverToBoxAdapter` + `Wrap` —— 末行自然排布、无空格观感，但 tile 是 `Stack fit expand` 方形卡，需要给定显式尺寸，diff 更大；观感细节属 W2 视觉收敛范围。

推荐 a)：W1 只保证内容可达性，排布观感留给 W2。

测试：`test/search_page_test.dart` 若有热门标签断言需对照更新；新增「余数标签仍渲染且可点」用例（构造 4/5 个标签，断言最后一个 tile 存在且 tap 后 push 小说搜索路由）。

## 8. 反向搜图：取消当前搜索 ≠ 离开页面

### 现状代码

`lib/features/search/reverse_image_search_page.dart`：

- L117–120 `_cancelAndPop()`：`await _controller.cancel()` 后 `Navigator.pop()` —— 「取消并离开」复合动作。
- L167–171 AppBar leading：`Icons.arrow_back` + `tooltip: context.l10n.searchReverseCancel` + `onPressed: _cancelAndPop`。箭头图标承担「返回」，tooltip 却叫「取消」，且语义是「销毁输入 + 离页」。
- L285–304 `_progress`（picking/preparing/searching 共用的进度卡）：唯一的 `OutlinedButton(取消)` → `_cancelAndPop` —— **Bug 核心**：进度中点「取消」不是停掉当前操作留在页面，而是销毁流程并退出页面。
- L338–347 `ready` 状态：`OutlinedButton(取消)` → `_controller.cancel()`（不 pop），回到 `canceled` → `_idle` 空态 —— 这里「取消」= 丢弃已持图片，留在页面，语义是合理的（但与进度卡的「取消」同名不同后果）。

`lib/core/reverse_image/reverse_image_controller.dart`：

- L14–33 `ReverseImageFlowStatus`：idle/picking/preparing/ready/searching/success/failure/canceled。
- L158–176 `pick()`：`platform.pickImage()` 返回后**没有 generation 校验**就直接 `prepare` —— picking 期间的取消意图无处安放（平台选图器也无法编程关闭，但返回后应能丢弃结果）。
- L178–249 `prepare()`：入口 `++_generation`（L180），copy 完成后 L201–203 校验 `generation != _generation` → 删文件丢弃 —— 与「取消时 gen++」兼容。
- L288–425 `search()`：L296 捕获 `final generation = _generation`（不自增）；取消/停止使 gen 变大后，异步结果在 L348 被丢弃。`ApiCancelledException` 走 error 分支 → `keepInput`（error != null）→ 跳过 `_releaseInput` → 输入保留 —— 天然支持「停搜保图」。
- L427–449 `cancel()`：`++_generation` + token cancel + `_releaseInput`（删临时文件、disarm）→ `canceled`。这是「销毁流程」语义，页面需要另一个「停止当前操作但保留输入」的方法。

`lib/core/reverse_image/reverse_image_navigation_policy.dart`：只管结果 WebView 的导航裁决（navigate/open-in-app/open-external/reject），与取消无关。

**Astra 断言成立**：进度中「取消」离开页面；需要新增 `stopSearch()`（或同义）controller 方法 + 页面接线。

### `stopSearch` 设计要点（静态可推，供 draft）

```dart
Future<void> stopSearch() async {
  if (_closed) return;
  ++_generation;                 // 使 picking/preparing/searching 的迟到结果全部失效
  _cancelToken?.cancel();        // 中断在飞 provider 请求
  _cancelToken = null;
  final input = _input;
  if (input != null) {
    _setState(ReverseImageFlowState(status: ready, engine: ..., input: input.info, engineFailures: state.engineFailures));
  } else {
    _setState(ReverseImageFlowState.idle(engine: state.engine));
  }
}
```

- searching 中调用 → provider 抛 `ApiCancelled` → keepInput 保留输入 → gen 校验丢弃迟到结果 → 停在 `ready`（图还在，可换引擎重搜）。
- preparing 中调用 → gen 已超前 → copy 完成后走 L201 删文件返回 → 停在 idle（输入未建立，无图可保）。
- picking 中调用 → 需在 `pick()` 的 `pickImage()` await 后补 `generation != _generation` 校验（当前缺失），picker 返回的文件引用被丢弃、停在 idle。
- 页面接线：进度卡「取消」→ `stopSearch`；AppBar leading 保留「离开」语义但图标/tooltip 统一为返回（`MaterialLocalizations.backButtonTooltip` 或 l10n.back），onPressed 仍 `_cancelAndPop`（离开页面时销毁临时输入是正确的资源语义——controller.dispose 也会兜底）。
- `ready` 状态的「取消」可保留为「丢弃图片」但更准确的命名是「清除/移除图片」，属于术语决策点。

### 测试

- `test/reverse_image_search_test.dart` L259–331：`cancel and rate limit both clean the owned input` 等已钉住 `cancel()` 全销毁语义 → 保留不改；新增 `stopSearch` 用例：searching 中调用 → ready + input 仍在；preparing 中调用 → idle + 迟到 copy 被删。
- `test/reverse_image_search_page_test.dart` L81–101：ready 态取消回 pick 屏的页面用例已存在；需新增「searching 进度卡取消留在页面且回到 ready」页面用例（FakeReverseImageProvider 挂起 search，pump 后点取消，断言路由未 pop、状态 ready）。
- 平台 fake：`_FakePlatform`/`_FakeProvider`/`controllable` provider 模式已在两测试文件内就绪。
