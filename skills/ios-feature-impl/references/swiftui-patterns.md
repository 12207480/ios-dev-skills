# SwiftUI 现代模式

## 状态管理

### 推荐模式（iOS 17+）

| 属性包装器 | 用途 | 示例 |
|-----------|------|------|
| `@Observable` | 业务逻辑和应用状态 | ViewModel、共享状态 |
| `@Bindable` | 需要绑定的 @Observable 对象 | 表单绑定 |
| `@State` | 视图本地状态 | 展开/折叠、文本输入 |
| `@Environment` | 依赖注入 | Service、Router |

### 避免使用（除非项目已有这些模式）

- `@StateObject` / `@ObservedObject` / `@Published` / `ObservableObject` / `@EnvironmentObject`
- 这些是旧模式，`@Observable` 是更高效的替代

```swift
// 推荐：@Observable
@Observable
final class CartViewModel {
    var items: [CartItem] = []
    var totalPrice: Decimal { items.reduce(0) { $0 + $1.price } }

    func addItem(_ item: CartItem) {
        items.append(item)
    }
}

// 视图中使用
struct CartView: View {
    @Bindable var viewModel: CartViewModel

    var body: some View {
        List($viewModel.items) { $item in
            CartItemRow(item: $item)
        }
    }
}

// 依赖注入
struct ContentView: View {
    @Environment(CartViewModel.self) private var cart
}
```

## 现代导航

### NavigationStack + 类型安全路由

```swift
enum Route: Hashable {
    case detail(Item)
    case settings
    case profile(userId: String)
}

struct RootView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: Route.self) { route in
                    switch route {
                    case .detail(let item): DetailView(item: item)
                    case .settings: SettingsView()
                    case .profile(let id): ProfileView(userId: id)
                    }
                }
        }
    }
}
```

### NavigationSplitView 多栏布局

```swift
struct MainView: View {
    @State private var selectedCategory: Category?
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView {
            CategoryList(selection: $selectedCategory)
        } content: {
            ItemList(category: selectedCategory, selection: $selectedItem)
        } detail: {
            ItemDetail(item: selectedItem)
        }
    }
}
```

## 布局系统

### Grid 复杂布局

```swift
Grid(alignment: .leading, horizontalSpacing: 12, verticalSpacing: 8) {
    GridRow {
        Text("名称")
        Text(item.name)
    }
    GridRow {
        Text("状态")
        StatusBadge(status: item.status)
    }
}
```

### ViewThatFits 自适应界面

```swift
ViewThatFits(in: .horizontal) {
    HStack { /* 宽屏布局 */ }
    VStack { /* 窄屏布局 */ }
}
```

### containerRelativeFrame 响应式尺寸

```swift
ScrollView(.horizontal) {
    LazyHStack {
        ForEach(items) { item in
            CardView(item: item)
                .containerRelativeFrame(.horizontal, count: 3, spacing: 8)
        }
    }
}
```

## 性能模式

### 拆分视图优于 computed property

```swift
// 推荐：拆分为独立子视图，SwiftUI 可精确追踪依赖
struct ItemRow: View {
    let item: Item
    var body: some View {
        HStack {
            ItemIcon(item: item)    // 独立子视图
            ItemInfo(item: item)    // 独立子视图
        }
    }
}
```

### 其他性能要点

- **避免 `AnyView`**：使用 `@ViewBuilder` 或 `some View` 返回类型
- **大列表用 `LazyVStack` / `LazyHStack`**：避免一次性创建所有视图
- **`task()` 优于 `onAppear() + Task {}`**：自动管理任务生命周期和取消
- **view body 中避免重计算**：将计算移到 ViewModel 或缓存为属性

```swift
// 推荐
.task {
    await viewModel.loadData()
}

// 避免
.onAppear {
    Task {
        await viewModel.loadData()
    }
}
```

## 交互与动画

### 动画触发

```swift
// 值驱动动画
.animation(.spring, value: isExpanded)

// 显式动画
withAnimation(.easeInOut(duration: 0.3)) {
    isExpanded.toggle()
}
```

### SF Symbol 效果

```swift
Image(systemName: "bell")
    .symbolEffect(.bounce, value: notificationCount)
```

### 触觉反馈

```swift
Button("添加") { addItem() }
    .sensoryFeedback(.success, trigger: itemAdded)
```

## 无障碍

所有可交互元素必须满足以下要求：

- **`accessibilityLabel`**：为所有图标按钮和非文字元素提供标签
- **Dynamic Type**：使用系统字体或 `@ScaledMetric` 支持动态字号
- **VoiceOver**：确保导航顺序合理，交互元素可聚焦
- **Reduce Motion**：尊重 `accessibilityReduceMotion` 设置

```swift
@ScaledMetric private var iconSize = 24.0

Image(systemName: "plus")
    .frame(width: iconSize, height: iconSize)
    .accessibilityLabel("添加项目")

// 尊重减少动画设置
@Environment(\.accessibilityReduceMotion) private var reduceMotion

.animation(reduceMotion ? .none : .spring, value: isExpanded)
```

## iOS 18+ 新特性

### @Previewable 宏

简化 Preview 中的状态绑定，无需再创建包装视图：

```swift
// iOS 18+：直接在 #Preview 中使用 @Previewable
#Preview {
    @Previewable @State var text = "Hello"
    TextField("输入", text: $text)
        .padding()
}

// iOS 17 及之前：需要包装视图
struct TextFieldPreview: View {
    @State private var text = "Hello"
    var body: some View {
        TextField("输入", text: $text)
    }
}
#Preview { TextFieldPreview() }
```

### Mesh Gradient

创建多点控制的网格渐变效果：

```swift
// 网格渐变背景
MeshGradient(
    width: 3, height: 3,
    points: [
        [0, 0], [0.5, 0], [1, 0],
        [0, 0.5], [0.5, 0.5], [1, 0.5],
        [0, 1], [0.5, 1], [1, 1]
    ],
    colors: [
        .blue, .purple, .indigo,
        .cyan, .mint, .teal,
        .blue, .purple, .indigo
    ]
)
.ignoresSafeArea()
```

### Tab 视图增强

iOS 18 引入了类型安全的 Tab API 和可自定义的 Tab 栏：

```swift
// iOS 18+：新 Tab 语法
TabView {
    Tab("首页", systemImage: "house") {
        HomeView()
    }
    Tab("搜索", systemImage: "magnifyingglass") {
        SearchView()
    }
    Tab("设置", systemImage: "gear") {
        SettingsView()
    }
}
.tabViewStyle(.sidebarAdaptable)  // iPad 上自动适配侧边栏样式

// 支持 Tab 分区
TabView {
    TabSection("主要功能") {
        Tab("首页", systemImage: "house") { HomeView() }
        Tab("浏览", systemImage: "square.grid.2x2") { BrowseView() }
    }
    TabSection("其他") {
        Tab("设置", systemImage: "gear") { SettingsView() }
    }
}
```

> **注意：** iOS 18+ 新特性需要 `if #available(iOS 18.0, *)` 保护，或将最低部署版本设为 iOS 18。在混编项目中，这些特性仅适用于 SwiftUI 部分。
