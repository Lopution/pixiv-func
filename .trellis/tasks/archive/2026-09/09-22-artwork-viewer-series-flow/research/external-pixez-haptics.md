# 外部调研：PixEz `HapticUtil` — `lib/utils/haptic_util.dart`（快照行号核对）

## 1. 结构（薄静态包装，共 ~86 行核心区）

```dart
class HapticUtil {
  static int _lastTriggerTime = 0;                    // L25

  static bool get _isEnabled {                        // L28-33
    try { /* 读 settings 开关（平台 + 用户偏好） */ }
    catch (_) { return false; }                       // 平台不支持 → 静默关
  }

  static bool _canTrigger([int minIntervalMs = 60,    // L40-46
                          bool force = false]) {
    final now = ...millisecondsSinceEpoch;
    if (!force && now - _lastTriggerTime < minIntervalMs) return false;
    _lastTriggerTime = now;
    return true;
  }

  static void _trigger(Future<void> Function() action,// L50-55
                       int minIntervalMs, {bool force = false}) {
    if (!_isEnabled || !_canTrigger(minIntervalMs, force)) return;
    try { action(); } catch (_) {}                    // MissingPluginException 等吞掉
  }
}
```

## 2. 公开 API 与节流参数（L60-83）

| 方法 | 底层 | 最小间隔 | 语义用途（PixEz 内实际用法） |
|---|---|---|---|
| `selectionClick` | `HapticFeedback.selectionClick` | 50ms | 普通 tap 确认（卡片 tap、tab 切换） |
| `light` | `lightImpact` | 80ms，可 force | 轻确认（复制成功） |
| `medium` | `mediumImpact` | 100ms | 中确认 |
| `heavy` | `heavyImpact` | 120ms | 长按弹菜单、保存触发 |
| `success` | `mediumImpact` | 120ms | 成功语义复用 medium |
| `warning` | `heavyImpact` | 150ms | 警告复用 heavy |
| `error` | `heavyImpact` | 150ms | 失败复用 heavy |
| `vibrate` | `vibrate` | 150ms | 兜底 |

## 3. 对本项目薄包装的借鉴与裁剪

设计 §5.6 只要求 `selectionClick`/`heavyImpact` 两级起步。建议（见 implementation-draft.md）：

- **保留**：`unawaited`-safe 的 `try/catch`（平台无触觉硬件/插件缺失时静默）、最小间隔节流（防滚动连发/快速连点震成一片）、settings 开关位（接 `settingsController` 已有通道）。
- **裁剪**：success/warning/error/vibrate 先不做——§5.6 分级表用 `selectionClick` + `heavyImpact` 即可覆盖 W4 触点；API 表面越大，W6/W7 消费前越容易发散。
- **差异点**：PixEz 是全局静态类 + `userSetting` 直读；本项目应做成 `lib/app/` 下的薄 wrapper（静态或 Riverpod provider 均可，静态更贴合 `HapticFeedback` 自身的静态形态，但开关要从 `settingsController` 读——建议静态方法内部 `ref.read` 不可行，故做成可在 widget 层直接调的静态 + 内部持有可由 bootstrap 注入的 enabled getter，或 provider。实现稿给结论）。
- PixEz 调用点印证分级：长按菜单=heavy（L476 detail / L257 viewer）、复制成功=light（L463）——与本项目 §5.6 “进入管理模式/危险确认/保存成功=显式振动”映射一致。
