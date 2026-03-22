# Swift Concurrency 测试模式

## Combine Publisher 测试

### 基本 Publisher 测试

```swift
import Combine
import XCTest
@testable import MyApp

final class PublisherTests: XCTestCase {
    private var cancellables: Set<AnyCancellable> = []

    override func tearDown() {
        cancellables.removeAll()
        super.tearDown()
    }

    // 使用 values 属性收集 AsyncSequence（Combine + Swift Concurrency）
    func test_fetchUser_emitsCorrectValues() async {
        let service = MockUserService()
        service.stubbedUser = User(name: "张三")

        var receivedValues: [User] = []
        let expectation = expectation(description: "收到用户数据")

        service.userPublisher
            .sink(
                receiveCompletion: { _ in expectation.fulfill() },
                receiveValue: { receivedValues.append($0) }
            )
            .store(in: &cancellables)

        service.fetchUser(id: "1")
        await fulfillment(of: [expectation], timeout: 2)

        XCTAssertEqual(receivedValues.count, 1)
        XCTAssertEqual(receivedValues.first?.name, "张三")
    }

    // 测试 Publisher 的错误传播
    func test_fetchUser_propagatesError() {
        let service = MockUserService()
        service.stubbedError = APIError.notFound

        let expectation = expectation(description: "收到错误")
        var receivedError: Error?

        service.userPublisher
            .sink(
                receiveCompletion: { completion in
                    if case .failure(let error) = completion {
                        receivedError = error
                        expectation.fulfill()
                    }
                },
                receiveValue: { _ in }
            )
            .store(in: &cancellables)

        service.fetchUser(id: "invalid")
        waitForExpectations(timeout: 2)

        XCTAssertTrue(receivedError is APIError)
    }
}
```

### 使用 collect 收集多个值

```swift
func test_searchSuggestions_emitsMultipleResults() {
    let service = MockSearchService()
    let expectation = expectation(description: "收集完成")
    var collectedResults: [[String]] = []

    service.suggestionsPublisher
        .collect(3)  // 收集 3 个值后完成
        .sink(
            receiveCompletion: { _ in expectation.fulfill() },
            receiveValue: { collectedResults = $0 }
        )
        .store(in: &cancellables)

    service.emitSuggestions(["苹果"])
    service.emitSuggestions(["苹果手机", "苹果电脑"])
    service.emitSuggestions(["苹果手机", "苹果电脑", "苹果耳机"])

    waitForExpectations(timeout: 2)
    XCTAssertEqual(collectedResults.count, 3)
}
```

### 使用 first 只取第一个值

```swift
func test_locationUpdate_firstValue() {
    let service = MockLocationService()
    let expectation = expectation(description: "收到首次定位")
    var firstLocation: CLLocation?

    service.locationPublisher
        .first()
        .sink(
            receiveCompletion: { _ in expectation.fulfill() },
            receiveValue: { firstLocation = $0 }
        )
        .store(in: &cancellables)

    service.emitLocation(CLLocation(latitude: 31.23, longitude: 121.47))
    service.emitLocation(CLLocation(latitude: 39.90, longitude: 116.40))

    waitForExpectations(timeout: 2)
    XCTAssertEqual(firstLocation?.coordinate.latitude, 31.23, accuracy: 0.01)
}
```

---

## AsyncSequence 测试

```swift
// 被测代码：异步序列生产者
struct NumberSequence: AsyncSequence {
    typealias Element = Int
    let range: Range<Int>

    struct AsyncIterator: AsyncIteratorProtocol {
        var current: Int
        let end: Int
        mutating func next() async -> Int? {
            guard current < end else { return nil }
            defer { current += 1 }
            return current
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(current: range.lowerBound, end: range.upperBound)
    }
}

// 测试 AsyncSequence 输出
@Test("AsyncSequence 产生正确的值序列")
func asyncSequenceValues() async {
    let sequence = NumberSequence(range: 0..<5)
    var collected: [Int] = []
    for await value in sequence {
        collected.append(value)
    }
    #expect(collected == [0, 1, 2, 3, 4])
}

// 测试带过滤的 AsyncSequence
@Test("AsyncSequence 过滤偶数")
func asyncSequenceFilter() async {
    let sequence = NumberSequence(range: 0..<10)
    var evens: [Int] = []
    for await value in sequence where value.isMultiple(of: 2) {
        evens.append(value)
    }
    #expect(evens == [0, 2, 4, 6, 8])
}
```

---

## AsyncStream 测试

```swift
// 被测代码：使用 AsyncStream 的事件监听器
class EventMonitor {
    private var continuation: AsyncStream<AppEvent>.Continuation?

    var events: AsyncStream<AppEvent> {
        AsyncStream { continuation in
            self.continuation = continuation
        }
    }

    func emit(_ event: AppEvent) {
        continuation?.yield(event)
    }

    func stop() {
        continuation?.finish()
    }
}

// 测试 AsyncStream 的值发射
@Test("AsyncStream 正确接收事件")
func asyncStreamReceivesEvents() async {
    let monitor = EventMonitor()
    var received: [AppEvent] = []

    // 在后台消费事件
    let task = Task {
        for await event in monitor.events {
            received.append(event)
        }
    }

    // 发射事件
    monitor.emit(.userLogin(userId: "001"))
    monitor.emit(.pageView(page: "home"))
    monitor.stop()  // 结束流

    await task.value  // 等待消费完成
    #expect(received.count == 2)
    #expect(received[0] == .userLogin(userId: "001"))
}

// 测试 AsyncStream 的取消行为
@Test("AsyncStream 支持取消")
func asyncStreamCancellation() async throws {
    let monitor = EventMonitor()
    var received: [AppEvent] = []

    let task = Task {
        for await event in monitor.events {
            received.append(event)
        }
    }

    monitor.emit(.pageView(page: "home"))
    // 给一点时间让事件被处理
    try await Task.sleep(for: .milliseconds(50))

    task.cancel()  // 取消消费
    monitor.emit(.pageView(page: "settings"))  // 取消后的事件不应被收到

    try await Task.sleep(for: .milliseconds(50))
    #expect(received.count == 1)
}
```

---

## Actor 方法测试

```swift
// 被测 Actor
actor BankAccount {
    private(set) var balance: Decimal

    init(balance: Decimal) {
        self.balance = balance
    }

    func deposit(_ amount: Decimal) throws {
        guard amount > 0 else { throw BankError.invalidAmount }
        balance += amount
    }

    func withdraw(_ amount: Decimal) throws {
        guard amount > 0 else { throw BankError.invalidAmount }
        guard balance >= amount else { throw BankError.insufficientFunds }
        balance -= amount
    }
}

// Actor 基本方法测试
@Test("存款增加余额")
func deposit() async throws {
    let account = BankAccount(balance: 1000)
    try await account.deposit(500)
    let balance = await account.balance
    #expect(balance == 1500)
}

@Test("余额不足时取款失败")
func withdrawInsufficientFunds() async {
    let account = BankAccount(balance: 100)
    await #expect(throws: BankError.insufficientFunds) {
        try await account.withdraw(200)
    }
}

// Actor 并发安全测试
@Test("并发存取款保持数据一致")
func concurrentOperations() async throws {
    let account = BankAccount(balance: 10000)

    await withTaskGroup(of: Void.self) { group in
        // 100 次存款，每次 100
        for _ in 0..<100 {
            group.addTask { try? await account.deposit(100) }
        }
        // 100 次取款，每次 50
        for _ in 0..<100 {
            group.addTask { try? await account.withdraw(50) }
        }
    }

    // 10000 + (100 * 100) - (100 * 50) = 15000
    let finalBalance = await account.balance
    #expect(finalBalance == 15000)
}
```

---

## Task 取消测试

```swift
// 被测代码：支持取消的异步操作
class DataLoader {
    func loadLargeDataset() async throws -> [DataItem] {
        var items: [DataItem] = []
        for i in 0..<1000 {
            // 检查取消状态
            try Task.checkCancellation()
            items.append(DataItem(id: i))
        }
        return items
    }
}

// 测试正常完成
@Test("完整加载数据集")
func loadComplete() async throws {
    let loader = DataLoader()
    let items = try await loader.loadLargeDataset()
    #expect(items.count == 1000)
}

// 测试取消行为
@Test("取消后抛出 CancellationError")
func cancellation() async {
    let loader = DataLoader()

    let task = Task {
        try await loader.loadLargeDataset()
    }

    // 立即取消
    task.cancel()

    do {
        _ = try await task.value
        Issue.record("应该抛出 CancellationError")
    } catch {
        #expect(error is CancellationError)
    }
}

// 测试取消后的清理逻辑
@Test("取消后正确释放资源")
func cancellationCleanup() async throws {
    let resourceTracker = ResourceTracker()
    let service = DataService(resourceTracker: resourceTracker)

    let task = Task {
        try await service.processData()
    }

    try await Task.sleep(for: .milliseconds(50))
    task.cancel()

    // 等待清理完成
    try await Task.sleep(for: .milliseconds(100))
    let activeResources = await resourceTracker.activeCount
    #expect(activeResources == 0)
}
```

---

## 超时处理

### Swift Testing confirmation 超时

```swift
// 使用 confirmation 验证异步回调是否被调用
@Test("通知在 2 秒内发送", .timeLimit(.seconds(5)))
func notificationSentInTime() async throws {
    let service = NotificationService(sender: MockSender())

    // confirmation 默认 timeout 继承 .timeLimit
    await confirmation("收到通知") { confirm in
        service.onNotificationSent = { confirm() }
        await service.sendNotification(to: "user-001", message: "你好")
    }
}

// 验证回调被调用指定次数
@Test("批量操作触发 3 次进度回调")
func batchProgressCallbacks() async {
    let processor = BatchProcessor()

    await confirmation("进度回调", expectedCount: 3) { confirm in
        processor.onProgress = { _ in confirm() }
        await processor.process(items: ["a", "b", "c"])
    }
}
```

### XCTest 超时处理

```swift
// XCTestExpectation 超时
func test_asyncOperation_completesWithinTimeout() {
    let expectation = expectation(description: "操作完成")
    let service = AsyncService()

    service.performOperation { result in
        XCTAssertNotNil(result)
        expectation.fulfill()
    }

    waitForExpectations(timeout: 3) { error in
        if error != nil {
            XCTFail("操作超时：3 秒内未完成")
        }
    }
}
```

---

## 常见异步测试陷阱

### 1. 忘记 await 导致断言在异步操作前执行

```swift
// 错误：断言在 loadData 完成前执行
@Test("错误示例")
func badAsyncTest() {
    let vm = ViewModel(service: MockService())
    Task { await vm.loadData() }
    #expect(vm.items.count == 3)  // 此时 loadData 可能还没执行完
}

// 正确：使用 await 等待完成
@Test("正确示例")
func goodAsyncTest() async {
    let vm = ViewModel(service: MockService())
    await vm.loadData()
    #expect(vm.items.count == 3)
}
```

### 2. 在非隔离上下文中访问 MainActor 属性

```swift
// 错误：非 MainActor 测试访问 @MainActor 属性
@Test("错误示例")
func badMainActorTest() async {
    let vm = MainActorViewModel()
    await vm.loadData()
    // 编译警告/错误：从非 MainActor 上下文访问
    // #expect(vm.title == "首页")
}

// 正确：将测试标记为 @MainActor
@Test("正确示例")
@MainActor
func goodMainActorTest() async {
    let vm = MainActorViewModel()
    await vm.loadData()
    #expect(vm.title == "首页")
}
```

### 3. 使用 sleep 代替正确的等待机制

```swift
// 错误：使用 sleep 硬等待
@Test("错误示例")
func badWaitTest() async throws {
    let vm = ViewModel(service: MockService())
    await vm.loadData()
    try await Task.sleep(for: .seconds(2))  // 不要用 sleep 等待
    #expect(vm.isReady)
}

// 正确：让 Mock 控制异步行为，直接 await 结果
@Test("正确示例")
func goodWaitTest() async {
    let mock = MockService()
    mock.stubbedItems = [Item.mock]
    let vm = ViewModel(service: mock)
    await vm.loadData()
    #expect(vm.isReady)  // Mock 是同步返回的，无需额外等待
}
```

### 4. 异步测试中的资源泄漏

```swift
// 错误：Task 没有等待完成或取消
@Test("错误示例")
func leakyTest() async {
    let vm = ViewModel(service: MockService())
    Task { await vm.loadData() }  // 这个 Task 没人管
    // 测试结束时 Task 可能还在跑
}

// 正确：持有 Task 引用并等待完成
@Test("正确示例")
func cleanTest() async {
    let vm = ViewModel(service: MockService())
    let task = Task { await vm.loadData() }
    await task.value  // 等待完成
    #expect(vm.items.isEmpty == false)
}
```

### 5. 未处理 Task Group 中的错误

```swift
// 错误：TaskGroup 中的错误被静默吞掉
@Test("错误示例")
func silentErrorTest() async {
    await withTaskGroup(of: Void.self) { group in
        group.addTask {
            try? await riskyOperation()  // try? 吞掉了错误
        }
    }
    // 无法知道是否有错误发生
}

// 正确：使用 ThrowingTaskGroup 传播错误
@Test("正确示例")
func properErrorTest() async throws {
    try await withThrowingTaskGroup(of: Void.self) { group in
        group.addTask {
            try await riskyOperation()
        }
        try await group.waitForAll()
    }
}
```

---

## 速查 Checklist

> 完整的测试质量标准见 `test-quality-checklist.md`，此处仅列异步测试专项检查。

- [ ] 异步测试方法标记为 `async` / `async throws`
- [ ] 使用 `await` 等待异步操作完成，不使用 `sleep`
- [ ] `@MainActor` 属性在 `@MainActor` 上下文中访问
- [ ] Task 引用被正确持有，完成后 `await task.value`
- [ ] Actor 测试覆盖了并发场景（多 Task 同时操作）
- [ ] Task 取消测试验证了 `CancellationError` 和资源清理
- [ ] Combine Publisher 测试使用 `expectation` 等待完成
- [ ] AsyncStream 测试在结束时调用 `finish()` 避免挂起
- [ ] 异步 Mock 控制返回时机，不依赖真实延迟
- [ ] ThrowingTaskGroup 用于需要传播错误的并发场景
