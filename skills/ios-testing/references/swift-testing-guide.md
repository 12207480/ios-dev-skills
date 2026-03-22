# Swift Testing 与 XCTest 指南

## Swift Testing vs XCTest 决策表

| 维度 | Swift Testing | XCTest |
|------|--------------|--------|
| 最低要求 | Xcode 16+ / Swift 6+ | 任何 Xcode 版本 |
| 测试声明 | `@Test` 属性 | `test` 前缀方法 |
| 测试组织 | `@Suite` struct/actor | `XCTestCase` class |
| 断言 | `#expect` / `#require` 宏 | `XCTAssert` 系列函数 |
| 参数化测试 | 原生支持 `@Test(arguments:)` | 需手动循环或多个方法 |
| 并发安全 | 设计时即考虑 Swift Concurrency | 需额外处理 |
| 性能测试 | 不支持（使用 XCTest） | `measure {}` |
| UI 测试 | 不支持（使用 XCUITest） | `XCUIApplication` |
| 混合使用 | 可与 XCTest 共存于同一 Target | — |

**选择建议：**
- 新项目 / 新测试文件 → Swift Testing
- 现有 XCTest 项目 → 新文件用 Swift Testing，旧文件按需迁移
- 性能测试 / UI 测试 → 仍使用 XCTest
- 项目要求 Xcode 15 以下 → 只能用 XCTest

---

## Swift Testing 核心概念

### @Test 属性与标签

```swift
import Testing

// 基础用法
@Test("用户登录成功后跳转首页")
func loginSuccess() {
    let auth = AuthManager(service: MockAuthService(shouldSucceed: true))
    let result = auth.login(username: "test", password: "pass123")
    #expect(result == .success)
}

// 带标签分组
@Test("空用户名时显示错误提示", .tags(.validation))
func emptyUsername() {
    let validator = LoginValidator()
    let result = validator.validate(username: "", password: "pass123")
    #expect(result == .invalid(reason: .emptyUsername))
}

// 自定义标签
extension Tag {
    @Tag static var validation: Self
    @Tag static var networking: Self
    @Tag static var persistence: Self
}
```

### @Suite 测试套件组织

```swift
@Suite("用户认证模块")
struct AuthTests {
    // 共享的测试初始化（替代 setUp）
    let authManager: AuthManager

    init() {
        authManager = AuthManager(service: MockAuthService())
    }

    @Suite("登录流程")
    struct LoginTests {
        @Test("有效凭证登录成功")
        func validCredentials() { ... }

        @Test("无效密码登录失败")
        func invalidPassword() { ... }
    }

    @Suite("注册流程")
    struct RegistrationTests {
        @Test("有效信息注册成功")
        func validRegistration() { ... }
    }
}
```

### #expect 和 #require 断言

```swift
@Test("断言示例")
func assertions() throws {
    let user = User(name: "张三", age: 25)

    // 基础断言 — 失败后继续执行
    #expect(user.name == "张三")
    #expect(user.age >= 18)
    #expect(user.isActive)
    #expect(!user.isBlocked)

    // 可选值断言
    #expect(user.email != nil)
    #expect(user.avatar == nil)

    // 必须断言 — 失败后立即停止当前测试
    let profile = try #require(user.profile)
    #expect(profile.bio.count <= 200)

    // 异常断言
    #expect(throws: ValidationError.tooShort) {
        try validator.validate(password: "ab")
    }

    // 验证抛出任意错误
    #expect(throws: (any Error).self) {
        try riskyOperation()
    }

    // 验证不抛出错误
    #expect(throws: Never.self) {
        try safeOperation()
    }
}
```

### 参数化测试（@Test(arguments:)）

```swift
// 单参数
@Test("邮箱格式验证", arguments: [
    "user@example.com",
    "test.user@domain.co",
    "a@b.cn",
])
func validEmails(email: String) {
    #expect(EmailValidator.isValid(email))
}

@Test("无效邮箱拒绝", arguments: [
    "",
    "not-an-email",
    "@domain.com",
    "user@",
])
func invalidEmails(email: String) {
    #expect(!EmailValidator.isValid(email))
}

// 多参数组合 — 期望值必须硬编码，不能复制生产代码逻辑
@Test("价格计算", arguments: [
    (1000, 0.1, 900),    // 原价1000，9折，期望900
    (2000, 0.2, 1600),   // 原价2000，8折，期望1600
    (500,  0.0, 500),    // 无折扣，期望原价
    (0,    0.5, 0),      // 原价为0，期望0
])
func priceCalculation(price: Int, discount: Double, expected: Int) {
    let result = calculateFinalPrice(price: price, discount: discount)
    #expect(result == expected)
}

// 使用枚举 CaseIterable
enum Theme: CaseIterable {
    case light, dark, auto
}

@Test("所有主题都能正确应用", arguments: Theme.allCases)
func applyTheme(theme: Theme) {
    let manager = ThemeManager()
    manager.apply(theme)
    #expect(manager.currentTheme == theme)
}
```

### 异步测试

```swift
@Test("加载商品列表")
func loadProducts() async throws {
    let viewModel = ProductListViewModel(service: MockProductService())
    await viewModel.loadProducts()
    #expect(viewModel.products.count == 3)
    #expect(viewModel.isLoading == false)
}

@Test("并发添加到购物车不丢数据")
func concurrentAdd() async {
    let cart = Cart()
    await withTaskGroup(of: Void.self) { group in
        for i in 0..<100 {
            group.addTask {
                await cart.add(Item(id: i))
            }
        }
    }
    #expect(await cart.items.count == 100)
}
```

### 测试 Trait

```swift
// 条件启用
@Test("仅在 iOS 18+ 运行", .enabled(if: ProcessInfo().operatingSystemVersion.majorVersion >= 18))
func newAPIFeature() { ... }

// 已知 Bug 标记
@Test("修复前暂时跳过", .bug("https://issues.example.com/123", "依赖后端修复"))
func knownIssue() { ... }

// 禁用测试（比注释更好）
@Test("暂时禁用", .disabled("等待 API v2 就绪"))
func disabledTest() { ... }

// 时间限制
@Test("网络请求应在 5 秒内完成", .timeLimit(.seconds(5)))
func networkTimeout() async throws { ... }

// 串行执行（默认并行）
@Suite("数据库操作", .serialized)
struct DatabaseTests { ... }
```

---

## XCTest 用法速查（兼容旧项目）

### XCTestCase 子类

```swift
import XCTest
@testable import MyApp

final class UserServiceTests: XCTestCase {
    private var sut: UserService!  // System Under Test
    private var mockAPI: MockAPIClient!

    override func setUp() {
        super.setUp()
        mockAPI = MockAPIClient()
        sut = UserService(api: mockAPI)
    }

    override func tearDown() {
        sut = nil
        mockAPI = nil
        super.tearDown()
    }

    func test_fetchUser_success_returnsUser() {
        // Arrange
        mockAPI.stubbedResult = .success(User.mock)
        // Act
        let result = sut.fetchUser(id: "123")
        // Assert
        XCTAssertNotNil(result)
        XCTAssertEqual(result?.name, "测试用户")
    }
}
```

### XCTAssert 系列断言

```swift
XCTAssertEqual(a, b)              // a == b
XCTAssertNotEqual(a, b)           // a != b
XCTAssertTrue(condition)           // condition == true
XCTAssertFalse(condition)          // condition == false
XCTAssertNil(value)                // value == nil
XCTAssertNotNil(value)             // value != nil
XCTAssertGreaterThan(a, b)        // a > b
XCTAssertLessThanOrEqual(a, b)    // a <= b
XCTAssertThrowsError(try expr())  // 表达式抛出错误
XCTAssertNoThrow(try expr())      // 表达式不抛出错误
XCTFail("不应该执行到这里")         // 直接标记失败
```

### 异步等待 XCTestExpectation

```swift
func test_downloadImage_completionCalled() {
    // Arrange
    let expectation = expectation(description: "下载完成回调")
    var downloadedImage: UIImage?

    // Act
    imageLoader.download(url: testURL) { image in
        downloadedImage = image
        expectation.fulfill()
    }

    // Assert
    waitForExpectations(timeout: 5)
    XCTAssertNotNil(downloadedImage)
}

// async/await 版本（推荐）
func test_downloadImage_async() async throws {
    let image = try await imageLoader.download(url: testURL)
    XCTAssertNotNil(image)
}
```

### 性能测试（measure）

性能测试仅支持 XCTest，Swift Testing 不提供此功能。

```swift
final class PerformanceTests: XCTestCase {
    // 基础用法：measure 块会执行 5 次，取平均值
    func test_sortPerformance() {
        var data = (0..<10000).map { _ in Int.random(in: 0..<100000) }
        measure {
            data.sort()
        }
    }

    // 指定度量指标
    func test_jsonParsingPerformance() throws {
        let largeJSON = try loadFixture("large_response.json")
        let metrics: [XCTMetric] = [
            XCTClockMetric(),           // 耗时（默认）
            XCTMemoryMetric(),          // 内存分配
            XCTCPUMetric(),             // CPU 使用率
        ]
        let options = XCTMeasureOptions()
        options.iterationCount = 10     // 自定义迭代次数（默认5）

        measure(metrics: metrics, options: options) {
            _ = try? JSONDecoder().decode([Product].self, from: largeJSON)
        }
    }

    // 使用基线检测性能回归
    // 首次运行后，在 Xcode 中点击测试结果旁的菱形图标设置基线
    func test_imageResizePerformance() {
        let image = UIImage(named: "testImage")!
        measure {
            _ = image.resized(to: CGSize(width: 100, height: 100))
        }
    }
}
```

**基线设置与管理：**
1. 首次运行性能测试后，在 Test Navigator 中点击测试方法旁的结果
2. 点击 "Set Baseline" 将当前结果设为基准
3. 后续运行中，Xcode 会对比结果与基线的偏差
4. 基线存储在 `.xcbaseline` 文件中，建议纳入版本控制

**标准偏差与阈值：**
- Xcode 默认允许 10% 的标准偏差
- 超过基线 10% 时测试标记为失败
- 可在 Xcode 测试结果中手动调整允许的偏差范围

**稳定性注意事项：**
- 避免在 CI 和本地共用同一基线（硬件差异大）
- 性能测试受系统负载影响，CI 上偶尔波动是正常的
- 测试前避免后台运行其他重任务
- 使用 Release 配置运行性能测试以获得真实性能数据
- 首次 `measure` 迭代通常较慢（冷启动），框架会自动处理

---

## 测试命名规范

### Swift Testing

```swift
// 格式：@Test("[场景描述] [预期结果]")
@Test("空购物车总价为零")
@Test("添加商品后数量增加")
@Test("超出库存时显示提示")
@Test("网络失败时重试三次后显示错误")
```

### XCTest

```swift
// 格式：test_[被测方法]_[条件]_[预期结果]
func test_calculateTotal_emptyCart_returnsZero()
func test_addItem_validItem_incrementsCount()
func test_checkout_insufficientStock_showsAlert()
func test_fetchData_networkFailure_retriesThreeTimes()
```

---

## 测试组织结构

```
MyAppTests/
├── UnitTests/
│   ├── ViewModels/
│   │   ├── HomeViewModelTests.swift
│   │   └── ProfileViewModelTests.swift
│   ├── Services/
│   │   ├── AuthServiceTests.swift
│   │   └── ProductServiceTests.swift
│   ├── Models/
│   │   ├── UserTests.swift
│   │   └── OrderTests.swift
│   └── Utilities/
│       └── DateFormatterTests.swift
├── IntegrationTests/
│   └── APIIntegrationTests.swift
├── Helpers/
│   ├── Mocks/
│   │   ├── MockAuthService.swift
│   │   └── MockAPIClient.swift
│   ├── Factories/
│   │   ├── UserFactory.swift
│   │   └── OrderFactory.swift
│   └── Extensions/
│       └── XCTestCase+Helpers.swift
└── Resources/
    └── Fixtures/
        ├── user_response.json
        └── products_response.json

MyAppUITests/
├── Screens/
│   ├── LoginScreen.swift
│   └── HomeScreen.swift
├── Tests/
│   ├── LoginFlowTests.swift
│   └── CheckoutFlowTests.swift
└── Helpers/
    └── XCUIApplication+Launch.swift
```

**关键原则：**
- 测试文件与被测文件一一对应（`UserService.swift` → `UserServiceTests.swift`）
- Mock 和 Factory 放在 `Helpers/` 下共享
- UI 测试使用独立 Target（`MyAppUITests`）
- JSON fixture 放在 `Resources/Fixtures/` 下

---

## 速查 Checklist

> 完整的测试质量标准见 `test-quality-checklist.md`，此处仅列框架选择与用法专项检查。

- [ ] 新文件使用 Swift Testing，除非需要性能测试或 UI 测试
- [ ] Swift Testing 和 XCTest 不在同一个文件中混用
- [ ] 参数化测试用于替代复制粘贴的相似测试
- [ ] `#require` 用于前置条件（失败则跳过后续），`#expect` 用于断言
- [ ] 异步测试使用 `async/await`，不使用 `sleep` 或 `DispatchQueue.asyncAfter`
- [ ] 测试命名清晰描述场景和预期结果
- [ ] 测试文件与源文件目录结构对应
- [ ] Mock/Factory 放在共享 Helpers 目录
