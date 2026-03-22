# UIKit → SwiftUI 迁移

## 迁移策略

### 策略选择决策

| 策略 | 适用场景 | 优点 | 风险 |
|------|---------|------|------|
| **新页面用 SwiftUI** | 项目仍在活跃开发，持续有新页面 | 零风险，渐进过渡 | 新旧页面体验可能不一致 |
| **组件级迁移** | 页面中有独立的 UI 组件可以替换 | 影响面小，可逐步验证 | 需要处理 UIKit/SwiftUI 交互 |
| **整页迁移** | 页面相对独立，依赖明确 | 一次到位，代码更干净 | 迁移成本高，风险集中 |

### 迁移优先级排序

```
优先迁移：
├── 纯展示页面（无复杂交互）
├── 设置/配置页面（表单类）
├── 独立的弹窗/底部弹出
└── 新增的页面（直接用 SwiftUI）

暂缓迁移：
├── 复杂自定义布局（CollectionView 复杂布局）
├── 高性能列表（大数据量、复杂 Cell）
├── 重度依赖 UIKit 特性的页面（手势、动画）
└── 深度嵌套在 UIKit 导航栈中的页面
```

---

## UIHostingController 嵌入模式

### 在 UIKit 中嵌入 SwiftUI 视图

```swift
// 基本用法：将 SwiftUI 视图嵌入 UIKit 导航栈
let swiftUIView = ProfileView(user: user)
let hostingController = UIHostingController(rootView: swiftUIView)
navigationController?.pushViewController(hostingController, animated: true)

// 作为子视图控制器嵌入
func embedSwiftUIView() {
    let swiftUIView = StatusBanner(status: .active)
    let hostingController = UIHostingController(rootView: swiftUIView)

    addChild(hostingController)
    view.addSubview(hostingController.view)
    hostingController.view.translatesAutoresizingMaskIntoConstraints = false
    NSLayoutConstraint.activate([
        hostingController.view.topAnchor.constraint(equalTo: headerView.bottomAnchor),
        hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
        hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
        hostingController.view.heightAnchor.constraint(equalToConstant: 44)
    ])
    hostingController.didMove(toParent: self)
}
```

### 传递数据和回调

```swift
// SwiftUI 视图接收回调
struct ActionButton: View {
    let title: String
    let onTap: () -> Void

    var body: some View {
        Button(title, action: onTap)
    }
}

// UIKit 侧创建并传递回调
let actionButton = ActionButton(title: "提交") { [weak self] in
    self?.submitForm()
}
let hostingController = UIHostingController(rootView: actionButton)
```

### 注意事项

```
- UIHostingController 默认背景是不透明的
  → 设置 hostingController.view.backgroundColor = .clear
- UIHostingController 会根据内容自动调整大小
  → 如需固定大小，添加约束或设置 preferredContentSize
- 避免频繁创建/销毁 UIHostingController
  → 如果数据变化频繁，使用 @Observable 或 Binding 更新，而非重建
```

---

## UIViewRepresentable 包装模式

### 在 SwiftUI 中使用 UIKit 视图

```swift
// 包装 UIKit 视图
struct MapViewWrapper: UIViewRepresentable {
    @Binding var region: MKCoordinateRegion
    let annotations: [MKAnnotation]

    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        mapView.delegate = context.coordinator
        return mapView
    }

    func updateUIView(_ mapView: MKMapView, context: Context) {
        mapView.setRegion(region, animated: true)
        mapView.removeAnnotations(mapView.annotations)
        mapView.addAnnotations(annotations)
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, MKMapViewDelegate {
        let parent: MapViewWrapper

        init(_ parent: MapViewWrapper) {
            self.parent = parent
        }

        func mapView(_ mapView: MKMapView, regionDidChangeAnimated animated: Bool) {
            parent.region = mapView.region
        }
    }
}
```

### UIViewControllerRepresentable

```swift
// 包装 UIKit ViewController
struct ImagePickerWrapper: UIViewControllerRepresentable {
    @Binding var selectedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: ImagePickerWrapper

        init(_ parent: ImagePickerWrapper) {
            self.parent = parent
        }

        func imagePickerController(_ picker: UIImagePickerController,
                                   didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]) {
            parent.selectedImage = info[.originalImage] as? UIImage
            parent.dismiss()
        }
    }
}
```

---

## 状态管理迁移

### Delegate/KVO → @Observable

```swift
// === UIKit：Delegate + KVO ===
protocol CartDelegate: AnyObject {
    func cartDidUpdate(_ cart: Cart)
}

class Cart {
    weak var delegate: CartDelegate?
    var items: [CartItem] = [] {
        didSet { delegate?.cartDidUpdate(self) }
    }
}

// ViewController 中
class CartViewController: UIViewController, CartDelegate {
    func cartDidUpdate(_ cart: Cart) {
        tableView.reloadData()
    }
}

// === SwiftUI：@Observable ===
@Observable
class Cart {
    var items: [CartItem] = []

    var totalPrice: Decimal {
        items.reduce(0) { $0 + $1.price }
    }
}

// SwiftUI View 中自动响应变化
struct CartView: View {
    var cart: Cart

    var body: some View {
        List(cart.items) { item in
            CartItemRow(item: item)
        }
        Text("总计: \(cart.totalPrice)")
    }
}
```

### 混编期间的状态共享

```swift
// 使用 @Observable 类同时服务 UIKit 和 SwiftUI
@Observable
class UserSession {
    var currentUser: User?
    var isLoggedIn: Bool { currentUser != nil }
}

// SwiftUI 中直接使用
struct ProfileView: View {
    var session: UserSession
    var body: some View {
        if let user = session.currentUser {
            Text(user.name)
        }
    }
}

// UIKit 中通过 Combine 观察
class SettingsViewController: UIViewController {
    private var cancellables = Set<AnyCancellable>()

    func observeSession(_ session: UserSession) {
        // 使用 withObservationTracking 在 UIKit 中观察 @Observable 对象
        func observe() {
            withObservationTracking {
                // 读取需要观察的属性，触发追踪注册
                let user = session.currentUser
                let isLoggedIn = session.isLoggedIn

                // 在主线程更新 UI
                DispatchQueue.main.async { [weak self] in
                    if isLoggedIn {
                        self?.nameLabel.text = user?.name
                        self?.emailLabel.text = user?.email
                        self?.loginButton.isHidden = true
                    } else {
                        self?.nameLabel.text = "未登录"
                        self?.emailLabel.text = nil
                        self?.loginButton.isHidden = false
                    }
                }
            } onChange: {
                // 属性变化时重新调用 observe 以持续观察
                DispatchQueue.main.async {
                    observe()
                }
            }
        }
        observe()
    }
}
```

---

## 导航迁移

### UINavigationController → NavigationStack

```swift
// === UIKit 导航 ===
// 使用 Coordinator 或直接 push
func showDetail(item: Item) {
    let detailVC = ItemDetailViewController(item: item)
    navigationController?.pushViewController(detailVC, animated: true)
}

// === SwiftUI 导航 ===
struct ItemListView: View {
    let items: [Item]

    var body: some View {
        NavigationStack {
            List(items) { item in
                NavigationLink(value: item) {
                    ItemRow(item: item)
                }
            }
            .navigationDestination(for: Item.self) { item in
                ItemDetailView(item: item)
            }
        }
    }
}
```

### 混编导航策略

```
UIKit 导航栈中嵌入 SwiftUI 页面：
  → 使用 UIHostingController push/present
  → SwiftUI 页面通过回调通知 UIKit 导航

SwiftUI 导航栈中嵌入 UIKit 页面：
  → 使用 UIViewControllerRepresentable
  → 通过 Environment 或 Binding 传递导航动作

过渡期建议：
  → 保持一个主导航栈（通常是 UIKit 的 UINavigationController）
  → SwiftUI 页面作为叶子节点嵌入
  → 待大部分页面迁移后再切换到 NavigationStack
```

---

## 常见 UIKit 组件的 SwiftUI 等价物

| UIKit | SwiftUI | 迁移注意事项 |
|-------|---------|------------|
| `UILabel` | `Text` | SwiftUI 的 Text 默认多行，注意 `lineLimit` |
| `UIButton` | `Button` | 样式通过 `buttonStyle` 修改 |
| `UIImageView` | `Image` / `AsyncImage` | 网络图片用 `AsyncImage` 或第三方库 |
| `UITextField` | `TextField` | 键盘类型通过 `.keyboardType` 修改 |
| `UITextView` | `TextEditor` | SwiftUI 的 TextEditor 功能有限 |
| `UISwitch` | `Toggle` | 样式差异注意 |
| `UISlider` | `Slider` | 接口类似 |
| `UITableView` | `List` | 性能差异注意，大数据量慎用 |
| `UICollectionView` | `LazyVGrid` / `LazyHGrid` | 复杂布局仍建议用 UIKit |
| `UIScrollView` | `ScrollView` | 嵌套滚动需要注意 |
| `UIStackView` | `VStack` / `HStack` / `ZStack` | 直接对应 |
| `UIAlertController` | `.alert` / `.confirmationDialog` | 通过 modifier 触发 |
| `UIActivityIndicatorView` | `ProgressView` | 样式略有不同 |
| `UIPageViewController` | `TabView(.page)` | 功能更受限 |
| `UITabBarController` | `TabView` | iOS 18+ 支持更多自定义 |

### iOS 18 新增 SwiftUI API

| 新 API | 用途 | 迁移说明 |
|--------|------|---------|
| `@Entry` macro | 自定义 `EnvironmentValues` / `FocusValues` / `ContainerValues` | 替代手动编写 `EnvironmentKey` + 扩展的样板代码，显著减少代码量 |
| 新 `TabView` API | 类型安全的标签页定义 + 可自定义侧边栏 | 使用 `Tab("标题", systemImage:)` 声明标签页，支持 `TabSection` 分组 |
| `@Previewable` | 在 `#Preview` 中直接使用 `@State` | 预览代码更简洁，无需包装 View |
| `MeshGradient` | 网格渐变效果 | UIKit 无直接等价物，SwiftUI 独有特性 |

```swift
// === iOS 18 @Entry 宏：简化 EnvironmentValues ===
// Before（iOS 17 及以前）
struct ThemeColorKey: EnvironmentKey {
    static let defaultValue: Color = .blue
}
extension EnvironmentValues {
    var themeColor: Color {
        get { self[ThemeColorKey.self] }
        set { self[ThemeColorKey.self] = newValue }
    }
}

// After（iOS 18 @Entry 宏）
extension EnvironmentValues {
    @Entry var themeColor: Color = .blue
}

// === iOS 18 新 TabView API ===
// Before（iOS 17 及以前）
TabView {
    HomeView()
        .tabItem {
            Label("首页", systemImage: "house")
        }
    ProfileView()
        .tabItem {
            Label("我的", systemImage: "person")
        }
}

// After（iOS 18 新 API）
TabView {
    Tab("首页", systemImage: "house") {
        HomeView()
    }
    Tab("我的", systemImage: "person") {
        ProfileView()
    }
}
```

---

## 不应迁移的场景

```
以下场景建议保留 UIKit 实现：

1. 复杂自定义布局
   - UICollectionView 的 CompositionalLayout
   - 需要像素级控制的布局
   - 瀑布流等非标准布局

2. 高性能列表
   - 数据量 > 1000 条的列表
   - 复杂 Cell（多层嵌套、异步加载）
   - 需要预加载和缓存优化

3. 复杂动画
   - Core Animation 深度使用
   - 交互式转场动画
   - 粒子效果、自定义绘制

4. 系统级集成
   - 深度使用 UIAppearance
   - 自定义 UINavigationBar 行为
   - 需要精确控制 ViewController 生命周期

5. 第三方 SDK 依赖
   - SDK 提供 UIView/UIViewController
   - 无法用 Representable 简单包装
```

---

## 迁移 Checklist

```
迁移前评估：
- [ ] 目标页面/组件的职责边界清晰
- [ ] 识别了所有 UIKit 特有的依赖（手势、动画、生命周期）
- [ ] 确认最低 iOS 版本支持目标 SwiftUI 特性
- [ ] 确认不属于"不应迁移"的场景
- [ ] 现有测试覆盖充分（或已补充）

迁移执行：
- [ ] 状态管理已迁移到 @Observable 或适当的模式
- [ ] UIKit 组件已替换为 SwiftUI 等价物
- [ ] 导航逻辑已适配（UIHostingController 或 NavigationStack）
- [ ] 数据传递使用 SwiftUI 惯用模式（@Binding / @Environment）
- [ ] 需要保留的 UIKit 视图已正确包装（UIViewRepresentable）

迁移后验证：
- [ ] 视觉效果与迁移前一致（截图对比）
- [ ] 交互行为与迁移前一致（手势、动画、响应）
- [ ] 状态管理正确（数据流、响应式更新）
- [ ] 导航正确（push/pop/present/dismiss）
- [ ] 适配验证（Dark Mode / Dynamic Type / 多屏幕尺寸）
- [ ] 性能无回退（尤其是滚动和动画帧率）
- [ ] 无障碍访问功能正常
- [ ] 内存无泄漏（UIHostingController 的引用管理）
```
