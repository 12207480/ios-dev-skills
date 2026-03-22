# 集成测试指南

## 集成测试 vs 单元测试的边界

| 维度 | 单元测试 | 集成测试 |
|------|---------|---------|
| **测试范围** | 单个类/方法，依赖全部 Mock | 多个真实组件协作 |
| **依赖处理** | Mock 所有外部依赖 | 仅 Mock 系统边界（网络/文件系统） |
| **运行速度** | < 0.1 秒/个 | < 1 秒/个 |
| **失败定位** | 精确到某个方法 | 需排查组件间交互 |
| **适用场景** | 业务逻辑、算法、数据转换 | 模块间协作、数据流、API 集成 |

**何时写集成测试：**
- 单元测试无法覆盖的组件间交互逻辑
- 数据在多层之间传递时的转换和丢失风险
- 多个模块协作完成一个业务流程
- 验证真实依赖（数据库、缓存）的集成行为

**何时不需要集成测试：**
- 纯函数、数据模型、工具方法 → 单元测试足够
- UI 交互和视觉表现 → UI 测试 / 快照测试
- 简单的 CRUD 操作且已有单元测试覆盖

---

## URLProtocol 集成测试

使用 `URLProtocol` 拦截真实的 `URLSession` 请求，测试网络层从请求到解析的完整链路。

```swift
import XCTest
@testable import MyApp

final class APIClientIntegrationTests: XCTestCase {
    private var session: URLSession!
    private var apiClient: APIClient!

    override func setUp() {
        super.setUp()
        let config = URLSessionConfiguration.ephemeral
        config.protocolClasses = [MockURLProtocol.self]
        session = URLSession(configuration: config)
        apiClient = APIClient(session: session)
    }

    override func tearDown() {
        MockURLProtocol.requestHandler = nil
        session = nil
        apiClient = nil
        super.tearDown()
    }

    func test_fetchUser_parsesResponseCorrectly() async throws {
        // 模拟服务端返回
        let responseJSON = """
        {"id": "123", "name": "张三", "email": "zhang@example.com"}
        """.data(using: .utf8)!

        MockURLProtocol.requestHandler = { request in
            XCTAssertEqual(request.url?.path, "/api/users/123")
            XCTAssertEqual(request.httpMethod, "GET")
            let response = HTTPURLResponse(
                url: request.url!, statusCode: 200,
                httpVersion: nil, headerFields: ["Content-Type": "application/json"]
            )!
            return (response, responseJSON)
        }

        // 测试真实的 APIClient → URLSession → 解析链路
        let user = try await apiClient.fetchUser(id: "123")
        XCTAssertEqual(user.name, "张三")
        XCTAssertEqual(user.email, "zhang@example.com")
    }

    func test_fetchUser_handles404() async {
        MockURLProtocol.requestHandler = { request in
            let response = HTTPURLResponse(
                url: request.url!, statusCode: 404,
                httpVersion: nil, headerFields: nil
            )!
            return (response, Data())
        }

        do {
            _ = try await apiClient.fetchUser(id: "nonexistent")
            XCTFail("应抛出 notFound 错误")
        } catch {
            XCTAssertEqual(error as? APIError, .notFound)
        }
    }
}
```

> `MockURLProtocol` 的完整实现见 `mock-stub-patterns.md` → URLProtocol Mock

---

## 多层协作测试（ViewModel + Service + Repository）

测试多个真实组件的协作行为，仅在系统边界使用 Mock。

```swift
@Suite("订单流程集成测试")
struct OrderFlowIntegrationTests {
    // 仅 Mock 最外层依赖：网络和持久化
    let mockAPI = MockAPIClient()
    let inMemoryStorage = InMemoryOrderStorage()

    @Test("下单流程：创建订单 → 保存本地 → 同步服务端")
    func createOrderFlow() async throws {
        // 真实的 Service 和 Repository，仅 Mock 网络和存储
        let repository = OrderRepository(storage: inMemoryStorage)
        let service = OrderService(api: mockAPI, repository: repository)
        let viewModel = OrderViewModel(service: service)

        mockAPI.stubbedResponse = OrderResponse(orderId: "ORD-001", status: "created")

        // 执行完整流程
        await viewModel.createOrder(
            items: [OrderItem(productId: "P1", quantity: 2)]
        )

        // 验证最终结果
        #expect(viewModel.currentOrder?.id == "ORD-001")
        #expect(viewModel.errorMessage == nil)

        // 验证本地存储也被更新
        let savedOrder = inMemoryStorage.fetch(id: "ORD-001")
        #expect(savedOrder != nil)
        #expect(savedOrder?.status == "created")
    }

    @Test("下单失败：网络错误时显示提示且不保存本地")
    func createOrderNetworkError() async {
        let repository = OrderRepository(storage: inMemoryStorage)
        let service = OrderService(api: mockAPI, repository: repository)
        let viewModel = OrderViewModel(service: service)

        mockAPI.stubbedError = URLError(.notConnectedToInternet)

        await viewModel.createOrder(
            items: [OrderItem(productId: "P1", quantity: 1)]
        )

        #expect(viewModel.currentOrder == nil)
        #expect(viewModel.errorMessage == "网络不可用，请检查网络设置")
        #expect(inMemoryStorage.fetchAll().isEmpty)
    }
}
```

---

## 端到端数据流测试

验证数据从输入到展示的完整流转。

```swift
@Suite("搜索数据流集成测试")
struct SearchDataFlowTests {
    @Test("搜索关键词 → API请求 → 解析响应 → 展示结果")
    func searchEndToEnd() async {
        // 准备模拟 API 响应
        let mockAPI = MockAPIClient()
        mockAPI.stubbedData = ProductListResponse(
            products: [
                ProductDTO(id: "1", name: "iPhone 16", price: 7999),
                ProductDTO(id: "2", name: "iPhone 16 Pro", price: 9999),
            ],
            total: 2
        )

        // 使用真实的 Service 和 ViewModel
        let service = ProductService(api: mockAPI)
        let viewModel = SearchViewModel(service: service)

        viewModel.query = "iPhone"
        await viewModel.search()

        // 验证完整数据流的结果
        #expect(viewModel.results.count == 2)
        #expect(viewModel.results[0].name == "iPhone 16")
        #expect(viewModel.results[0].displayPrice == "¥7,999")  // 验证格式化
        #expect(viewModel.totalCount == 2)
    }
}
```

---

## Core Data / SwiftData 集成测试

测试数据持久化层与业务逻辑的集成。

### Core Data 集成测试

```swift
final class UserCoreDataIntegrationTests: XCTestCase {
    private var container: NSPersistentContainer!
    private var repository: UserRepository!

    override func setUp() {
        super.setUp()
        // 使用内存存储，每个测试独立
        container = .inMemory(name: "Model")
        repository = UserRepository(context: container.viewContext)
    }

    override func tearDown() {
        container = nil
        repository = nil
        super.tearDown()
    }

    func test_saveAndFetchWithPredicate() throws {
        // 保存多个用户
        try repository.save(User(id: "1", name: "张三", age: 25))
        try repository.save(User(id: "2", name: "李四", age: 30))
        try repository.save(User(id: "3", name: "王五", age: 20))

        // 测试条件查询
        let adults = try repository.fetch(where: { $0.age >= 25 })
        XCTAssertEqual(adults.count, 2)

        // 测试排序
        let sorted = try repository.fetchAll(sortedBy: \.age, ascending: true)
        XCTAssertEqual(sorted.first?.name, "王五")
        XCTAssertEqual(sorted.last?.name, "李四")
    }

    func test_cascadeDelete() throws {
        // 测试级联删除：删除用户时关联的订单也被删除
        let user = User(id: "1", name: "张三", age: 25)
        try repository.save(user)
        try repository.addOrder(Order(id: "O1", amount: 100), to: user)

        try repository.deleteUser(id: "1")

        XCTAssertNil(repository.fetchUser(id: "1"))
        XCTAssertTrue(repository.fetchOrders(forUserId: "1").isEmpty)
    }
}
```

### SwiftData 集成测试

```swift
@Suite("SwiftData 集成测试")
struct SwiftDataIntegrationTests {
    let container: ModelContainer

    init() throws {
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        container = try ModelContainer(
            for: TaskModel.self, TagModel.self,
            configurations: config
        )
    }

    @Test("任务与标签多对多关系")
    @MainActor
    func taskTagRelationship() throws {
        let context = container.mainContext

        let tag = TagModel(name: "重要")
        let task = TaskModel(title: "完成报告", tags: [tag])
        context.insert(task)
        try context.save()

        // 通过标签查询任务
        let descriptor = FetchDescriptor<TaskModel>(
            predicate: #Predicate { task in
                task.tags.contains { $0.name == "重要" }
            }
        )
        let results = try context.fetch(descriptor)
        #expect(results.count == 1)
        #expect(results.first?.title == "完成报告")
    }
}
```

---

## 集成测试环境配置

### 测试环境隔离

```swift
// 集成测试专用配置
enum TestEnvironment {
    // 创建隔离的测试依赖容器
    static func makeDependencies() -> AppDependencies {
        AppDependencies(
            api: makeTestAPIClient(),
            storage: InMemoryStorage(),
            cache: InMemoryCache(),
            dateProvider: FixedDateProvider(date: TestDates.reference)
        )
    }

    // 使用 URLProtocol 拦截的 URLSession
    static func makeTestAPIClient() -> APIClient {
        let config = URLSessionConfiguration.ephemeral
        config.protocolClasses = [MockURLProtocol.self]
        let session = URLSession(configuration: config)
        return APIClient(
            session: session,
            baseURL: URL(string: "https://api.test.example.com")!
        )
    }
}

// 在集成测试中使用
@Suite("用户模块集成测试")
struct UserModuleIntegrationTests {
    let deps = TestEnvironment.makeDependencies()

    @Test("注册 → 登录 → 获取个人信息")
    func registrationToProfileFlow() async throws {
        let authService = AuthService(deps: deps)
        let profileService = ProfileService(deps: deps)

        // 配置模拟响应...
        // 执行完整流程...
    }
}
```

### CI 环境注意事项

```yaml
# 集成测试通常比单元测试慢，建议在 CI 中分开运行
- name: Run Unit Tests
  run: |
    xcodebuild test \
      -scheme MyApp \
      -only-testing:MyAppTests/UnitTests \
      -destination 'platform=iOS Simulator,name=iPhone 15'

- name: Run Integration Tests
  run: |
    xcodebuild test \
      -scheme MyApp \
      -only-testing:MyAppTests/IntegrationTests \
      -destination 'platform=iOS Simulator,name=iPhone 15'
```

---

## 速查 Checklist

> 完整的测试质量标准见 `test-quality-checklist.md`，此处仅列集成测试专项检查。

- [ ] 仅在系统边界 Mock（网络/文件系统），组件间使用真实实现
- [ ] 每个测试使用独立的 in-memory 存储，无测试间状态泄漏
- [ ] Core Data / SwiftData 测试使用 `isStoredInMemoryOnly` 配置
- [ ] URLProtocol Mock 在 `tearDown` 中清理 `requestHandler`
- [ ] 验证完整数据流的最终结果，而非中间步骤
- [ ] 集成测试命名清晰描述参与的组件和场景
- [ ] 集成测试与单元测试分目录存放，CI 可独立运行
- [ ] 网络错误、超时等异常场景有对应的集成测试
- [ ] 数据库关系（级联删除、关联查询）有集成测试覆盖
- [ ] 测试环境配置集中管理，避免每个测试重复设置
