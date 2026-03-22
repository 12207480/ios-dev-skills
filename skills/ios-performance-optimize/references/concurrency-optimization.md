# Swift Concurrency 性能优化

## Actor 重入问题与性能影响

Actor 保证串行访问，但在 `await` 挂起点处允许其他调用进入（重入），这既是设计也是性能陷阱。

```swift
// ⚠️ Actor 重入导致状态不一致
actor DataManager {
    var cache: [String: Data] = [:]

    func fetchData(for key: String) async -> Data {
        if let cached = cache[key] {
            return cached
        }
        // ⚠️ await 挂起时，另一个调用可能进入并重复请求
        let data = await networkService.fetch(key)
        cache[key] = data
        return data
    }
}

// ✅ 使用 Task 缓存防止重复请求
actor DataManager {
    private var cache: [String: Data] = [:]
    private var inFlightTasks: [String: Task<Data, Error>] = [:]

    func fetchData(for key: String) async throws -> Data {
        if let cached = cache[key] {
            return cached
        }
        // 复用已有的请求任务，避免重复网络调用
        if let existingTask = inFlightTasks[key] {
            return try await existingTask.value
        }
        let task = Task {
            let data = try await networkService.fetch(key)
            cache[key] = data
            inFlightTasks[key] = nil
            return data
        }
        inFlightTasks[key] = task
        return try await task.value
    }
}
```

---

## Task 创建开销

每个 Task 都有堆分配和调度开销，在循环中创建大量 Task 会导致严重的性能问题。

```swift
// ❌ 在循环中创建大量 Task（堆分配 + 调度开销爆炸）
func processItems(_ items: [Item]) {
    for item in items {
        Task {  // 1000 个 item = 1000 个 Task
            await process(item)
        }
    }
}

// ✅ 使用 TaskGroup 控制并发（自动管理任务调度）
func processItems(_ items: [Item]) async {
    await withTaskGroup(of: Void.self) { group in
        for item in items {
            group.addTask {
                await process(item)
            }
        }
    }
}

// ✅ 限制并发数量，避免资源争抢
func processItems(_ items: [Item]) async {
    let batchSize = 10
    for batch in items.chunked(into: batchSize) {
        await withTaskGroup(of: Void.self) { group in
            for item in batch {
                group.addTask {
                    await process(item)
                }
            }
        }
    }
}
```

---

## MainActor 上下文切换成本

`@MainActor` 标注会导致函数在主线程执行，频繁跨 Actor 切换有调度开销。

```swift
// ❌ 不必要的 MainActor 标注导致频繁切换
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []

    // 整个方法都在主线程执行，包括数据解析
    func loadData() async {
        let data = await api.fetchItems()  // 切到后台再切回来
        let parsed = parseItems(data)      // 解析在主线程，浪费
        items = parsed
    }
}

// ✅ 仅在需要更新 UI 时切到 MainActor
class ViewModel: ObservableObject {
    @MainActor @Published var items: [Item] = []

    // nonisolated：不绑定到 MainActor，在协作线程池执行
    nonisolated func loadData() async {
        let data = await api.fetchItems()
        let parsed = parseItems(data)  // 解析在后台线程
        await MainActor.run {
            items = parsed  // 仅 UI 更新在主线程
        }
    }
}

// ❌ 在循环中频繁切换到 MainActor
for item in items {
    await MainActor.run {
        updateProgress(item)  // 每次循环都切换一次
    }
}

// ✅ 批量更新减少上下文切换次数
await MainActor.run {
    for item in items {
        updateProgress(item)  // 一次切换，批量更新
    }
}
```

---

## TaskGroup vs 手动 Task 管理

| 维度 | TaskGroup | 手动创建 Task |
|------|-----------|--------------|
| 结构化并发 | 是（自动取消子任务） | 否（需手动管理） |
| 错误传播 | 自动（一个失败可取消全部） | 需手动处理 |
| 内存管理 | group 结束时自动清理 | 需手动持有和释放 |
| 并发控制 | 系统自动调度 | 无限制，可能过载 |
| 适用场景 | 批量并行操作 | 独立的后台任务 |

```swift
// ✅ TaskGroup：批量并行请求（推荐）
func fetchAllProfiles(_ ids: [String]) async throws -> [Profile] {
    try await withThrowingTaskGroup(of: Profile.self) { group in
        for id in ids {
            group.addTask {
                try await api.fetchProfile(id)
            }
        }
        var results: [Profile] = []
        for try await profile in group {
            results.append(profile)
        }
        return results
    }
}

// ✅ 手动 Task：独立的后台任务（不依赖调用者生命周期）
class ImagePrefetcher {
    private var prefetchTasks: [URL: Task<UIImage?, Never>] = [:]

    func prefetch(_ url: URL) {
        guard prefetchTasks[url] == nil else { return }
        prefetchTasks[url] = Task {
            await ImageLoader.load(url)
        }
    }

    func cancel(_ url: URL) {
        prefetchTasks[url]?.cancel()
        prefetchTasks[url] = nil
    }
}
```

---

## async let 并行执行优化

```swift
// ❌ 串行 await（总耗时 = 各请求耗时之和）
func loadDashboard() async throws -> Dashboard {
    let profile = try await api.fetchProfile()
    let feed = try await api.fetchFeed()         // 等 profile 完成才开始
    let notifications = try await api.fetchNotifications()  // 等 feed 完成才开始
    return Dashboard(profile: profile, feed: feed, notifications: notifications)
}

// ✅ async let 并行（总耗时 ≈ 最慢的单个请求）
func loadDashboard() async throws -> Dashboard {
    async let profile = api.fetchProfile()
    async let feed = api.fetchFeed()
    async let notifications = api.fetchNotifications()
    // 三个请求同时发出，并行执行
    return try await Dashboard(profile: profile, feed: feed, notifications: notifications)
}
```

---

## Sendable 检查的性能影响

Sendable 是编译期检查，本身不影响运行时性能，但为满足 Sendable 要求而做的改造可能影响性能。

```swift
// ⚠️ 为满足 Sendable 而将 class 改为 struct 可能导致不必要的拷贝
// 大型数据模型频繁跨 Actor 传递时，struct 深拷贝可能比 class 引用传递更慢

// ✅ 使用 @unchecked Sendable 在确认线程安全时避免不必要的改造
final class ThreadSafeCache: @unchecked Sendable {
    private let lock = NSLock()
    private var storage: [String: Data] = [:]

    func get(_ key: String) -> Data? {
        lock.lock()
        defer { lock.unlock() }
        return storage[key]
    }
}

// ✅ 对于值类型集合，利用 COW 减少实际拷贝
// Swift Array/Dictionary/String 等自带 COW，跨 Actor 传递时如果不修改则不会真正拷贝
```

---

## 常见反模式速查

```
在循环中创建大量 Task
  → 问题：堆分配和调度开销爆炸，可能导致线程饥饿
  → 修正：使用 TaskGroup 或分批处理

在 Actor 方法中执行耗时同步操作
  → 问题：阻塞 Actor 的串行队列，其他调用全部排队等待
  → 修正：将耗时操作移到 nonisolated 方法中

整个 ViewModel 标记为 @MainActor
  → 问题：所有方法（含数据处理）都在主线程执行
  → 修正：仅对 UI 属性和更新方法标记 @MainActor

忽略 Task 取消检查
  → 问题：已取消的 Task 继续执行无意义的工作
  → 修正：在耗时操作前检查 Task.isCancelled 或使用 try Task.checkCancellation()

async/await 替代回调时未考虑取消语义
  → 问题：旧回调 API 不支持取消，包装后 Task 取消但底层操作仍在执行
  → 修正：使用 withTaskCancellationHandler 将取消传递给底层 API

过度使用 Actor 导致串行瓶颈
  → 问题：高频访问的 Actor 成为性能瓶颈
  → 修正：拆分 Actor 职责、减少 Actor 隔离范围、或使用细粒度锁
```

---

## Swift Concurrency 性能优化 checklist

```
- [ ] 是否避免在循环中创建大量 Task？（使用 TaskGroup）
- [ ] Actor 方法是否避免了耗时的同步操作？
- [ ] 是否处理了 Actor 重入问题（避免重复请求）？
- [ ] @MainActor 标注是否最小化（仅标注 UI 相关）？
- [ ] 可并行的 async 调用是否使用了 async let？
- [ ] 循环中是否避免了频繁的 MainActor 上下文切换？
- [ ] Task 是否正确检查了取消状态？
- [ ] withTaskCancellationHandler 是否用于包装不支持取消的 API？
- [ ] 高频调用的 Actor 是否存在串行瓶颈？
- [ ] Sendable 改造是否避免了不必要的性能损失？
```
