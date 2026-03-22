# SwiftUI 专项排查指南

## 视图不更新排查

### `Self._printChanges()` 诊断
在 View 的 `body` 中添加以下代码，查看视图重算原因：
```swift
var body: some View {
    let _ = Self._printChanges()
    // ...
}
```
输出示例：`MyView: @self, @identity, _myState changed.`

### `@State` 误用导致不更新
- **常见错误：** 将外部传入的值声明为 `@State`，父视图更新时子视图不会刷新
- **原因：** `@State` 仅在视图首次初始化时赋值，后续父视图传入的新值被忽略
- **修复：** 外部传入的值应使用 `let`（只读）或 `@Binding`（双向绑定）

### `@Observable` 属性修改未触发更新
- 确认属性是直接在 `body` 中访问的（非通过闭包/方法间接访问）
- 确认修改发生在主线程（`@MainActor`）
- 确认没有通过 `withMutation` 绕过观察机制
- 确认未将 `@Observable` 对象存储在 `@State` 中（iOS 17+ 可以，iOS 16 不行）

### `@Observable` 调试技巧
- **属性级追踪**：`@Observable` 只通知实际被 `body` 访问的属性变化，与 `ObservableObject` 的全量通知不同
- **未访问的属性变化不触发更新**：如果 `body` 中没有直接读取某个属性，修改该属性不会触发视图刷新
- **通过方法间接访问不被追踪**：在 `body` 中调用 `viewModel.formattedTitle()` 不会追踪 `title` 属性的变化，应改为直接访问 `viewModel.title` 或使用 computed property
- **`@ObservationIgnored`**：标记为 ignored 的属性变化永远不触发更新，确认是否误标
- **混用 `ObservableObject`**：如果项目混用两种模式，确认视图使用了正确的接收方式（`@State` vs `@StateObject`）

### `ForEach` identity 不稳定导致列表异常
- `ForEach` 要求每个元素有稳定唯一的 `id`
- 使用数组索引作为 `id` 会导致删除/插入时视图错乱
- **修复：** 让 Model 遵循 `Identifiable`，使用稳定的业务 ID

### `id()` modifier 误用导致视图重建
- `id()` 变化时 SwiftUI 会销毁并重建整个视图（非更新）
- 导致 `@State` 丢失、动画中断、性能下降
- **仅在需要强制重建时使用**（如切换用户后重置整个页面）

---

## 性能问题排查

### 使用 Instruments 检测掉帧
- 打开 Instruments → Core Animation 模板
- 关注 FPS（低于 60 说明掉帧）和 Commit 次数
- 检查 Render Loop 中是否有不必要的布局计算

### `LazyVStack` vs `VStack` 在大列表中的差异
- `VStack` 会一次性加载所有子视图 — 大列表必卡
- `LazyVStack` 按需加载，配合 `ScrollView` 使用
- 注意：`List` 内部已经是 lazy 的，不需要额外包装

### `AnyView` 导致的 diff 性能损失
- `AnyView` 擦除类型信息，SwiftUI 无法高效 diff
- **替代方案：** 使用 `@ViewBuilder`、`Group`、或具体的条件分支

### body 中的重计算
- 排序/过滤/格式化操作不应放在 `body` 中
- 移到 ViewModel 或使用 computed property + 缓存
- `DateFormatter`、`NumberFormatter` 应复用，不要在 `body` 中创建

### `@Observable` 过细粒度更新导致级联刷新
- 一个 `@Observable` 对象的任意属性变化都会触发访问了该属性的视图更新
- 如果多个视图共享一个大对象，修改任一属性会导致所有相关视图刷新
- **优化：** 拆分为多个小的 `@Observable` 对象，或使用 `@ObservationIgnored` 标记无关属性

---

## 导航问题排查

### `NavigationStack` 路径管理错误
- 确认 `NavigationStack(path:)` 绑定的数组类型与 `navigationDestination(for:)` 注册类型匹配
- 路径数组中的值类型必须是 `Hashable`
- 编程式导航时注意路径操作在主线程执行

### `NavigationStack` 常见陷阱
- **Deep Link 构建路径后视图不对** — 确认路径数组中的值顺序正确，NavigationStack 按数组顺序依次 push
- **类型注册冲突** — 同一 NavigationStack 内为同一类型注册了多个 `navigationDestination`，只有最近的一个生效
- **子视图中注册的 destination 被忽略** — `navigationDestination` 必须注册在 NavigationStack 的直接子视图层级中
- **pop 到根视图** — 将 path 数组清空：`path = NavigationPath()` 或 `path.removeLast(path.count)`
- **调试技巧** — 在 `onChange(of: path)` 中打印路径内容，确认推入的值类型和顺序

### Sheet 重复弹出 / 无法关闭
- 检查 `isPresented` 绑定的 `@State` 是否被多处修改
- 检查 `sheet(item:)` 的 item 是否在 dismiss 后正确置 nil
- 多个 sheet 不能挂在同一个视图上 — 使用 `sheet(item:)` 配合 enum 管理

### `navigationDestination` 注册冲突
- 同一 `NavigationStack` 内不要为相同类型注册多个 `navigationDestination`
- 子视图中注册的 `navigationDestination` 也会被父 NavigationStack 捕获
- 调试时打印 path 数组内容，确认推入的值类型正确

---

## 动画问题排查

### `.animation(value:)` 缺少 value 参数
- iOS 15+ 废弃了无参数的 `.animation()`，必须指定 `value`
- 不指定 value 会导致所有状态变化都触发动画（非预期行为）

### `withAnimation` vs implicit animation 冲突
- `withAnimation` 是显式动画，作用于闭包内的状态变化
- `.animation()` 是隐式动画，作用于上游修饰符的变化
- 两者同时存在时可能产生双重动画或动画覆盖
- **建议：** 在同一视图层级中统一使用一种方式

### `matchedGeometryEffect` 闪烁
- 确认两个视图不会同时出现在屏幕上
- 确认 `id` 和 `namespace` 完全匹配
- 使用 `if/else` 切换视图，而非 `opacity` 隐藏
- 检查是否有其他动画干扰过渡效果
