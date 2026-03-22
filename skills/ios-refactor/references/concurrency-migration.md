# Swift Concurrency 迁移

## 迁移路线概览

```
迁移阶段：
├── 1. 启用 Strict Concurrency Checking（先 Targeted，再 Complete）
├── 2. completion handler → async/await
├── 3. Delegate → AsyncSequence / AsyncStream
├── 4. GCD (DispatchQueue) → structured concurrency
├── 5. @MainActor 标注与 Sendable 合规
└── 6. 启用 Swift 6 language mode
```

---

## completion handler → async/await

### 基本迁移模式

```swift
// === Before：completion handler ===
func fetchUser(id: String, completion: @escaping (Result<User, Error>) -> Void) {
    URLSession.shared.dataTask(with: makeRequest(id: id)) { data, response, error in
        if let error = error {
            completion(.failure(error))
            return
        }
        guard let data = data else {
            completion(.failure(APIError.noData))
            return
        }
        do {
            let user = try JSONDecoder().decode(User.self, from: data)
            completion(.success(user))
        } catch {
            completion(.failure(error))
        }
    }.resume()
}

// === After：async/await ===
func fetchUser(id: String) async throws -> User {
    let (data, _) = try await URLSession.shared.data(for: makeRequest(id: id))
    return try JSONDecoder().decode(User.self, from: data)
}
```

### 渐进式迁移：同时保留两个版本

```swift
// 新增 async 版本，旧版本调用新版本
// 在所有调用方迁移完毕后删除旧版本
func fetchUser(id: String) async throws -> User {
    let (data, _) = try await URLSession.shared.data(for: makeRequest(id: id))
    return try JSONDecoder().decode(User.self, from: data)
}

// 旧版本作为桥接，标记为 deprecated 提醒迁移
@available(*, deprecated, message: "请使用 async 版本")
func fetchUser(id: String, completion: @escaping (Result<User, Error>) -> Void) {
    Task {
        do {
            let user = try await fetchUser(id: id)
            completion(.success(user))
        } catch {
            completion(.failure(error))
        }
    }
}
```

---

## withCheckedContinuation 桥接

### 桥接不抛错误的回调

```swift
// 将 completion handler 桥接为 async
func loadImage(url: URL) async -> UIImage? {
    await withCheckedContinuation { continuation in
        ImageLoader.shared.load(url: url) { image in
            continuation.resume(returning: image)
        }
        // 注意：continuation 必须恰好被 resume 一次
    }
}
```

### 桥接可能抛错误的回调

```swift
func fetchData(from url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                continuation.resume(throwing: error)
                return
            }
            guard let data = data else {
                continuation.resume(throwing: APIError.noData)
                return
            }
            continuation.resume(returning: data)
        }.resume()
    }
}
```

### 桥接注意事项

```
关键规则：
├── continuation 必须恰好被 resume 一次
│   ├── 不 resume → 任务永远挂起（内存泄漏）
│   └── resume 多次 → 运行时崩溃
├── 调试期使用 withCheckedContinuation / withCheckedThrowingContinuation
│   └── 会在违规时给出诊断信息
├── 发布时可切换为 withUnsafeContinuation / withUnsafeThrowingContinuation
│   └── 性能更好但无安全检查
└── 确保回调的所有分支都调用了 resume
```

---

## Delegate → AsyncSequence / AsyncStream

### 基本迁移模式

```swift
// === Before：Delegate 模式 ===
protocol LocationManagerDelegate: AnyObject {
    func locationManager(_ manager: LocationManager, didUpdateLocation location: CLLocation)
    func locationManager(_ manager: LocationManager, didFailWithError error: Error)
}

class LocationManager {
    weak var delegate: LocationManagerDelegate?
    // ...
}

// === After：AsyncStream 模式 ===
class LocationManager {
    // 使用 AsyncStream 提供位置更新的异步序列
    func locationUpdates() -> AsyncThrowingStream<CLLocation, Error> {
        AsyncThrowingStream { continuation in
            let delegate = StreamDelegate(continuation: continuation)
            self.internalDelegate = delegate
            self.startUpdating()

            continuation.onTermination = { @Sendable _ in
                self.stopUpdating()
            }
        }
    }

    private class StreamDelegate: NSObject, CLLocationManagerDelegate {
        let continuation: AsyncThrowingStream<CLLocation, Error>.Continuation

        init(continuation: AsyncThrowingStream<CLLocation, Error>.Continuation) {
            self.continuation = continuation
        }

        func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
            for location in locations {
                continuation.yield(location)
            }
        }

        func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
            continuation.finish(throwing: error)
        }
    }
}

// 消费方
func trackUserLocation() async throws {
    for try await location in locationManager.locationUpdates() {
        updateMap(with: location)
    }
}
```

### AsyncStream 用于一次性事件

```swift
// 包装只需要单次结果的 delegate
func requestAuthorization() async -> CLAuthorizationStatus {
    await withCheckedContinuation { continuation in
        let delegate = AuthDelegate { status in
            continuation.resume(returning: status)
        }
        self.authDelegate = delegate
        self.clManager.requestWhenInUseAuthorization()
    }
}
```

---

## GCD (DispatchQueue) → structured concurrency

### 常见模式对照

```swift
// === Before：DispatchQueue.main.async ===
DispatchQueue.main.async {
    self.label.text = newText
}

// === After：@MainActor ===
await MainActor.run {
    self.label.text = newText
}
// 或在 @MainActor 标注的方法/类型中直接使用
@MainActor
func updateLabel(_ text: String) {
    label.text = text
}

// === Before：DispatchQueue.global().async ===
DispatchQueue.global(qos: .userInitiated).async {
    let result = self.heavyComputation()
    DispatchQueue.main.async {
        self.display(result)
    }
}

// === After：Task + @MainActor ===
Task {
    let result = await heavyComputation()
    await MainActor.run {
        display(result)
    }
}

// === Before：DispatchGroup ===
let group = DispatchGroup()
var results: [Data] = []

for url in urls {
    group.enter()
    fetchData(from: url) { data in
        results.append(data)
        group.leave()
    }
}
group.notify(queue: .main) {
    self.process(results)
}

// === After：TaskGroup ===
let results = await withTaskGroup(of: Data.self) { group in
    for url in urls {
        group.addTask {
            await self.fetchData(from: url)
        }
    }
    var collected: [Data] = []
    for await result in group {
        collected.append(result)
    }
    return collected
}
process(results)

// === Before：DispatchSemaphore（限制并发数）===
let semaphore = DispatchSemaphore(value: 3)
for url in urls {
    semaphore.wait()
    DispatchQueue.global().async {
        defer { semaphore.signal() }
        self.download(url)
    }
}

// === After：TaskGroup + 手动限制 ===
await withTaskGroup(of: Void.self) { group in
    var active = 0
    for url in urls {
        if active >= 3 {
            await group.next() // 等待一个任务完成
            active -= 1
        }
        group.addTask { await self.download(url) }
        active += 1
    }
}

// === Before：DispatchQueue 作串行队列保护状态 ===
class DataStore {
    private let queue = DispatchQueue(label: "com.app.datastore")
    private var _items: [Item] = []

    var items: [Item] {
        queue.sync { _items }
    }

    func addItem(_ item: Item) {
        queue.async { self._items.append(item) }
    }
}

// === After：Actor ===
actor DataStore {
    private var items: [Item] = []

    func getItems() -> [Item] { items }
    func addItem(_ item: Item) { items.append(item) }
}
```

---

## @MainActor 标注策略

### 标注层级选择

```swift
// 策略一：类型级标注（推荐用于 ViewModel 和 UI 相关类型）
@MainActor
class ProfileViewModel: ObservableObject {
    @Published var userName: String = ""
    @Published var isLoading = false

    func loadProfile() async {
        isLoading = true
        defer { isLoading = false }
        // 网络请求会自动在后台执行，结果回到主线程
        userName = try await userService.fetchProfile().name
    }
}

// 策略二：方法级标注（用于部分方法需要主线程的类型）
class ImageProcessor {
    // 只有 UI 更新方法标注 @MainActor
    @MainActor
    func displayResult(_ image: UIImage) {
        imageView.image = image
    }

    // 计算方法不需要主线程
    func processImage(_ data: Data) async -> UIImage {
        // 在后台处理
    }
}

// 策略三：属性级标注
class DataManager {
    @MainActor var displayItems: [Item] = []

    func loadItems() async {
        let items = await fetchFromNetwork()
        await MainActor.run {
            displayItems = items
        }
    }
}
```

### 标注策略建议

```
@MainActor 标注优先级：
├── UIViewController 子类 → 类型级 @MainActor
├── ViewModel / ObservableObject → 类型级 @MainActor
├── SwiftUI View → 已隐式 @MainActor（无需标注）
├── Service / Repository → 不标注（除非有 UI 耦合）
├── 纯计算方法 → 标注 nonisolated 以允许后台调用
└── 混合类型 → 方法级标注，仅标注需要主线程的方法
```

---

## Sendable 合规迁移

### Sendable 基本概念

```swift
// Sendable = 可以安全跨并发域传递的类型
// 值类型（struct, enum）自动满足 Sendable（如果成员也是 Sendable）
struct UserProfile: Sendable {
    let name: String
    let age: Int
}

// 引用类型需要显式合规
final class Configuration: Sendable {
    let apiKey: String  // 所有存储属性必须是 let + Sendable
    let baseURL: URL

    init(apiKey: String, baseURL: URL) {
        self.apiKey = apiKey
        self.baseURL = baseURL
    }
}

// 无法满足 Sendable 的类型 → 使用 @unchecked Sendable
// 仅在确认线程安全时使用（如内部有锁保护）
final class ThreadSafeCache: @unchecked Sendable {
    private let lock = NSLock()
    private var storage: [String: Any] = [:]

    func set(_ value: Any, forKey key: String) {
        lock.lock()
        defer { lock.unlock() }
        storage[key] = value
    }
}
```

### 常见 Sendable 问题修复

```swift
// 问题：闭包捕获非 Sendable 类型
// 错误：Capture of 'viewModel' with non-sendable type 'ViewModel' in @Sendable closure
Task {
    await viewModel.load() // ViewModel 不是 Sendable
}

// 修复方案一：将 ViewModel 标记为 @MainActor
@MainActor
class ViewModel: ObservableObject { ... }

// 修复方案二：对必须跨域传递的数据使用 Sendable 类型
struct ViewState: Sendable { ... }

// 问题：Delegate 跨隔离域回调
// 修复：将回调通过 @MainActor 限定
@MainActor
protocol ViewModelDelegate: AnyObject {
    func viewModelDidUpdate(_ viewModel: ViewModel)
}
```

---

## Swift 6 Strict Concurrency 启用步骤

### 分阶段启用

```
阶段一：Targeted Checking（最小影响）
  Xcode → Build Settings → Strict Concurrency Checking → Targeted
  - 仅对显式标注了并发的代码进行检查
  - 修复出现的警告

阶段二：Complete Checking（全面检查）
  Xcode → Build Settings → Strict Concurrency Checking → Complete
  - 对所有代码进行 Sendable 和隔离检查
  - 逐模块修复警告

阶段三：Swift 6 Language Mode
  Xcode → Build Settings → Swift Language Version → 6
  - 所有并发警告升级为编译错误
  - 确保零警告后再启用

建议策略：
  ├── 先对依赖最少的底层模块启用
  ├── 逐步向上层模块推进
  ├── 使用 @preconcurrency import 暂时抑制第三方库警告
  └── 每个模块单独一个 PR，便于 review
```

### 具体 Build Settings 配置

```
// Package.swift 中启用
.target(
    name: "MyModule",
    swiftSettings: [
        // 阶段二：Complete Checking
        .enableExperimentalFeature("StrictConcurrency"),
        // 阶段三：Swift 6 模式
        // .swiftLanguageMode(.v6)
    ]
)

// 对第三方模块暂时抑制检查
@preconcurrency import ThirdPartySDK
```

---

## 常见迁移陷阱

### 1. 在 Actor 中调用同步代码

```swift
actor DatabaseActor {
    func save(_ item: Item) {
        // 陷阱：同步调用外部非隔离方法可能意外跑在 actor 的执行器上
        LegacyDatabase.shared.write(item)
        // 如果 write() 内部使用了 DispatchQueue.sync → 可能死锁

        // 正确做法：将同步阻塞操作移到 actor 外执行
    }
}

// 修复：使用 nonisolated 或在 Task 外执行
actor DatabaseActor {
    nonisolated func save(_ item: Item) async {
        // 不在 actor 的串行执行器上运行，避免死锁
        LegacyDatabase.shared.write(item)
    }
}
```

### 2. Task 不继承 actor 上下文

```swift
@MainActor
class ViewModel {
    func load() {
        // Task {} 继承当前 actor 上下文 → 在 MainActor 上执行
        Task {
            await fetchData() // 仍在 MainActor 上
        }

        // Task.detached {} 不继承上下文 → 在后台执行
        Task.detached {
            await self.fetchData() // 需要显式 self，不在 MainActor 上
            await MainActor.run {
                self.updateUI() // 需要手动回到主线程
            }
        }
    }
}
```

### 3. AsyncStream 的生命周期管理

```swift
// 陷阱：忘记处理 onTermination 导致资源泄漏
let stream = AsyncStream<Data> { continuation in
    let listener = NetworkListener()
    listener.onData = { data in
        continuation.yield(data)
    }
    // 错误：忘记在流终止时清理 listener
    // 正确：
    continuation.onTermination = { @Sendable _ in
        listener.stop()
    }
}
```

### 4. 跨隔离域的可变状态

```swift
// 陷阱：在 @Sendable 闭包中捕获可变变量
var count = 0
Task { @Sendable in
    count += 1 // 编译错误：Mutation of captured var 'count' in concurrently-executing code
}

// 修复：使用 actor 管理可变状态
actor Counter {
    var count = 0
    func increment() { count += 1 }
}
```

---

## 迁移 Checklist

```
准备阶段：
- [ ] 启用 Strict Concurrency Checking = Targeted
- [ ] 盘点项目中所有 GCD / completion handler / Delegate 回调
- [ ] 确定迁移优先级（从叶子模块开始，向上推进）
- [ ] 团队了解 async/await、Actor、Sendable 基本概念

逐步迁移：
- [ ] completion handler → async/await（使用 withCheckedContinuation 桥接）
- [ ] Delegate 多次回调 → AsyncStream
- [ ] DispatchQueue.main.async → @MainActor
- [ ] DispatchQueue.global().async → Task / TaskGroup
- [ ] DispatchQueue 串行队列（保护状态） → Actor
- [ ] DispatchGroup → TaskGroup
- [ ] DispatchSemaphore → TaskGroup + 手动限制并发数

Sendable 合规：
- [ ] 值类型确认成员均为 Sendable
- [ ] 不可变引用类型标记为 final class: Sendable
- [ ] 内部有锁保护的类型使用 @unchecked Sendable
- [ ] ViewModel / UIViewController 标注 @MainActor
- [ ] @Sendable 闭包不捕获可变变量
- [ ] 第三方库使用 @preconcurrency import 暂时抑制警告

验证与推进：
- [ ] 将 Strict Concurrency Checking 升级为 Complete
- [ ] 修复所有并发警告至零
- [ ] 单元测试在 async 上下文中通过
- [ ] 无死锁（特别是 actor + 同步代码的交互）
- [ ] 无 Task 泄漏（确认 AsyncStream 正确终止）
- [ ] 评估是否可启用 Swift 6 language mode
```
