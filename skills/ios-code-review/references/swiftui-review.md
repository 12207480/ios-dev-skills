# SwiftUI 审查

## 正确性清单

### 属性包装器使用

```
- [ ] @State 必须声明为 private
      → @State private var count = 0 ✅
      → @State var count = 0 ❌（外部不应直接修改视图的本地状态）

- [ ] @Binding 只在子视图需要修改父状态时使用
      → 仅读取 → 直接传入值即可，不需要 @Binding

- [ ] 传入的值不要声明为 @State 或 @StateObject
      → @State 是视图的本地状态源，传入值会被忽略更新
      → 正确做法：直接用 let 接收，或用 @Binding

- [ ] iOS 17+：@State 配合 @Observable 使用
      → @State private var viewModel = ViewModel()（ViewModel 标注 @Observable）
      → 不再需要 @StateObject / @ObservedObject

- [ ] @Bindable 用于需要绑定的注入 @Observable 对象
      → @Bindable var item: Item（当需要 $item.name 这样的绑定时）
```

### ForEach 与 Identity

```
- [ ] ForEach 使用稳定的 identity
      → ForEach(items, id: \.stableID) ✅
      → ForEach(items.indices, id: \.self) ❌（索引不稳定，增删会错乱）
      → ForEach(0..<items.count, id: \.self) ❌（同上）
      → 数据模型应遵循 Identifiable 协议
```

### 动画

```
- [ ] .animation(_:value:) 必须包含 value 参数
      → .animation(.easeInOut, value: isExpanded) ✅
      → .animation(.easeInOut) ❌（已废弃，会导致意外动画）

- [ ] withAnimation 优先用于触发明确的状态变更动画
```

### 版本兼容

```
- [ ] iOS 26+ API 必须用 #available 门控 + fallback
      → if #available(iOS 26, *) { NewView() } else { FallbackView() }
- [ ] 使用 @available 标注整个视图的最低版本要求

iOS 18+ 新 API 检查项：
- [ ] @Previewable 宏使用是否正确？
      → 仅用于 #Preview 中创建动态预览状态
      → #Preview { @Previewable @State var text = ""; TextField("输入", text: $text) }
      → 不要在生产代码中使用 @Previewable
- [ ] .meshGradient 使用是否合理？
      → 需要 #available(iOS 18, *) 门控
      → 提供 fallback 渐变方案（LinearGradient 等）
      → 控制网格点数量，过多会影响性能
- [ ] CustomTabView / Tab 新 API 使用是否正确？
      → iOS 18+ TabView 支持 Tab 类型安全构建
      → TabView { Tab("首页", systemImage: "house") { HomeView() } }
      → 低版本 fallback 使用 .tabItem { } 修饰符
- [ ] @Entry 宏用于 EnvironmentValues 扩展？
      → 替代手动定义 EnvironmentKey + 计算属性
      → extension EnvironmentValues { @Entry var myValue: String = "" }
      → 需要 iOS 18+ / macOS 15+
```

## 性能规则

### 避免不必要的重绘

```
- [ ] 避免 AnyView，用 @ViewBuilder / Group / 泛型替代
      → AnyView 会破坏 SwiftUI 的 diff 算法，导致全量重绘
      → @ViewBuilder 保留类型信息，性能更优

- [ ] ternary 优于 if/else（避免 _ConditionalContent 重建）
      → Text(isActive ? "活跃" : "不活跃") ✅
      → if isActive { Text("活跃") } else { Text("不活跃") } 🟡（会创建 _ConditionalContent）

- [ ] view body 中避免排序/过滤等重计算
      → 应在 ViewModel 或 onChange 中预处理
      → body 每次重绘都会执行，复杂计算造成卡顿
```

### 视图拆分与组合

```
- [ ] 拆分子视图优于 computed property
      → 独立 struct 可以独立 diff，减少不必要的重绘
      → computed property 每次父视图更新都会重新计算

- [ ] LazyVStack / LazyHStack 用于大数据集
      → 普通 VStack 会一次性创建所有子视图
      → Lazy 版本按需创建，内存和性能更优
```

### 异步与副作用

```
- [ ] task() 优于 onAppear() 做异步操作
      → task 自动管理取消，视图消失时自动取消
      → onAppear 中手动 Task {} 需要自行管理生命周期

- [ ] 避免内联 DateFormatter，用 Text(date, format:)
      → DateFormatter 创建代价高，不应在 body 中创建
      → Text(date, format: .dateTime.year().month().day()) ✅
```

## 状态管理审查

```
- [ ] @Observable vs ObservableObject 使用是否正确？
      → iOS 17+ 新项目应使用 @Observable
      → 混用时注意：@Observable 不需要 @Published 标注属性
      → ObservableObject 的 objectWillChange 是全量通知，@Observable 是属性级精准通知

- [ ] @Environment 依赖注入是否合理？
      → 避免通过 @Environment 传递业务数据（应用于系统级配置）
      → 自定义 EnvironmentKey 的默认值是否合理？
      → 是否在正确的层级注入？（过高会影响不需要的子树）

- [ ] @EnvironmentObject 是否有缺失注入的风险？
      → 缺失注入会在运行时崩溃，无编译期检查
      → iOS 17+ 建议迁移到 @Environment + @Observable
```

## 现代导航审查

```
- [ ] NavigationStack 使用是否正确？
      → path 绑定的类型是否为 Hashable
      → navigationDestination 是否覆盖所有路由类型
      → 避免在深层子视图中直接操作 NavigationPath（通过 Environment 传递）

- [ ] 类型安全路由是否完整？
      → 使用 enum 定义路由，避免字符串或 magic number
      → NavigationSplitView 的 selection 绑定是否正确

- [ ] 编程式导航是否可控？
      → 返回操作（dismiss）是否可靠
      → Deep Link 是否能正确构建导航栈
```

## 现代布局审查

```
- [ ] 使用 ViewThatFits 替代手动尺寸判断？
      → 自适应横竖屏、不同设备尺寸

- [ ] containerRelativeFrame 替代 GeometryReader？
      → GeometryReader 会占满可用空间，影响布局
      → containerRelativeFrame 更精准且不影响兄弟视图

- [ ] Grid 替代嵌套 HStack/VStack 的复杂对齐？
      → GridRow 保证列对齐
      → alignment 和 spacing 统一控制
```

## 交互与反馈审查

```
- [ ] 触觉反馈是否使用 .sensoryFeedback()？
      → 替代手动 UIImpactFeedbackGenerator
      → 声明式绑定到状态变化

- [ ] SF Symbol 动效是否使用 .symbolEffect()？
      → .bounce / .pulse / .variableColor
      → 替代手动动画

- [ ] 动画是否尊重 Reduce Motion？
      → @Environment(\.accessibilityReduceMotion)
      → 提供无动画或简化动画替代
```
