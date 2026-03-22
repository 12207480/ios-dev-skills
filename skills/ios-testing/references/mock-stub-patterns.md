# Mock 与 Stub 模式

## Mock vs Stub vs Spy vs Fake 概念区分

| 类型 | 目的 | 行为 | 适用场景 |
|------|------|------|---------|
| **Stub** | 提供预设返回值 | 被调用时返回固定数据，不验证交互 | 替代外部依赖，让测试聚焦被测逻辑 |
| **Mock** | 验证交互行为 | 记录是否被调用、调用参数、调用次数 | 验证被测代码是否正确调用了依赖 |
| **Spy** | 记录调用但不替换实现 | 委托给真实实现，同时记录调用信息 | 需要真实行为但也想验证调用 |
| **Fake** | 提供简化的真实实现 | 有实际逻辑但比真实实现简单 | In-memory 数据库、本地文件服务 |

```swift
// Stub — 固定返回值
class StubUserService: UserServiceProtocol {
    var stubbedUser: User? = User(name: "测试用户")
    func fetchUser(id: String) async -> User? { stubbedUser }
}

// Mock — 记录交互
class MockAnalyticsService: AnalyticsProtocol {
    var trackedEvents: [(name: String, params: [String: Any])] = []
    func track(event: String, params: [String: Any]) {
        trackedEvents.append((event, params))
    }
}

// Spy — 包装真实实现
class SpyLogger: LoggerProtocol {
    let realLogger = Logger()
    var loggedMessages: [String] = []
    func log(_ message: String) {
        loggedMessages.append(message)
        realLogger.log(message)
    }
}

// Fake — 简化实现
class FakeUserRepository: UserRepositoryProtocol {
    private var storage: [String: User] = [:]
    func save(_ user: User) { storage[user.id] = user }
    func fetch(id: String) -> User? { storage[id] }
    func delete(id: String) { storage.removeValue(forKey: id) }
}
```

---

## Protocol-based Mock 模式

### 步骤一：定义 Protocol

```swift
// 为被测代码的依赖定义协议（如果还没有）
protocol ProductServiceProtocol {
    func fetchProducts(page: Int) async throws -> [Product]
    func fetchProduct(id: String) async throws -> Product
    func searchProducts(query: String) async throws -> [Product]
}

// 生产代码实现
class ProductService: ProductServiceProtocol {
    private let apiClient: APIClient
    init(apiClient: APIClient) { self.apiClient = apiClient }

    func fetchProducts(page: Int) async throws -> [Product] {
        try await apiClient.request(.products(page: page))
    }
    // ...
}
```

### 步骤二：创建 Mock 实现

```swift
class MockProductService: ProductServiceProtocol {
    // Stub：预设返回值
    var stubbedProducts: [Product] = []
    var stubbedError: Error?

    // Mock：记录调用
    var fetchProductsCallCount = 0
    var lastFetchProductsPage: Int?

    func fetchProducts(page: Int) async throws -> [Product] {
        fetchProductsCallCount += 1
        lastFetchProductsPage = page
        if let error = stubbedError { throw error }
        return stubbedProducts
    }

    var stubbedProduct: Product?
    func fetchProduct(id: String) async throws -> Product {
        guard let product = stubbedProduct else {
            throw APIError.notFound
        }
        return product
    }

    var stubbedSearchResults: [Product] = []
    func searchProducts(query: String) async throws -> [Product] {
        stubbedSearchResults
    }
}
```

### 步骤三：注入依赖

```swift
// 被测 ViewModel 通过初始化器注入依赖
class ProductListViewModel: ObservableObject {
    private let service: ProductServiceProtocol

    @Published var products: [Product] = []
    @Published var isLoading = false
    @Published var error: String?

    init(service: ProductServiceProtocol) {
        self.service = service
    }

    func loadProducts() async {
        isLoading = true
        defer { isLoading = false }
        do {
            products = try await service.fetchProducts(page: 1)
        } catch {
            self.error = error.localizedDescription
        }
    }
}

// 测试中注入 Mock
@Test("加载商品列表成功")
func loadProductsSuccess() async {
    let mock = MockProductService()
    mock.stubbedProducts = [Product.mock, Product.mock]
    let viewModel = ProductListViewModel(service: mock)

    await viewModel.loadProducts()

    #expect(viewModel.products.count == 2)
    #expect(viewModel.isLoading == false)
    #expect(viewModel.error == nil)
    #expect(mock.fetchProductsCallCount == 1)
}
```

---

## 手动 Mock vs 框架生成

| 维度 | 手动 Mock | 框架（如 Mockolo、Sourcery） |
|------|----------|--------------------------|
| 学习成本 | 低，纯 Swift | 需要学习框架语法和配置 |
| 维护成本 | Protocol 变化时手动更新 | 自动生成，维护成本低 |
| 灵活性 | 完全可控 | 受框架能力限制 |
| 适用场景 | 小型项目、Protocol 少 | 大型项目、Protocol 多 |
| 推荐 | 优先使用 | Protocol > 10 个时考虑 |

---

## 常见 Mock 场景

### 网络层 Mock

**方式一：Protocol Mock（推荐）**

```swift
protocol APIClientProtocol {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T
}

class MockAPIClient: APIClientProtocol {
    var stubbedData: Any?
    var stubbedError: Error?

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        if let error = stubbedError { throw error }
        guard let data = stubbedData as? T else {
            fatalError("Mock 未配置正确的返回值类型")
        }
        return data
    }
}
```

**方式二：URLProtocol Mock（集成测试）**

```swift
class MockURLProtocol: URLProtocol {
    static var requestHandler: ((URLRequest) throws -> (HTTPURLResponse, Data))?

    override class func canInit(with request: URLRequest) -> Bool { true }
    override class func canonicalRequest(for request: URLRequest) -> URLRequest { request }

    override func startLoading() {
        guard let handler = Self.requestHandler else {
            fatalError("requestHandler 未设置")
        }
        do {
            let (response, data) = try handler(request)
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } catch {
            client?.urlProtocol(self, didFailWithError: error)
        }
    }

    override func stopLoading() {}
}

// 使用
let config = URLSessionConfiguration.ephemeral
config.protocolClasses = [MockURLProtocol.self]
let session = URLSession(configuration: config)

MockURLProtocol.requestHandler = { request in
    let response = HTTPURLResponse(url: request.url!, statusCode: 200, httpVersion: nil, headerFields: nil)!
    let data = try! JSONEncoder().encode(User.mock)
    return (response, data)
}
```

### 持久化层 Mock（In-Memory Store）

```swift
protocol UserStorageProtocol {
    func save(_ user: User) throws
    func fetch(id: String) -> User?
    func fetchAll() -> [User]
    func delete(id: String) throws
}

// 生产环境：Core Data / SwiftData
class UserCoreDataStorage: UserStorageProtocol { ... }

// 测试环境：内存存储
class InMemoryUserStorage: UserStorageProtocol {
    private var users: [String: User] = [:]

    func save(_ user: User) throws { users[user.id] = user }
    func fetch(id: String) -> User? { users[id] }
    func fetchAll() -> [User] { Array(users.values) }
    func delete(id: String) throws { users.removeValue(forKey: id) }
}
```

### 系统服务 Mock

```swift
// 位置服务
protocol LocationServiceProtocol {
    var currentLocation: CLLocation? { get }
    func requestPermission() async -> CLAuthorizationStatus
}

class MockLocationService: LocationServiceProtocol {
    var currentLocation: CLLocation? = CLLocation(latitude: 31.23, longitude: 121.47)
    var stubbedPermission: CLAuthorizationStatus = .authorizedWhenInUse

    func requestPermission() async -> CLAuthorizationStatus {
        stubbedPermission
    }
}

// 通知服务
protocol NotificationServiceProtocol {
    func requestAuthorization() async throws -> Bool
    func scheduleNotification(_ content: NotificationContent) throws
}

class MockNotificationService: NotificationServiceProtocol {
    var authorizationGranted = true
    var scheduledNotifications: [NotificationContent] = []

    func requestAuthorization() async throws -> Bool { authorizationGranted }
    func scheduleNotification(_ content: NotificationContent) throws {
        scheduledNotifications.append(content)
    }
}
```

### 日期/时间 Mock

```swift
// 不要直接使用 Date()，注入时间源
protocol DateProviderProtocol {
    var now: Date { get }
}

struct SystemDateProvider: DateProviderProtocol {
    var now: Date { Date() }
}

struct MockDateProvider: DateProviderProtocol {
    var now: Date

    init(year: Int, month: Int, day: Int, hour: Int = 0, minute: Int = 0) {
        var components = DateComponents()
        components.year = year
        components.month = month
        components.day = day
        components.hour = hour
        components.minute = minute
        self.now = Calendar.current.date(from: components)!
    }
}

// 使用
@Test("优惠券过期检查")
func couponExpired() {
    let dateProvider = MockDateProvider(year: 2025, month: 6, day: 1)
    let coupon = Coupon(expiresAt: Date(timeIntervalSince1970: 1_700_000_000))
    let checker = CouponChecker(dateProvider: dateProvider)
    #expect(checker.isExpired(coupon))
}
```

---

## @Observable 对象的测试模式

```swift
@Observable
class ProfileViewModel {
    var profile: Profile?
    var isLoading = false
    var errorMessage: String?

    private let service: ProfileServiceProtocol

    init(service: ProfileServiceProtocol) {
        self.service = service
    }

    func loadProfile(userId: String) async {
        isLoading = true
        defer { isLoading = false }
        do {
            profile = try await service.fetchProfile(userId: userId)
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

// 测试 @Observable — 直接访问属性即可
@Test("加载用户资料成功")
func loadProfileSuccess() async {
    let mock = MockProfileService()
    mock.stubbedProfile = Profile(name: "张三", bio: "iOS 开发者")
    let vm = ProfileViewModel(service: mock)

    await vm.loadProfile(userId: "123")

    #expect(vm.profile?.name == "张三")
    #expect(vm.isLoading == false)
    #expect(vm.errorMessage == nil)
}
```

---

## async/await 测试中的 Mock

```swift
// Mock 可以控制异步行为
class MockAuthService: AuthServiceProtocol {
    var stubbedResult: Result<AuthToken, Error> = .success(AuthToken.mock)
    var delay: Duration?

    func login(username: String, password: String) async throws -> AuthToken {
        if let delay { try await Task.sleep(for: delay) }
        return try stubbedResult.get()
    }
}

// 测试正常流程
@Test("登录成功返回 Token")
func loginSuccess() async throws {
    let mock = MockAuthService()
    mock.stubbedResult = .success(AuthToken(value: "abc123"))
    let vm = LoginViewModel(authService: mock)

    await vm.login(username: "user", password: "pass")

    #expect(vm.isLoggedIn == true)
    #expect(vm.token?.value == "abc123")
}

// 测试错误流程
@Test("登录失败显示错误信息")
func loginFailure() async {
    let mock = MockAuthService()
    mock.stubbedResult = .failure(AuthError.invalidCredentials)
    let vm = LoginViewModel(authService: mock)

    await vm.login(username: "user", password: "wrong")

    #expect(vm.isLoggedIn == false)
    #expect(vm.errorMessage == "用户名或密码错误")
}
```

---

## Mock 的反模式

### 1. 过度 Mock

```swift
// 错误：Mock 了一切，测试没有意义
@Test func badTest() {
    let mockVM = MockViewModel()       // Mock 了被测对象本身！
    mockVM.stubbedResult = "Hello"
    #expect(mockVM.greet() == "Hello") // 测试了 Mock，不是真实代码
}

// 正确：只 Mock 外部依赖
@Test func goodTest() {
    let mockService = MockGreetingService()
    mockService.stubbedGreeting = "你好"
    let vm = GreetingViewModel(service: mockService)  // 真实的 ViewModel
    #expect(vm.greet() == "你好")
}
```

### 2. Mock 实现细节

```swift
// 错误：验证内部方法调用顺序
@Test func badTest() {
    let mock = MockService()
    vm.doSomething()
    #expect(mock.callOrder == ["validate", "transform", "save"])  // 重构后必然失败
}

// 正确：验证最终结果
@Test func goodTest() {
    let mock = MockService()
    vm.doSomething()
    #expect(mock.savedData == expectedData)  // 只关心结果
}
```

### 3. Mock 泄漏生产逻辑

```swift
// 错误：在 Mock 中复制了生产代码的逻辑
class BadMockCalculator: CalculatorProtocol {
    func calculate(price: Int, tax: Double) -> Int {
        Int(Double(price) * (1 + tax))  // 复制了真实计算逻辑
    }
}

// 正确：Mock 返回固定值，真实逻辑由被测代码执行
class GoodMockTaxProvider: TaxProviderProtocol {
    var stubbedRate: Double = 0.1
    func taxRate(for region: String) -> Double { stubbedRate }
}
```

---

## 速查 Checklist

> 完整的测试质量标准见 `test-quality-checklist.md`，此处仅列 Mock/Stub 专项检查。

- [ ] 只 Mock 外部依赖，不 Mock 被测对象本身
- [ ] 优先使用 Protocol-based Mock，保持类型安全
- [ ] Mock 返回固定值，不复制生产代码逻辑
- [ ] Mock 的属性命名清晰：`stubbedXxx`（返回值）、`xxxCallCount`（调用次数）
- [ ] 日期、时间等非确定性依赖必须通过注入 Mock
- [ ] 网络层优先用 Protocol Mock，集成测试用 URLProtocol Mock
- [ ] 持久化层用 In-Memory Fake 替代真实数据库
- [ ] 验证行为结果，不验证内部方法调用顺序
- [ ] Mock 复杂度与被测代码匹配——简单依赖用简单 Mock
