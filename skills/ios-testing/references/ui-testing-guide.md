# UI 测试指南

## XCUITest 基础

### 启动与配置

```swift
import XCTest

final class LoginFlowTests: XCTestCase {
    let app = XCUIApplication()

    override func setUp() {
        super.setUp()
        continueAfterFailure = false

        // 通过启动参数配置测试环境
        app.launchArguments = ["--uitesting"]
        app.launchEnvironment = [
            "API_BASE_URL": "http://localhost:8080",
            "DISABLE_ANIMATIONS": "1",
        ]
        app.launch()
    }

    override func tearDown() {
        app.terminate()
        super.tearDown()
    }
}
```

### 元素查找策略

优先级从高到低：

```swift
// 1. accessibilityIdentifier（推荐：稳定、不受文本变化影响）
let loginButton = app.buttons["loginButton"]
let emailField = app.textFields["emailTextField"]

// 在 SwiftUI 中设置
Button("登录") { ... }
    .accessibilityIdentifier("loginButton")

// 在 UIKit 中设置
button.accessibilityIdentifier = "loginButton"

// 2. accessibilityLabel（次选：用于验证显示内容）
let welcomeLabel = app.staticTexts["欢迎回来"]

// 3. 类型 + 索引（最后手段：不稳定，UI 变化易碎）
let firstCell = app.cells.element(boundBy: 0)  // 避免使用

// 查找嵌套元素
let cell = app.cells["orderCell_001"]
let statusLabel = cell.staticTexts["statusLabel"]

// 使用 predicate 查找
let matchingCells = app.cells.matching(
    NSPredicate(format: "identifier CONTAINS 'order'")
)
```

### 常用交互

```swift
// 点击
app.buttons["submitButton"].tap()
app.cells.element(boundBy: 0).tap()

// 输入文本
let field = app.textFields["emailTextField"]
field.tap()
field.clearAndTypeText("user@example.com")  // 自定义扩展，见下文

// 清除文本（XCUIElement 扩展）
extension XCUIElement {
    func clearAndTypeText(_ text: String) {
        guard let currentValue = value as? String, !currentValue.isEmpty else {
            tap()
            typeText(text)
            return
        }
        tap()
        let deleteString = String(repeating: XCUIKeyboardKey.delete.rawValue, count: currentValue.count)
        typeText(deleteString)
        typeText(text)
    }
}

// 滑动
app.swipeUp()
app.tables.firstMatch.swipeDown()

// 长按
app.cells["itemCell"].press(forDuration: 1.5)

// 拖拽
let cell = app.cells["draggableCell"]
cell.press(forDuration: 0.5, thenDragTo: app.cells.element(boundBy: 0))
```

### 等待策略

```swift
// 等待元素出现（推荐）
let successMessage = app.staticTexts["operationSuccess"]
let exists = successMessage.waitForExistence(timeout: 10)
XCTAssertTrue(exists, "成功提示未在预期时间内出现")

// 等待元素满足条件（复杂场景）
let loadingIndicator = app.activityIndicators["loadingSpinner"]
let disappeared = NSPredicate(format: "exists == false")
let expectation = XCTNSPredicateExpectation(predicate: disappeared, object: loadingIndicator)
let result = XCTWaiter().wait(for: [expectation], timeout: 15)
XCTAssertEqual(result, .completed, "加载指示器未在预期时间内消失")

// 等待元素可交互
let button = app.buttons["nextButton"]
let enabled = NSPredicate(format: "isEnabled == true")
let enabledExpectation = XCTNSPredicateExpectation(predicate: enabled, object: button)
XCTWaiter().wait(for: [enabledExpectation], timeout: 5)
button.tap()
```

---

## Page Object Pattern

### Screen 对象封装

```swift
// 基础 Screen 协议
protocol Screen {
    var app: XCUIApplication { get }
    func verify() -> Self
}

// 登录页
struct LoginScreen: Screen {
    let app: XCUIApplication

    // 元素定义
    private var emailField: XCUIElement { app.textFields["emailTextField"] }
    private var passwordField: XCUIElement { app.secureTextFields["passwordTextField"] }
    private var loginButton: XCUIElement { app.buttons["loginButton"] }
    private var errorLabel: XCUIElement { app.staticTexts["errorLabel"] }

    // 验证页面已加载
    @discardableResult
    func verify() -> Self {
        XCTAssertTrue(emailField.waitForExistence(timeout: 5), "登录页未加载")
        return self
    }

    // 操作方法
    @discardableResult
    func typeEmail(_ email: String) -> Self {
        emailField.tap()
        emailField.typeText(email)
        return self
    }

    @discardableResult
    func typePassword(_ password: String) -> Self {
        passwordField.tap()
        passwordField.typeText(password)
        return self
    }

    // 导航方法 — 返回目标页面
    func tapLogin() -> HomeScreen {
        loginButton.tap()
        return HomeScreen(app: app).verify()
    }

    func tapLoginExpectingError() -> Self {
        loginButton.tap()
        XCTAssertTrue(errorLabel.waitForExistence(timeout: 5))
        return self
    }

    // 断言方法
    func assertErrorMessage(_ message: String) -> Self {
        XCTAssertEqual(errorLabel.label, message)
        return self
    }
}

// 首页
struct HomeScreen: Screen {
    let app: XCUIApplication

    private var welcomeLabel: XCUIElement { app.staticTexts["welcomeLabel"] }
    private var profileTab: XCUIElement { app.tabBars.buttons["个人中心"] }

    @discardableResult
    func verify() -> Self {
        XCTAssertTrue(welcomeLabel.waitForExistence(timeout: 5), "首页未加载")
        return self
    }

    func tapProfileTab() -> ProfileScreen {
        profileTab.tap()
        return ProfileScreen(app: app).verify()
    }
}
```

### 使用 Page Object 编写测试

```swift
final class LoginFlowTests: XCTestCase {
    let app = XCUIApplication()

    override func setUp() {
        super.setUp()
        continueAfterFailure = false
        app.launchArguments = ["--uitesting"]
        app.launch()
    }

    func test_登录成功跳转首页() {
        LoginScreen(app: app)
            .verify()
            .typeEmail("user@example.com")
            .typePassword("password123")
            .tapLogin()  // 返回 HomeScreen
            .verify()    // 验证首页已加载
    }

    func test_密码错误显示提示() {
        LoginScreen(app: app)
            .verify()
            .typeEmail("user@example.com")
            .typePassword("wrong")
            .tapLoginExpectingError()
            .assertErrorMessage("用户名或密码错误")
    }
}
```

---

## UI 测试最佳实践

### 测试用户行为而非实现

```swift
// 错误：测试 UI 层级结构
func test_bad() {
    let cell = app.tables.cells.element(boundBy: 0)
    let label = cell.staticTexts.element(boundBy: 1)  // 脆弱：依赖索引
    XCTAssertEqual(label.label, "商品名称")
}

// 正确：测试用户可见的行为
func test_good() {
    let productName = app.cells["productCell_0"].staticTexts["productName"]
    XCTAssertTrue(productName.exists)
}
```

### 保持测试独立性

```swift
// 每个测试从 app launch 开始，不依赖其他测试的状态
func test_添加商品到购物车() {
    // 通过 Launch Environment 预设测试数据
    app.launchEnvironment["MOCK_SCENARIO"] = "product_list"
    app.launch()

    HomeScreen(app: app)
        .tapFirstProduct()
        .tapAddToCart()
        .verify()
}
```

### 处理系统弹窗

```swift
override func setUp() {
    super.setUp()

    // 方式一：自动处理系统权限弹窗
    addUIInterruptionMonitor(withDescription: "系统权限弹窗") { alert in
        let allowButton = alert.buttons["允许"]
        if allowButton.exists {
            allowButton.tap()
            return true
        }
        return false
    }

    app.launch()
}

// 方式二：通过 Launch Arguments 绕过权限请求
app.launchArguments += [
    "-com.apple.CoreData.SQLDebug", "0",
]

// 方式三：重置权限状态（Xcode 15+）
func test_locationPermission() {
    app.resetAuthorizationStatus(for: .location)
    app.launch()
    // 测试首次请求权限的流程
}
```

### 测试环境准备

```swift
// AppDelegate 或 App 入口检测测试模式
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            if ProcessInfo.processInfo.arguments.contains("--uitesting") {
                // UI 测试模式：使用 Mock 数据
                ContentView()
                    .environment(\.apiClient, MockAPIClient())
            } else {
                ContentView()
            }
        }
    }
}

// 通过 Launch Environment 控制 Mock 场景
let scenario = ProcessInfo.processInfo.environment["MOCK_SCENARIO"]
switch scenario {
case "empty_list":
    return []
case "error":
    throw APIError.serverError
default:
    return MockData.defaultProducts
}
```

---

## UI 测试的成本收益分析

### 值得做 UI 测试的场景

| 场景 | 原因 | 测试重点 |
|------|------|---------|
| 核心用户路径 | 注册/登录/下单/支付，出错影响大 | 完整流程走通 |
| 复杂表单流程 | 多步骤、条件联动、验证规则多 | 输入验证、步骤跳转 |
| 关键导航流程 | 深层级导航、Tab 切换、返回逻辑 | 导航正确性 |
| 权限相关流程 | 首次请求权限、权限变更后行为 | 各权限状态下的表现 |

### 不值得做 UI 测试的场景

| 场景 | 原因 | 替代方案 |
|------|------|---------|
| 纯展示页面 | UI 经常变化，维护成本高 | 快照测试 |
| 简单列表页 | 逻辑简单，单元测试足够 | ViewModel 单元测试 |
| 动画效果 | 难以准确验证 | 人工验收 |
| 第三方 SDK 页面 | 无法控制内部实现 | 集成测试 / 人工验收 |

---

## 稳定性治理（消除 Flaky Test）

### 常见 Flaky 原因与解决

| 原因 | 表现 | 解决方案 |
|------|------|---------|
| 等待时间不足 | 偶尔超时失败 | 使用 `waitForExistence` 并设置合理 timeout |
| 动画未完成 | 点击操作偶尔无效 | 禁用动画 `UIView.setAnimationsEnabled(false)` |
| 网络不稳定 | 依赖真实 API 偶尔超时 | 使用 Mock Server 或 Launch Environment |
| 系统弹窗干扰 | 权限弹窗挡住 UI 元素 | 使用 `addUIInterruptionMonitor` |
| 键盘遮挡 | 输入框被键盘遮挡无法交互 | 先滚动到可见区域 |
| 测试间状态泄漏 | 上一个测试残留数据影响下一个 | 确保 tearDown 清理状态 |

### 禁用动画

```swift
// 方式一：Launch Arguments
app.launchArguments += ["DISABLE_ANIMATIONS"]

// 在 App 中检测
if ProcessInfo.processInfo.arguments.contains("DISABLE_ANIMATIONS") {
    UIView.setAnimationsEnabled(false)
}

// 方式二：直接在 setUp 中禁用（仅影响测试进程）
override func setUp() {
    UIView.setAnimationsEnabled(false)
}
```

### Flaky Test 处理流程

1. **标记** — 用 `XCTExpectFailure` 临时标记已知不稳定测试
2. **排查** — 运行 10 次以上，统计失败率和失败模式
3. **修复** — 根据原因表对症修复
4. **验证** — 修复后连续运行 20 次无失败
5. **移除标记** — 确认稳定后移除 `XCTExpectFailure`

```swift
// 临时标记不稳定测试（不要长期保留）
func test_possiblyFlaky() {
    XCTExpectFailure("已知不稳定，正在修复中 — #ISSUE-123")
    // ... 测试代码
}
```

---

## 无障碍测试

### accessibilityIdentifier 最佳实践

```swift
// 命名规范：[页面]_[元素类型]_[用途]
// 好：清晰、唯一、不随文本变化
Button("提交") { ... }
    .accessibilityIdentifier("checkout_button_submit")

TextField("搜索", text: $query)
    .accessibilityIdentifier("home_textfield_search")

// 列表项使用索引或唯一标识
ForEach(products) { product in
    ProductCell(product: product)
        .accessibilityIdentifier("productList_cell_\(product.id)")
}

// 避免：
// - 使用中文作为 identifier（本地化时会变）
// - 使用 "button1", "label2" 等无意义命名
// - 多个元素使用相同 identifier
```

### VoiceOver 导航测试

```swift
func test_homeScreen_voiceOverOrder() {
    app.launch()

    // 获取所有可被 VoiceOver 访问的元素
    let accessibleElements = app.descendants(matching: .any)
        .allElementsBoundByAccessibilityElement

    // 验证关键元素有 accessibilityLabel
    let searchField = app.textFields["home_textfield_search"]
    XCTAssertFalse(searchField.label.isEmpty, "搜索框缺少 accessibilityLabel")

    // 验证图片有描述
    let avatar = app.images["profile_image_avatar"]
    XCTAssertFalse(avatar.label.isEmpty, "头像缺少无障碍描述")

    // 验证按钮的 label 描述了操作而非外观
    let menuButton = app.buttons["nav_button_menu"]
    XCTAssertNotEqual(menuButton.label, "三条横线")  // 不应描述外观
    // 应该是 "菜单" 或 "打开导航菜单"
}

// 验证自定义组件的无障碍特性
func test_ratingView_accessibility() {
    app.launch()

    let ratingView = app.otherElements["rating_view"]
    XCTAssertTrue(ratingView.exists)

    // 验证 value 属性（VoiceOver 朗读的值）
    XCTAssertEqual(ratingView.value as? String, "4 星，共 5 星")
}
```

### Dynamic Type 测试

```swift
// 使用快照测试验证 Dynamic Type 适配
// 放在快照测试文件中（XCTest），此处展示思路

func test_productCard_dynamicType() {
    let view = ProductCardView(product: .mock)

    // 默认字体大小
    assertSnapshot(
        of: view.environment(\.sizeCategory, .medium),
        as: .image(layout: .fixed(width: 375, height: 200)),
        named: "medium"
    )

    // 最大字体
    assertSnapshot(
        of: view.environment(\.sizeCategory, .accessibilityExtraExtraExtraLarge),
        as: .image(layout: .fixed(width: 375, height: 400)),
        named: "xxxLarge"
    )

    // 最小字体
    assertSnapshot(
        of: view.environment(\.sizeCategory, .extraSmall),
        as: .image(layout: .fixed(width: 375, height: 150)),
        named: "extraSmall"
    )
}

// UI 测试中验证大字体下内容不被截断
func test_largeText_contentNotTruncated() {
    app.launchArguments += ["-UIPreferredContentSizeCategoryName", "UICTContentSizeCategoryAccessibilityXXXL"]
    app.launch()

    let title = app.staticTexts["productDetail_label_title"]
    XCTAssertTrue(title.exists)
    // 验证文本没有被截断（label 和 value 一致）
    XCTAssertFalse(title.label.hasSuffix("…"), "标题在大字体下被截断")
}
```

### 颜色对比度检查

```swift
// 颜色对比度主要依赖设计审查和自动化工具
// 在测试中可以验证 Dark Mode 下关键元素存在且可见

func test_darkMode_elementsVisible() {
    // 使用 Dark Mode 启动
    app.launchArguments += ["-UIUserInterfaceStyle", "Dark"]
    app.launch()

    // 验证关键文本在 Dark Mode 下存在且可交互
    let title = app.staticTexts["home_label_title"]
    XCTAssertTrue(title.waitForExistence(timeout: 3))
    XCTAssertTrue(title.isHittable, "标题在 Dark Mode 下不可见或被遮挡")

    let button = app.buttons["home_button_action"]
    XCTAssertTrue(button.isEnabled, "操作按钮在 Dark Mode 下不可用")
}

// 使用 Xcode Accessibility Inspector 进行对比度审计（手动步骤）：
// 1. Xcode → Open Developer Tool → Accessibility Inspector
// 2. 选择模拟器，使用 "Audit" 功能扫描当前页面
// 3. 检查 "Color Contrast" 类别的警告
// 4. WCAG 2.1 AA 标准：普通文本对比度 ≥ 4.5:1，大文本 ≥ 3:1
```

---

## 速查 Checklist

> 完整的测试质量标准见 `test-quality-checklist.md`，此处仅列 UI 测试专项检查。

- [ ] 使用 `accessibilityIdentifier` 定位元素，不使用索引
- [ ] 使用 Page Object 封装页面元素和操作
- [ ] 每个测试独立运行，不依赖其他测试的状态
- [ ] 使用 Launch Arguments/Environment 控制测试环境
- [ ] 禁用动画以提高测试稳定性
- [ ] 用 `waitForExistence` 等待元素，不用 `sleep`
- [ ] 处理系统弹窗（权限、通知等）
- [ ] 只为核心用户路径和复杂交互编写 UI 测试
- [ ] Flaky Test 立即标记并限期修复
- [ ] CI 上 UI 测试使用固定模拟器配置
