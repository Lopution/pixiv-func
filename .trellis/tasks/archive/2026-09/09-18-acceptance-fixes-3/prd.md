# prd — acceptance-fixes-3（第三轮真机验收返工）

来源：用户对 PR #38 包的真机验收反馈。

## 背景

- 个人页头部改造不被认可：展开态按钮分散、折叠态 4 图标把名字挤偏、且设置入口依赖个人页加载（失败时无处进设置）。用户拍板改走 PixEz 模型：**设置升第 5 tab，个人页入口只剩设置页顶部卡片**。
- 以图搜源预览仍是大灰盒小图（`_aspect` 永 null → 卡死 4:3 兜底），重选/取消按钮被挤出屏；选图走文件管理器应换相册 picker。
- 小说 `acb(response parse error)` 诊断透出生效：真实错误是 `FormatException (line 2 char 21) sessionUserId:` —— bootstrap `value:` 是 JS 对象字面量，需宽松解析。

## 范围

1. `/me`↔`/settings` 对称置换：设置成分支 4（子树嫁接），`/me` 成根层推入路由，底栏/侧栏第 5 项换图标文案，宽屏 rail 冗余齿轮删除
2. 个人页 header：删冗余设置齿轮，折叠态 actions 收 `⋯` 溢出菜单（A 方案），展开态 share+edit 归并
3. 搜图：预览单解码贴图收缩（去监听器机制），重选/取消上移可见，选图改 `ACTION_PICK_IMAGES`/`ACTION_PICK`
4. novel parser：JS 字面量→JSON 规范化器（无引号 key、单引号串、undefined）

## 验收

- 第 5 tab=设置；设置页顶部卡片→个人页；个人页加载失败时设置仍可达
- 折叠 toolbar 名字居中且与控件不撞
- 搜图预览贴图无黑边，重选/取消首屏可见，相册界面选图
- 小说能加载正文（真实 bootstrap 含无引号 key）
- `flutter analyze` 0 issue、全量测试过、CI 绿
