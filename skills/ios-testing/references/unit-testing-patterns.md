# 单元测试模式

## 测试用例设计方法

### 等价类划分

将输入域划分为若干等价类，每类取一个代表值测试。

```swift
// 被测方法：年龄分类
func ageCategory(_ age: Int) -> String

// 等价类划分：
// 无效：age < 0
// 婴幼儿：0-2
// 儿童：3-12
// 青少年：13-17
// 成年：18-59
// 老年：60+

@Test("年龄分类", arguments: [
    (-1, "无效"),
    (0, "婴幼儿"), (1, "婴幼儿"),
    (5, "儿童"), (12, "儿童"),
    (15, "青少年"),
    (25, "成年"),
    (70, "老年"),
])
func ageCategory(age: Int, expected: String) {
    #expect(ageCategory(age) == expected)
}
```

### 边界值分析

重点测试边界点及其相邻值。

```swift
// 被测方法：密码强度校验（8-32 位）
@Suite("密码长度边界测试")
struct PasswordLengthTests {
    @Test("7 位密码太短")
    func tooShort() {
        #expect(PasswordValidator.validate("1234567") == .tooShort)
    }

    @Test("8 位密码刚好合格")
    func minimumLength() {
        #expect(PasswordValidator.validate("12345678") != .tooShort)
    }

    @Test("32 位密码刚好合格")
    func maximumLength() {
        let password = String(repeating: "a", count: 32)
        #expect(PasswordValidator.validate(password) != .tooLong)
    }

    @Test("33 位密码太长")
    func tooLong() {
        let password = String(repeating: "a", count: 33)
        #expect(PasswordValidator.validate(password) == .tooLong)
    }
}
```

### 决策表

多个条件组合时，列出所有条件组合的预期结果。

```swift
// 决策表：订单折扣规则
// | VIP会员 | 订单金额>500 | 有优惠券 | 折扣 |
// |---------|------------|---------|------|
// | 是      | 是         | 是      | 30%  |
// | 是      | 是         | 否      | 20%  |
// | 是      | 否         | 是      | 15%  |
// | 是      | 否         | 否      | 10%  |
// | 否      | 是         | 是      | 15%  |
// | 否      | 是         | 否      | 5%   |
// | 否      | 否         | 是      | 10%  |
// | 否      | 否         | 否      | 0%   |

@Test("订单折扣计算", arguments: [
    (true,  600, true,  0.30),
    (true,  600, false, 0.20),
    (true,  200, true,  0.15),
    (true,  200, false, 0.10),
    (false, 600, true,  0.15),
    (false, 600, false, 0.05),
    (false, 200, true,  0.10),
    (false, 200, false, 0.00),
])
func orderDiscount(isVIP: Bool, amount: Int, hasCoupon: Bool, expectedDiscount: Double) {
    let order = Order(amount: amount, isVIP: isVIP, hasCoupon: hasCoupon)
    #expect(order.calculateDiscount() == expectedDiscount)
}
```

### 状态转换测试

测试对象在不同事件下的状态变化。

```swift
// 订单状态机：pending → paid → shipped → delivered
//                    → cancelled
@Suite("订单状态转换")
struct OrderStateTests {
    @Test("待支付 → 已支付")
    func pendingToPaid() {
        var order = Order(state: .pending)
        order.pay()
        #expect(order.state == .paid)
    }

    @Test("已支付 → 已发货")
    func paidToShipped() {
        var order = Order(state: .paid)
        order.ship()
        #expect(order.state == .shipped)
    }

    @Test("待支付 → 已取消")
    func pendingToCancelled() {
        var order = Order(state: .pending)
        order.cancel()
        #expect(order.state == .cancelled)
    }

    @Test("已发货不能取消")
    func shippedCannotCancel() {
        var order = Order(state: .shipped)
        order.cancel()
        #expect(order.state == .shipped)  // 状态不变
    }
}
```

---

## ViewModel 测试模式

### 输入 → 输出验证

```swift
@Observable
class SearchViewModel {
    var query = ""
    var results: [Product] = []
    var isLoading = false
    var errorMessage: String?

    private let service: SearchServiceProtocol

    init(service: SearchServiceProtocol) {
        self.service = service
    }

    func search() async {
        guard !query.trimmingCharacters(in: .whitespaces).isEmpty else {
            results = []
            return
        }
        isLoading = true
        defer { isLoading = false }
        do {
            results = try await service.search(query: query)
        } catch {
            errorMessage = "搜索失败，请重试"
        }
    }
}

@Suite("搜索 ViewModel")
struct SearchViewModelTests {
    let mockService = MockSearchService()

    @Test("空查询清空结果")
    func emptyQuery() async {
        let vm = SearchViewModel(service: mockService)
        vm.query = "  "
        await vm.search()
        #expect(vm.results.isEmpty)
        #expect(vm.isLoading == false)
    }

    @Test("正常搜索返回结果")
    func normalSearch() async {
        mockService.stubbedResults = [Product.mock]
        let vm = SearchViewModel(service: mockService)
        vm.query = "iPhone"
        await vm.search()
        #expect(vm.results.count == 1)
        #expect(vm.errorMessage == nil)
    }

    @Test("搜索失败显示错误")
    func searchFailure() async {
        mockService.stubbedError = URLError(.notConnectedToInternet)
        let vm = SearchViewModel(service: mockService)
        vm.query = "iPhone"
        await vm.search()
        #expect(vm.results.isEmpty)
        #expect(vm.errorMessage == "搜索失败，请重试")
    }
}
```

### 状态变化验证

```swift
@Test("加载过程中 isLoading 状态正确")
func loadingState() async {
    let mockService = MockProductService()
    // 设置延迟以便观察中间状态
    mockService.delay = .milliseconds(100)
    mockService.stubbedProducts = [Product.mock]

    let vm = ProductListViewModel(service: mockService)
    #expect(vm.isLoading == false)

    let task = Task { await vm.loadProducts() }
    // 给异步操作一点启动时间
    try? await Task.sleep(for: .milliseconds(10))
    #expect(vm.isLoading == true)

    await task.value
    #expect(vm.isLoading == false)
    #expect(vm.products.count == 1)
}
```

### 错误处理验证

```swift
@Suite("ViewModel 错误处理")
struct ErrorHandlingTests {
    @Test("网络错误显示友好提示")
    func networkError() async {
        let mock = MockService()
        mock.stubbedError = URLError(.notConnectedToInternet)
        let vm = DataViewModel(service: mock)

        await vm.loadData()

        #expect(vm.errorMessage == "网络不可用，请检查网络设置")
    }

    @Test("服务器错误显示重试提示")
    func serverError() async {
        let mock = MockService()
        mock.stubbedError = APIError.serverError(500)
        let vm = DataViewModel(service: mock)

        await vm.loadData()

        #expect(vm.errorMessage == "服务器繁忙，请稍后重试")
    }

    @Test("未知错误显示通用提示")
    func unknownError() async {
        let mock = MockService()
        mock.stubbedError = NSError(domain: "test", code: -1)
        let vm = DataViewModel(service: mock)

        await vm.loadData()

        #expect(vm.errorMessage == "操作失败，请重试")
    }
}
```

### 异步操作测试

```swift
@Test("下拉刷新重新加载数据")
func pullToRefresh() async {
    let mock = MockProductService()
    mock.stubbedProducts = [Product(name: "旧数据")]
    let vm = ProductListViewModel(service: mock)

    await vm.loadProducts()
    #expect(vm.products.first?.name == "旧数据")

    // 模拟数据更新
    mock.stubbedProducts = [Product(name: "新数据")]
    await vm.refresh()

    #expect(vm.products.first?.name == "新数据")
    #expect(mock.fetchProductsCallCount == 2)
}
```

---

## 业务逻辑测试模式

### 纯函数测试

```swift
// 纯函数 — 无副作用，输入决定输出
struct PriceCalculator {
    static func finalPrice(original: Decimal, discount: Decimal, tax: Decimal) -> Decimal {
        let discounted = original * (1 - discount)
        return (discounted * (1 + tax)).rounded(scale: 2)
    }
}

@Test("价格计算", arguments: [
    (Decimal(100), Decimal(0.1), Decimal(0.06), Decimal(95.40)),
    (Decimal(0), Decimal(0.5), Decimal(0.06), Decimal(0)),
    (Decimal(999), Decimal(0), Decimal(0), Decimal(999)),
])
func priceCalculation(original: Decimal, discount: Decimal, tax: Decimal, expected: Decimal) {
    let result = PriceCalculator.finalPrice(original: original, discount: discount, tax: tax)
    #expect(result == expected)
}
```

### 有副作用的函数测试

```swift
// 有副作用 — 通过 Mock 验证副作用
class OrderProcessor {
    private let repository: OrderRepositoryProtocol
    private let notifier: NotificationServiceProtocol
    private let analytics: AnalyticsProtocol

    init(repository: OrderRepositoryProtocol,
         notifier: NotificationServiceProtocol,
         analytics: AnalyticsProtocol) {
        self.repository = repository
        self.notifier = notifier
        self.analytics = analytics
    }

    func processOrder(_ order: Order) async throws {
        try await repository.save(order)
        try notifier.send(.orderConfirmed(order))
        analytics.track("order_processed", params: ["orderId": order.id])
    }
}

@Test("处理订单触发保存、通知和埋点")
func processOrder() async throws {
    let mockRepo = MockOrderRepository()
    let mockNotifier = MockNotificationService()
    let mockAnalytics = MockAnalyticsService()
    let processor = OrderProcessor(
        repository: mockRepo, notifier: mockNotifier, analytics: mockAnalytics
    )

    let order = Order(id: "ORD-001", amount: 99)
    try await processor.processOrder(order)

    #expect(mockRepo.savedOrders.contains(where: { $0.id == "ORD-001" }))
    #expect(mockNotifier.sentNotifications.count == 1)
    #expect(mockAnalytics.trackedEvents.first?.name == "order_processed")
}
```

### 并发安全测试（Actor 测试）

```swift
actor CartActor {
    private(set) var items: [CartItem] = []

    func add(_ item: CartItem) {
        items.append(item)
    }

    func remove(id: String) {
        items.removeAll { $0.id == id }
    }

    var totalPrice: Decimal {
        items.reduce(0) { $0 + $1.price * Decimal($1.quantity) }
    }
}

@Test("并发添加商品不丢失数据")
func concurrentAdd() async {
    let cart = CartActor()
    await withTaskGroup(of: Void.self) { group in
        for i in 0..<50 {
            group.addTask {
                await cart.add(CartItem(id: "item-\(i)", price: 10, quantity: 1))
            }
        }
    }
    let count = await cart.items.count
    #expect(count == 50)
}

@Test("并发读写安全")
func concurrentReadWrite() async {
    let cart = CartActor()
    await withTaskGroup(of: Void.self) { group in
        // 写操作
        for i in 0..<20 {
            group.addTask {
                await cart.add(CartItem(id: "item-\(i)", price: 10, quantity: 1))
            }
        }
        // 同时读操作
        for _ in 0..<20 {
            group.addTask {
                _ = await cart.totalPrice
            }
        }
    }
    let count = await cart.items.count
    #expect(count == 20)
}
```

---

## 数据层测试模式

### Codable 模型测试

```swift
@Suite("用户模型解析")
struct UserDecodingTests {
    @Test("完整 JSON 解析成功")
    func fullJSON() throws {
        let json = """
        {"id": "123", "name": "张三", "email": "zhang@example.com", "age": 25}
        """.data(using: .utf8)!

        let user = try JSONDecoder().decode(User.self, from: json)
        #expect(user.id == "123")
        #expect(user.name == "张三")
        #expect(user.email == "zhang@example.com")
        #expect(user.age == 25)
    }

    @Test("缺少可选字段解析成功")
    func missingOptionalFields() throws {
        let json = """
        {"id": "123", "name": "张三"}
        """.data(using: .utf8)!

        let user = try JSONDecoder().decode(User.self, from: json)
        #expect(user.email == nil)
        #expect(user.age == nil)
    }

    @Test("缺少必需字段解析失败")
    func missingRequiredFields() {
        let json = """
        {"name": "张三"}
        """.data(using: .utf8)!

        #expect(throws: DecodingError.self) {
            try JSONDecoder().decode(User.self, from: json)
        }
    }

    @Test("编码后再解码数据一致")
    func roundTrip() throws {
        let original = User(id: "123", name: "张三", email: "zhang@example.com", age: 25)
        let data = try JSONEncoder().encode(original)
        let decoded = try JSONDecoder().decode(User.self, from: data)
        #expect(decoded == original)
    }
}
```

### Repository 测试

```swift
@Suite("用户仓库")
struct UserRepositoryTests {
    let storage = InMemoryUserStorage()
    var repository: UserRepository { UserRepository(storage: storage) }

    @Test("保存并读取用户")
    func saveAndFetch() throws {
        let user = User(id: "1", name: "张三")
        try repository.save(user)

        let fetched = repository.fetch(id: "1")
        #expect(fetched?.name == "张三")
    }

    @Test("更新用户信息")
    func update() throws {
        try repository.save(User(id: "1", name: "张三"))
        try repository.save(User(id: "1", name: "李四"))

        let fetched = repository.fetch(id: "1")
        #expect(fetched?.name == "李四")
    }

    @Test("删除用户")
    func delete() throws {
        try repository.save(User(id: "1", name: "张三"))
        try repository.delete(id: "1")
        #expect(repository.fetch(id: "1") == nil)
    }

    @Test("查询不存在的用户返回 nil")
    func fetchNonExistent() {
        #expect(repository.fetch(id: "non-existent") == nil)
    }
}
```

### Core Data / SwiftData 测试

```swift
// Core Data 内存测试配置
extension NSPersistentContainer {
    static func inMemory(name: String = "Model") -> NSPersistentContainer {
        let container = NSPersistentContainer(name: name)
        let description = NSPersistentStoreDescription()
        description.type = NSInMemoryStoreType
        container.persistentStoreDescriptions = [description]
        container.loadPersistentStores { _, error in
            if let error { fatalError("Core Data 内存存储加载失败: \(error)") }
        }
        return container
    }
}

// SwiftData 内存测试配置
@Suite("SwiftData 存储测试")
struct SwiftDataTests {
    let container: ModelContainer

    init() throws {
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        container = try ModelContainer(for: UserModel.self, configurations: config)
    }

    @Test("保存和查询")
    @MainActor
    func saveAndQuery() throws {
        let context = container.mainContext
        let user = UserModel(name: "张三", age: 25)
        context.insert(user)
        try context.save()

        let descriptor = FetchDescriptor<UserModel>(
            predicate: #Predicate { $0.name == "张三" }
        )
        let results = try context.fetch(descriptor)
        #expect(results.count == 1)
        #expect(results.first?.age == 25)
    }
}
```

---

## 测试数据管理

### Factory Pattern

```swift
// 工厂方法提供带默认值的测试数据
enum UserFactory {
    static func make(
        id: String = "user-001",
        name: String = "测试用户",
        email: String = "test@example.com",
        age: Int = 25,
        isVIP: Bool = false
    ) -> User {
        User(id: id, name: name, email: email, age: age, isVIP: isVIP)
    }

    static func makeVIP() -> User {
        make(name: "VIP 用户", isVIP: true)
    }

    static func makeList(count: Int = 5) -> [User] {
        (0..<count).map { i in make(id: "user-\(i)", name: "用户\(i)") }
    }
}

// 使用
@Test("VIP 用户享受折扣")
func vipDiscount() {
    let user = UserFactory.makeVIP()
    let discount = DiscountService.calculate(for: user)
    #expect(discount > 0)
}
```

### Builder Pattern

```swift
// Builder 适合复杂对象的灵活构造
class OrderBuilder {
    private var id = "order-001"
    private var items: [OrderItem] = []
    private var customer = UserFactory.make()
    private var coupon: Coupon?
    private var state: OrderState = .pending

    func with(id: String) -> OrderBuilder { self.id = id; return self }
    func with(items: [OrderItem]) -> OrderBuilder { self.items = items; return self }
    func with(customer: User) -> OrderBuilder { self.customer = customer; return self }
    func with(coupon: Coupon) -> OrderBuilder { self.coupon = coupon; return self }
    func with(state: OrderState) -> OrderBuilder { self.state = state; return self }

    func build() -> Order {
        Order(id: id, items: items, customer: customer, coupon: coupon, state: state)
    }
}

// 使用
@Test("已支付订单可以发货")
func paidOrderCanShip() {
    let order = OrderBuilder()
        .with(state: .paid)
        .with(items: [OrderItem(product: "iPhone", quantity: 1)])
        .build()
    #expect(order.canShip)
}
```

### JSON Fixture 文件

```swift
// 从 Bundle 加载 JSON fixture
extension Bundle {
    func loadFixture<T: Decodable>(_ filename: String, as type: T.Type) throws -> T {
        guard let url = self.url(forResource: filename, withExtension: "json") else {
            throw FixtureError.fileNotFound(filename)
        }
        let data = try Data(contentsOf: url)
        return try JSONDecoder().decode(type, from: data)
    }
}

// 放在 Tests/Resources/Fixtures/ 目录下
// user_response.json, products_response.json 等

@Test("解析真实 API 响应格式")
func parseRealResponse() throws {
    let response: APIResponse<User> = try Bundle.testBundle.loadFixture(
        "user_response", as: APIResponse<User>.self
    )
    #expect(response.data.name == "张三")
    #expect(response.code == 200)
}
```

---

## 速查 Checklist

> 完整的测试质量标准见 `test-quality-checklist.md`，此处仅列单元测试模式专项检查。

- [ ] 每个测试遵循 Arrange-Act-Assert 三段式结构
- [ ] 测试覆盖正常路径、边界条件和异常路径
- [ ] ViewModel 测试验证输入输出，不验证内部实现
- [ ] 纯函数使用参数化测试覆盖多组输入
- [ ] 副作用函数通过 Mock 验证交互结果
- [ ] Codable 模型测试覆盖完整/缺少字段/无效数据
- [ ] Actor 测试覆盖并发场景
- [ ] 测试数据使用 Factory/Builder，不在测试中硬编码复杂对象
- [ ] JSON fixture 文件与真实 API 响应格式一致
- [ ] Core Data/SwiftData 测试使用 in-memory 配置
