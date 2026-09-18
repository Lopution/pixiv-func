# 设计：Feed 快照冷启动与离线动作队列

## 1. 存储层

```
lib/core/persistence/
  feed_database.dart        — FeedDatabase（feeds.db，schemaVersion 1，同款 factory 注入）
  feed_snapshot_store.dart  — FeedSnapshotStore（快照 CRUD + LRU 逐出）
lib/core/actionqueue/
  action_queue.dart         — ActionQueue 引擎（enqueue/drain/coalesce/backoff）
  action_models.dart        — ActionRequest/StoredAction/RetryScope/ActionTelemetry
  action_store.dart         — ActionStore 接口（sqflite 实现 + 内存测试实现）
```

`feed_snapshots` 表：

```sql
CREATE TABLE feed_snapshots (
  account_id TEXT NOT NULL,
  feed_key TEXT NOT NULL,
  ids TEXT NOT NULL,            -- JSON int[]
  entities TEXT NOT NULL,       -- JSON {entityType: {id: payload}}
  cursor TEXT,
  saved_at INTEGER NOT NULL,
  snapshot_version INTEGER NOT NULL DEFAULT 1,
  PRIMARY KEY (account_id, feed_key)
)
```

`action_queue` 表：

```sql
CREATE TABLE action_queue (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  owner TEXT NOT NULL,          -- accountId
  type TEXT NOT NULL,           -- 'bookmark.add' | 'bookmark.delete' | 'follow.add' | 'follow.delete'
  dedupe_key TEXT NOT NULL,     -- '$type:$targetId'
  payload TEXT NOT NULL,        -- JSON，由 handler 解析
  status TEXT NOT NULL CHECK (status IN ('pending','running','failed')),
  attempt INTEGER NOT NULL DEFAULT 0,
  next_attempt_at INTEGER,
  created_at INTEGER NOT NULL
)
CREATE INDEX aq_ready ON action_queue(owner, status, next_attempt_at);
```

## 2. 快照冷启动

```
build()
  ├─ ref.watch(accountStore boundary)            （现有，不变）
  ├─ snapshot = await FeedSnapshotStore.read(accountId, feedKey)
  │    ├─ hit & fresh  → store.merge(entities); 返回 ids 态 + schedule(refresh)
  │    ├─ expired/corrupt → delete row; telemetry.snapshotDiscarded++
  │    └─ miss → 正常首载（现状路径）
  └─ fetch 成功后 → store.write(前 60 ids + 实体)（后台 fire-and-forget）
```

- 快照恢复把 `initialPhase` 直接置 `idle`（内容已可见）；后台 refresh 复用现有 `refresh()`（refreshPhase spinner 保留在 app bar，不打断阅读）。
- `refresh()` 成功提交后同样写快照——内容始终「最后一次看到的样子」。
- **codec 挂钩**：`PagedFeedController` 新增可选 `snapshotCodec`——`FeedEntityCodec { encode(Map<int, Entity>) → json; decode(json) → Map<int, Entity>; storeMerge(entities) }`。illust feeds 提供 `IllustSnapshotCodec`，novel feeds 提供 `NovelSnapshotCodec`；子类不实现即关闭快照（comment/spotlight 等）。
- 恢复的实体先 `store.merge`，卡片照常渲染；失效实体（商店已清）静默缺失——快照内容源自 store 本身，无一致性问题。

## 3. 离线动作队列

```
enqueue(type, dedupeKey, payload, {coalesce: true})
  └─ 同事务删同 owner+dedupeKey 的 PENDING 行 → 插入新行

drain(owner)                                    串行
  └─ 取 owner 下 status='pending' 且 next_attempt_at<=now 的行
       → status='running' → handler.execute(payload)
         ├─ 成功 → 删行，telemetry.actionReplayed++
         ├─ RetryScope.queueCooldown（网络/429/5xx）
         │     → 行回 pending；全队 cooldownUntil = now+backoff；停 drain
         └─ RetryScope.row（4xx/业务拒绝）
               → attempt++；attempt<max(5)→ pending+行级 next_attempt_at；
                 否则 status='failed' + telemetry.actionDropped++
```

- `ActionHandler` 注册表：`type → (payload, cancelToken) → Future<void>`，handler 内部调**同一 repository 方法**（`bookmarkStore.add`/`userRepository.follow` 等），重放后确认态走既有 MutationLedger 路径。
- **触发点**（不引 connectivity_plus）：`ActionQueue` 装配在账号级 provider，`build` 时 drain 一次 + 订阅 `ref.listen` 前台恢复；另在 `MutationLedger` 确认成功路径里 piggyback `queue.pump(accountId)`（一次成功调用即连通证据）。
- **离线判定**：mutation 抛出 `ApiNetworkError`/`ApiCancelled` 之外的网络类 `ApiError`（连接超时/无 DNS）→ 入队；4xx/401 直接失败不入队。
- 入队时 store 的 pending 态保留（用户已见乐观态）；重放确认前 pending 不收敛为 server truth——与 MutationLedger 契约一致。

## 4. 测试策略

- `FeedDatabase`/`ActionStore`：sqflite_ffi 内存库（history 测试同款）。
- `PagedFeedController`：假 clock + 假 `FeedSnapshotStore` + 假 repository——验证首帧直出、过期丢弃、后台 refresh 替换、账号隔离。
- `ActionQueue`：内存 `ActionStore` + 假 handler——coalesce、串行、RetryScope 两档、maxAttempts、跨 owner 不取。
- store 集成：bookmark store 在假 client 抛网络错误时入队；重放后状态确认。

## 5. 不做的事

- 不持久化 `MutationEnvelope` 本身——队列行是独立 schema（type+payload），ledger 仍在内存管在途态；重放会重新生成 envelope。
- 不在快照里存实体二进制/图片；不做跨 feed 共享实体表（60×64 行级规模，单表 JSON 足够）。
- 不给 action_queue 做 UI——计数走 telemetry；failed 行由下次成功 drain 重试窗口自然清理或留存观测。
