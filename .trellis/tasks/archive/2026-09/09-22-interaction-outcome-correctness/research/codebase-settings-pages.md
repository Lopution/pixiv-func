# Codebase — 浏览设置「测试」+ 翻译凭据清除（§4.1 第 6、7 项）

## 6. 浏览设置「测试」隐式应用

`lib/features/settings/pages/browse_settings_page.dart`：

- L60–72 `_applyCustomInput()`：`ImageMirror.normalizeCustomSource` 校验 → **`_selectSource(normalized)`** → 返回 normalized。`_selectSource`（L40–44）调 `settingsControllerProvider.notifier.selectImageSource(...)`。
- `lib/core/settings/settings_controller.dart` L60–65 `selectImageSource`：`copyWith(imageSource: normalized, customImageSource: normalized)` + `queueSettingsWrite` —— **一次调用同时持久化选中源与自定义候选**。
- L85–119 `_testMirror()`：第一步就是 `_applyCustomInput()`（L90）→ 然后才 `pixivNetworkFactoryProvider.client(PixivRequestPurpose.image)` 发探针。即「测试」必然先持久化+切换源。注释（L56–59）自述原因：`imageMirrorAllowlistProvider` 只信任当前选中的镜像，候选必须先被选中才能走正常 client。
- 探针只判断「拿到 HTTP 响应」即成功（不区分状态码），`_testingMirror` 期间按钮禁用。
- L206–220 按钮排布：`保存`（`imageSourceSave`）与 `测试`（`imageSourceTest`，en "Test"）。

依赖链（已核实）：
- `settings_controller.dart` L336–343 `imageMirrorAllowlistProvider` = `ImageMirror.of(settings.imageSource).extraHosts`（只看 `imageSource` 字段）。
- `image_mirror.dart` L119–148 `normalizeCustomSource`：强制 https、仅 443、禁 IP/userinfo/query/fragment/尾点/连续点/非 ASCII host。
- `network_contracts.dart` L46–50：`PixivPolicyHttpClient.send` 对每个 host `registry.require`；image purpose 只允许 downloadHosts ∪ `_extraImageHosts`（L75–78）。候选不走 allowlist 就会被 `PixivUntrustedHostException` 拒掉。
- `network_policy.dart` L297–316 `clientFor(purpose, route, canonicalHost)` **不做 registry 校验**（registry 只在 `client()`/`send()` 路径）——存在绕过 allowlist 直连的通道，但绕过安全不变量不应在 W1 采用。

**Astra 断言成立**：「测试」有隐式持久化+切换副作用。

### 选项（§4.1 允许二选一，leaf 需用户确认）

- A) 改名「应用并测试」：`imageSourceTest` → 新 key（如 `imageSourceApplyAndTest`），行为不变。零网络层风险，最小 diff。
- B) 真测试语义：持久化只写 `customImageSource`（不动 `imageSource`）+ `imageMirrorAllowlistProvider` 改为同时放行 `customImageSource` host + 探针走正常 `client(image)`（保留镜像降级梯）。需要给 `SettingsController` 加 `saveCustomImageSource`（现有 copyWith 不暴露单字段写入）。注意：`customImageSource` 即使不被选中也会进 allowlist —— 该 host 已过硬编码校验（https/443/域名校验），放宽面可控，但属于安全策略扩张，须写明。

推荐 A 为 W1 默认；B 若采用需记入 leaf design 决策。

## 7. 翻译凭据清除后 UI 不同步

`lib/features/settings/pages/translation_credentials_page.dart`：

- L95–130 `_save()`：`_saving` 期间禁用两按钮；成功 `_status = savedOk`，异常 → `storeError`。
- L131–156 `_clear()`：`deleteBaidu()`/`deleteLlm()` → 成功 `_status = cleared`；异常 → `storeError`；`finally _saving=false`。**不清空 3 个 TextEditingController**（`_appIdController`/`_appKeyController`/`_llmKeyController`）——已删的密钥仍显示在输入框里，用户无法从 UI 判断「清除」是否生效；再次点保存会把刚删的密钥写回去。
- L206–213 `_status` 用 `colorScheme.primary` 渲染成功与错误同一样式 → 「成功/失败不可区分」。
- 处理中状态仅靠按钮禁用（无 spinner/文案），「处理中」与「空闲禁用」不可区分（虽然持续时间短，但满足不了「可区分」的验收措辞）。

`lib/core/comments/translation_credentials.dart`：`deleteBaidu`/`deleteLlm` 删 key，失败包 `TranslationCredentialsStoreException`（可观测错误链已就绪，UI 只需呈现）。

**Astra 断言成立**：清除不同步 UI + 三态不可区分。

修复要点：
- 成功清除后 `controller.clear()` 对应字段（baidu 两个 / llm 一个），或按 provider 分支清对应组；
- `_status` 记录语义类型（success/error）而非裸字符串，错误态用 `colorScheme.error`；
- 处理中在触发按钮上显式 `SizedBox` spinner（`FilledButton.icon`/`OutlinedButton.icon` 的 icon 位），或按钮文案变「清除中」——与 `_save` 对称实现；
- 不暴露密钥本身（controller.text 已是用户输入，清除即置空）。

测试：`test/settings_pages_test.dart` 有无覆盖凭据页需核对（搜索未直接命中 `translation_credentials` 测试文件——`grep -rln 'TranslationCredentials' test/` 需在实现时确认；若无则新建 widget 测试，secure store 已有 `inMemory` fake？需核实 `FlutterSecureStorage` 测试替代——`translation_credentials.dart` 若依赖 `flutter_secure_storage`，widget test 需 method-channel mock 或 provider override，见 risks）。

## 测试位置汇总

- `test/settings_pages_test.dart`、`test/settings_backup_page_test.dart` 存在；凭据页/浏览设置页的专门测试文件未在先前检索中确认命中，实现前需 `grep -rln 'browseSettings\|translationCredentials' test/` 核实（若无则新建，这是 W1 测试增量的主要部分）。
