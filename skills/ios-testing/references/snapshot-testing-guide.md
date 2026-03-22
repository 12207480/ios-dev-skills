# 快照测试指南

## swift-snapshot-testing 框架使用

### 基本用法

```swift
import SnapshotTesting
import XCTest
@testable import MyApp

final class ProfileViewSnapshotTests: XCTestCase {
    // 首次运行自动生成基线快照，之后每次运行对比差异
    func test_profileView_normalState() {
        let view = ProfileView(
            profile: Profile(name: "张三", bio: "iOS 开发者", avatar: nil)
        )
        assertSnapshot(of: view, as: .image)
    }

    // 指定设备尺寸
    func test_profileView_iPhone15() {
        let view = ProfileView(profile: .mock)
        assertSnapshot(
            of: view,
            as: .image(layout: .device(config: .iPhone13))
        )
    }

    // 指定固定尺寸
    func test_profileCard_fixedSize() {
        let view = ProfileCardView(profile: .mock)
        assertSnapshot(
            of: view,
            as: .image(layout: .fixed(width: 375, height: 200))
        )
    }
}
```

### 视图快照

```swift
// SwiftUI 视图快照
func test_settingsView() {
    let view = SettingsView()
        .environmentObject(SettingsStore.mock)
    assertSnapshot(of: view, as: .image(layout: .device(config: .iPhone13)))
}

// UIKit 视图快照
func test_customButton() {
    let button = PrimaryButton(title: "提交订单")
    button.frame = CGRect(x: 0, y: 0, width: 200, height: 50)
    assertSnapshot(of: button, as: .image)
}

// 带导航栏的页面
func test_detailPage_withNavBar() {
    let vc = DetailViewController()
    vc.viewModel = DetailViewModel.mock
    let nav = UINavigationController(rootViewController: vc)
    assertSnapshot(of: nav, as: .image(on: .iPhone13))
}
```

### ViewController 快照

```swift
func test_productListVC_loadedState() {
    let vc = ProductListViewController()
    vc.viewModel = ProductListViewModel(products: Product.mockList)

    // 触发视图加载
    vc.loadViewIfNeeded()
    vc.viewModel.state = .loaded

    assertSnapshot(of: vc, as: .image(on: .iPhone13))
}

func test_productListVC_emptyState() {
    let vc = ProductListViewController()
    vc.viewModel = ProductListViewModel(products: [])
    vc.loadViewIfNeeded()
    vc.viewModel.state = .empty

    assertSnapshot(of: vc, as: .image(on: .iPhone13))
}

func test_productListVC_errorState() {
    let vc = ProductListViewController()
    vc.viewModel = ProductListViewModel(products: [])
    vc.loadViewIfNeeded()
    vc.viewModel.state = .error("加载失败，请重试")

    assertSnapshot(of: vc, as: .image(on: .iPhone13))
}
```

### 多设备/多尺寸快照

```swift
func test_onboardingView_multipleDevices() {
    let view = OnboardingView()

    // iPhone SE（小屏）
    assertSnapshot(
        of: view,
        as: .image(layout: .device(config: .iPhoneSe)),
        named: "iPhoneSE"
    )

    // iPhone 15（标准屏）
    assertSnapshot(
        of: view,
        as: .image(layout: .device(config: .iPhone13)),
        named: "iPhone15"
    )

    // iPhone 15 Pro Max（大屏）
    assertSnapshot(
        of: view,
        as: .image(layout: .device(config: .iPhone13ProMax)),
        named: "iPhone15ProMax"
    )
}

// Dark Mode 快照
func test_settingsView_darkMode() {
    let view = SettingsView()
        .environmentObject(SettingsStore.mock)
        .environment(\.colorScheme, .dark)
    assertSnapshot(
        of: view,
        as: .image(layout: .device(config: .iPhone13)),
        named: "dark"
    )
}

// 多语言快照
func test_loginView_localized() {
    for locale in ["zh-Hans", "en"] {
        let view = LoginView()
            .environment(\.locale, Locale(identifier: locale))
        assertSnapshot(
            of: view,
            as: .image(layout: .device(config: .iPhone13)),
            named: locale
        )
    }
}
```

---

## SwiftUI Preview 快照测试

```swift
// 利用已有的 Preview 数据进行快照测试
func test_profileView_previewStates() {
    // 正常状态
    assertSnapshot(
        of: ProfileView(profile: .previewMock),
        as: .image(layout: .device(config: .iPhone13)),
        named: "normal"
    )

    // 长文本状态
    assertSnapshot(
        of: ProfileView(profile: .previewLongText),
        as: .image(layout: .device(config: .iPhone13)),
        named: "longText"
    )

    // 空头像状态
    assertSnapshot(
        of: ProfileView(profile: .previewNoAvatar),
        as: .image(layout: .device(config: .iPhone13)),
        named: "noAvatar"
    )
}
```

---

## 快照更新策略

### 何时更新基线

| 场景 | 操作 |
|------|------|
| 有意的 UI 变更（设计稿更新） | 删除旧快照文件，重新运行生成新基线 |
| 升级系统版本导致渲染差异 | 统一团队 Xcode/模拟器版本后重新生成 |
| 新增视图状态 | 添加新的测试方法 |
| 重构未改变外观 | 快照测试应该自动通过，无需更新 |

### 审查快照差异

```swift
// 设置记录模式强制更新所有快照（谨慎使用）
// 环境变量方式
// SNAPSHOT_TESTING_RECORD=1 xcodebuild test ...

// 代码方式（单个测试）
func test_updateBaseline() {
    withSnapshotTesting(record: .all) {
        assertSnapshot(of: MyView(), as: .image)
    }
}
```

**审查流程：**
1. PR 中包含快照变更时，必须审查 `__Snapshots__` 目录的 diff
2. 使用图片 diff 工具对比新旧快照
3. 确认变更符合设计预期
4. 不相关的快照变更说明存在问题（环境差异 / 不稳定渲染）

### CI 中的快照测试配置

```yaml
# CI 配置要点
# 1. 固定模拟器型号和系统版本
# 2. 固定 Xcode 版本
# 3. 快照文件纳入版本控制

# GitHub Actions 示例
- name: Run Snapshot Tests
  run: |
    xcodebuild test \
      -scheme MyApp \
      -destination 'platform=iOS Simulator,name=iPhone 15,OS=18.0' \
      -only-testing:MyAppTests/SnapshotTests
```

---

## 快照测试的陷阱

### 日期/时间导致的不稳定

```swift
// 错误：视图中显示当前日期，每天快照都不同
struct OrderView: View {
    var body: some View {
        Text("下单时间：\(Date(), style: .date)")  // 每天变化
    }
}

// 正确：注入固定日期
struct OrderView: View {
    let orderDate: Date
    var body: some View {
        Text("下单时间：\(orderDate, style: .date)")
    }
}

func test_orderView() {
    let fixedDate = DateComponents(
        calendar: .current, year: 2025, month: 1, day: 1
    ).date!
    let view = OrderView(orderDate: fixedDate)
    assertSnapshot(of: view, as: .image)
}
```

### 动画导致的不稳定

```swift
// 错误：视图有动画，截图时机不确定
// 正确：快照测试中禁用动画

func test_animatedView() {
    // 方式一：使用 transaction 禁用动画
    var transaction = Transaction()
    transaction.disableAnimations = true
    let view = AnimatedView()
        .transaction { $0.disableAnimations = true }

    assertSnapshot(of: view, as: .image)
}
```

### 系统版本差异

不同 iOS 版本的默认字体渲染、系统控件样式可能不同。

**解决方案：**
- 团队统一 Xcode 和模拟器版本
- CI 环境使用固定版本
- 如需支持多版本，按版本生成不同基线

```swift
func test_view_iOS17() {
    // 使用 named 参数区分不同环境的基线
    let osVersion = ProcessInfo.processInfo.operatingSystemVersion
    let name = "iOS\(osVersion.majorVersion)"
    assertSnapshot(of: MyView(), as: .image, named: name)
}
```

---

## 与 Swift Testing 集成

swift-snapshot-testing 当前主要基于 XCTest。在 Swift Testing 中使用需注意：

```swift
// 方式一：快照测试仍使用 XCTest（推荐，官方支持最好）
import XCTest
import SnapshotTesting

final class MySnapshotTests: XCTestCase {
    func test_myView() {
        assertSnapshot(of: MyView(), as: .image)
    }
}

// 方式二：在 Swift Testing 中使用（需要框架支持）
// swift-snapshot-testing 1.17+ 支持 Swift Testing
import Testing
import SnapshotTesting

@Suite("视图快照")
struct ViewSnapshotTests {
    @Test("首页正常状态")
    func homeNormalState() {
        // 图片快照仍使用 assertSnapshot，需要外部基线文件
        assertSnapshot(of: HomeView(state: .loaded), as: .image)
    }
}

// assertInlineSnapshot 适用于文本策略（dump / description）
// inline snapshot 将期望值直接嵌入代码中，适合结构化数据验证
@Suite("数据模型快照")
struct ModelSnapshotTests {
    @Test("用户模型 dump")
    func userModelDump() {
        let user = User(id: "123", name: "张三", email: "zhang@example.com")
        assertInlineSnapshot(of: user, as: .dump) {
            """
            ▿ User
              - email: "zhang@example.com"
              - id: "123"
              - name: "张三"
            """
        }
    }

    @Test("JSON 序列化格式")
    func jsonOutput() throws {
        let order = Order(id: "ORD-001", amount: 99, status: .pending)
        let encoder = JSONEncoder()
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
        let jsonString = String(data: try encoder.encode(order), encoding: .utf8)!
        assertInlineSnapshot(of: jsonString, as: .lines) {
            """
            {
              "amount" : 99,
              "id" : "ORD-001",
              "status" : "pending"
            }
            """
        }
    }
}
```

**建议：** 快照测试文件独立存放，使用 XCTest 编写。单元测试使用 Swift Testing。两者可以共存于同一个 Test Target。

---

## 速查 Checklist

> 完整的测试质量标准见 `test-quality-checklist.md`，此处仅列快照测试专项检查。

- [ ] 快照文件（`__Snapshots__`）纳入版本控制
- [ ] 注入固定日期/时间，避免快照每天变化
- [ ] 禁用动画，避免截图时机不确定
- [ ] 团队统一 Xcode 和模拟器版本
- [ ] 多设备尺寸使用 `named` 参数区分基线
- [ ] Dark Mode / Light Mode 分别生成快照
- [ ] PR 审查时检查 `__Snapshots__` 目录的 diff
- [ ] CI 使用固定模拟器型号和系统版本
- [ ] 新增视图状态时添加对应的快照测试
- [ ] 不要对频繁变化的 UI 写快照测试
