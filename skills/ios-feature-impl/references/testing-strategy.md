# 测试策略

## 测试覆盖优先级

### 1. 新增/修改的业务逻辑（必须）
- ViewModel 的状态流转
- 关键的条件分支和计算逻辑
- 数据模型的 JSON 解析

### 2. 与现有逻辑的交互点（必须）
- 修改了公共方法 → 确保现有调用方的测试仍通过
- 修改了数据模型 → 确保旧数据格式仍能解析
- 修改了状态管理 → 确保其他页面的状态同步正常

### 3. 异常路径（应该）
- 网络失败 / 空数据 / 非法输入 / 权限拒绝

### 4. UI 验证（关键页面）
- 快照测试覆盖关键状态
- XCUITest 覆盖核心操作路径

## Swift Testing 框架

使用 Swift Testing 替代 XCTest（iOS 16+ / Xcode 16+）：

### 基础用法

```swift
import Testing

@Suite("购物车逻辑测试")
struct CartTests {
    @Test("添加商品后数量增加")
    func addItem() {
        let cart = Cart()
        cart.add(Item(name: "iPhone", price: 999))
        #expect(cart.items.count == 1)
        #expect(cart.totalPrice == 999)
    }

    @Test("空购物车总价为零")
    func emptyCartTotal() {
        let cart = Cart()
        #expect(cart.totalPrice == 0)
    }
}
```

### 参数化测试

```swift
@Test("折扣计算", arguments: [
    (100, 0.1, 90),
    (200, 0.2, 160),
    (50, 0.0, 50),
])
func discount(price: Int, rate: Double, expected: Int) {
    let result = calculateDiscount(price: price, rate: rate)
    #expect(result == expected)
}
```

### 异步测试

```swift
@Test("加载用户资料")
func loadProfile() async throws {
    let viewModel = ProfileViewModel(service: MockProfileService())
    await viewModel.loadProfile()
    #expect(viewModel.profile != nil)
    #expect(viewModel.profile?.name == "测试用户")
}
```

### 断言对比

| XCTest | Swift Testing |
|--------|--------------|
| `XCTAssertEqual(a, b)` | `#expect(a == b)` |
| `XCTAssertTrue(x)` | `#expect(x)` |
| `XCTAssertNil(x)` | `#expect(x == nil)` |
| `XCTAssertThrowsError` | `#expect(throws:)` |
| `func test_xxx()` | `@Test func xxx()` |
| `class XxxTests: XCTestCase` | `@Suite struct XxxTests` |

## 快照测试

使用 swift-snapshot-testing 进行 UI 回归验证：

```swift
import SnapshotTesting

@Suite("个人资料页快照")
struct ProfileViewSnapshotTests {
    @Test("正常状态")
    func normalState() {
        let view = ProfileView(profile: .mock)
        assertSnapshot(of: view, as: .image(layout: .device(config: .iPhone13)))
    }

    @Test("空状态")
    func emptyState() {
        let view = ProfileView(profile: nil)
        assertSnapshot(of: view, as: .image(layout: .device(config: .iPhone13)))
    }
}
```

## 测试命名规范

```
// Swift Testing
@Test("[方法名] [条件] [预期结果]")
func descriptiveName() { ... }

// XCTest（如项目仍在使用）
func test_[方法名]_[条件]_[预期结果]()
```

示例：
- `@Test("loadData 网络失败时 状态变为error")`
- `func test_loadData_networkFailure_stateBecomesError()`
