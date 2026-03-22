# 数据库性能优化

## Core Data 批量操作

### NSBatchInsertRequest

```swift
// ❌ 逐条插入（每条都触发上下文变更通知和验证）
func importItems(_ jsonItems: [[String: Any]], context: NSManagedObjectContext) {
    for json in jsonItems {
        let item = Item(context: context)
        item.title = json["title"] as? String
        item.timestamp = json["timestamp"] as? Date
    }
    try? context.save()  // 大量数据时极慢
}

// ✅ 批量插入（绕过上下文，直接写入持久化存储）
func importItems(_ jsonItems: [[String: Any]]) {
    let batchInsert = NSBatchInsertRequest(
        entity: Item.entity(),
        managedObjectHandler: { (obj: NSManagedObject) -> Bool in
            // 返回 true 表示结束插入
            guard !jsonItems.isEmpty else { return true }
            let item = obj as! Item
            let json = jsonItems.removeFirst()
            item.title = json["title"] as? String
            item.timestamp = json["timestamp"] as? Date
            return false
        }
    )
    batchInsert.resultType = .statusOnly
    try? persistentContainer.viewContext.execute(batchInsert)
}
```

### NSBatchDeleteRequest

```swift
// ❌ 逐条删除
func deleteExpiredItems(context: NSManagedObjectContext) {
    let request: NSFetchRequest<Item> = Item.fetchRequest()
    request.predicate = NSPredicate(format: "expiresAt < %@", Date() as CVarArg)
    let items = try? context.fetch(request)
    items?.forEach { context.delete($0) }
    try? context.save()  // 大量数据时极慢
}

// ✅ 批量删除（直接在持久化存储层执行）
func deleteExpiredItems() {
    let fetchRequest: NSFetchRequest<NSFetchRequestResult> = Item.fetchRequest()
    fetchRequest.predicate = NSPredicate(format: "expiresAt < %@", Date() as CVarArg)
    let batchDelete = NSBatchDeleteRequest(fetchRequest: fetchRequest)
    batchDelete.resultType = .resultTypeObjectIDs

    // 执行批量删除
    let result = try? persistentContainer.viewContext.execute(batchDelete) as? NSBatchDeleteResult
    // ⚠️ 批量操作绕过上下文，需手动合并变更
    if let objectIDs = result?.result as? [NSManagedObjectID] {
        NSManagedObjectContext.mergeChanges(
            fromRemoteContextSave: [NSDeletedObjectsKey: objectIDs],
            into: [persistentContainer.viewContext]
        )
    }
}
```

### NSBatchUpdateRequest

```swift
// ✅ 批量更新（直接在持久化存储层执行）
func markAllAsRead() {
    let batchUpdate = NSBatchUpdateRequest(entity: Message.entity())
    batchUpdate.predicate = NSPredicate(format: "isRead == NO")
    batchUpdate.propertiesToUpdate = ["isRead": true]
    batchUpdate.resultType = .updatedObjectIDsResultType
    try? persistentContainer.viewContext.execute(batchUpdate)
}
```

---

## Core Data 索引优化

```
Data Model Inspector 中为常用查询字段添加索引：
  1. 选中 Entity → Attributes → 选中属性
  2. 勾选 "Indexed"

或在代码中配置复合索引：
  Entity Inspector → Fetch Index Elements → 添加复合索引
```

**索引使用原则：**

| 场景 | 是否加索引 | 说明 |
|------|----------|------|
| 频繁作为 predicate 条件的属性 | **是** | 加速查询过滤 |
| 频繁用于 sortDescriptors 的属性 | **是** | 加速排序 |
| 唯一标识字段（如 id） | **是** | 加速精确查找 |
| 很少查询的属性 | **否** | 索引增加写入开销和存储空间 |
| 频繁更新的属性 | **谨慎** | 每次更新需维护索引 |

```swift
// ✅ Predicate 使用索引字段在前
// 假设 userId 有索引，timestamp 有索引
let predicate = NSPredicate(format: "userId == %@ AND timestamp > %@", userId, cutoffDate)

// ✅ fetchLimit 限制结果数量
let request: NSFetchRequest<Message> = Message.fetchRequest()
request.fetchLimit = 20  // 只取 20 条
request.fetchBatchSize = 20  // 每批加载 20 条（惰性加载）
```

---

## Core Data 后台上下文

```swift
// ❌ 在主线程上下文执行耗时操作
func importData(_ data: [RawItem]) {
    let context = persistentContainer.viewContext  // 主线程上下文
    for raw in data {
        let item = Item(context: context)  // 阻塞主线程
        item.configure(with: raw)
    }
    try? context.save()  // 主线程卡顿
}

// ✅ 使用后台上下文执行耗时操作
func importData(_ data: [RawItem]) {
    persistentContainer.performBackgroundTask { context in
        // 在后台线程执行
        for raw in data {
            let item = Item(context: context)
            item.configure(with: raw)
        }
        do {
            try context.save()
            // 变更自动合并到 viewContext（需配置 automaticallyMergesChangesFromParent）
        } catch {
            print("后台保存失败: \(error)")
        }
    }
}

// ✅ 配置自动合并
persistentContainer.viewContext.automaticallyMergesChangesFromParent = true
```

---

## NSFetchedResultsController 性能

```swift
// ✅ 正确配置 NSFetchedResultsController
let fetchRequest: NSFetchRequest<Item> = Item.fetchRequest()
fetchRequest.sortDescriptors = [NSSortDescriptor(keyPath: \Item.timestamp, ascending: false)]
fetchRequest.fetchBatchSize = 20  // 关键：惰性加载

let controller = NSFetchedResultsController(
    fetchRequest: fetchRequest,
    managedObjectContext: persistentContainer.viewContext,
    sectionNameKeyPath: nil,  // 不分组时设 nil，避免额外计算
    cacheName: "ItemsCache"   // 启用缓存，避免重复计算 section 信息
)

// ⚠️ 性能注意事项：
// - sectionNameKeyPath 应使用索引字段，否则需全表扫描
// - cacheName 在 predicate/sortDescriptors 变化时需调用 deleteCache(withName:)
// - fetchBatchSize 设置合理值（约等于屏幕可见行数）
```

---

## SwiftData 性能注意事项

```swift
// ⚠️ SwiftData 目前（iOS 17-18）的已知性能限制
// 1. 无原生批量操作 API（内部仍逐条处理）
// 2. @Query 在大数据集时可能性能不佳
// 3. 后台操作需使用 ModelActor

// ✅ 使用 ModelActor 进行后台操作
@ModelActor
actor DataImporter {
    func importItems(_ rawItems: [RawItem]) throws {
        for raw in rawItems {
            let item = Item(title: raw.title, timestamp: raw.timestamp)
            modelContext.insert(item)
        }
        try modelContext.save()
    }
}

// ✅ @Query 配合分页避免一次加载过多数据
struct ItemListView: View {
    @Query(sort: \Item.timestamp, order: .reverse)
    private var items: [Item]
    // ⚠️ 大数据集时考虑 fetchLimit 或手动分页

    var body: some View {
        List(items.prefix(50)) { item in  // 限制显示数量
            ItemRow(item: item)
        }
    }
}

// ✅ 需要高性能批量操作时，仍可通过底层 Core Data 执行
// SwiftData 的 ModelContext 底层基于 Core Data
```

---

## SQLite 直接访问的性能优化

当 Core Data 无法满足性能需求时，可考虑直接使用 SQLite（通过 GRDB、SQLite.swift 等）。

```swift
// ✅ 使用事务批量操作
func importItems(_ items: [RawItem], db: Database) throws {
    try db.inTransaction {
        for item in items {
            try db.execute(
                sql: "INSERT INTO items (title, timestamp) VALUES (?, ?)",
                arguments: [item.title, item.timestamp]
            )
        }
        return .commit
    }
    // 事务内批量操作比逐条提交快 10-100 倍
}

// ✅ WAL 模式（Write-Ahead Logging）
// 允许并发读写，显著提升读写混合场景性能
try db.execute(sql: "PRAGMA journal_mode=WAL")

// ✅ 合理使用预编译语句
let statement = try db.cachedStatement(sql: "SELECT * FROM items WHERE id = ?")
// 预编译语句避免重复解析 SQL，高频查询时性能提升明显
```

**SQLite 性能参数：**

| PRAGMA | 作用 | 推荐值 |
|--------|------|--------|
| `journal_mode=WAL` | 写前日志，提升并发性能 | 推荐开启 |
| `synchronous=NORMAL` | 降低 fsync 频率（WAL 模式下安全） | WAL 模式下推荐 |
| `cache_size=-2000` | 缓存 2MB 数据页 | 根据数据量调整 |
| `mmap_size=268435456` | 内存映射 256MB | 大数据库推荐 |

---

## 常见问题速查

```
看到 Core Data 保存耗时长
  → 原因：主线程逐条插入/删除大量数据
  → 优化：使用 NSBatchInsertRequest/NSBatchDeleteRequest + 后台上下文

看到列表滚动时 Core Data 查询卡顿
  → 原因：fetchBatchSize 未设置，一次性加载全部数据
  → 优化：设置 fetchBatchSize + 使用 NSFetchedResultsController

看到 Core Data 查询慢
  → 原因：查询字段无索引 / predicate 复杂
  → 优化：为常用查询字段添加索引 + 优化 predicate

看到 SwiftData 大数据导入慢
  → 原因：SwiftData 无原生批量操作
  → 优化：使用 ModelActor 后台操作，或回退到 Core Data 批量 API

看到 SQLite 写入慢
  → 原因：未使用事务 / 未开启 WAL 模式
  → 优化：批量操作包裹在事务中 + 开启 WAL
```

---

## 数据库性能优化 checklist

```
- [ ] 大批量数据操作是否使用了 NSBatchInsertRequest/NSBatchDeleteRequest？
- [ ] 批量操作后是否正确合并了上下文变更？
- [ ] 常用查询字段是否添加了索引？
- [ ] 耗时的数据操作是否在后台上下文执行？
- [ ] NSFetchedResultsController 是否设置了 fetchBatchSize？
- [ ] NSFetchRequest 是否设置了 fetchLimit 避免全表加载？
- [ ] viewContext 是否配置了 automaticallyMergesChangesFromParent？
- [ ] SwiftData 大数据操作是否使用了 ModelActor？
- [ ] SQLite 批量写入是否使用了事务？
- [ ] SQLite 是否开启了 WAL 模式？
```
