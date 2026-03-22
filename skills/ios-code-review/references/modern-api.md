# 现代 API 迁移对照表

> 审查时发现使用废弃 API，应建议迁移到现代替代方案。标注为 🟡 Should Fix（不阻塞合并，但应计划迁移）。

## 导航

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| `NavigationView` | `NavigationStack`（单栏）/ `NavigationSplitView`（多栏） | iOS 16 |
| `NavigationLink(destination:isActive:)` | `NavigationLink(value:)` + `.navigationDestination` | iOS 16 |
| `.navigationBarTitle()` | `.navigationTitle()` | iOS 14 |
| `.navigationBarHidden()` | `.toolbar(.hidden, for: .navigationBar)` | iOS 16 |

## 数据流与状态管理

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| `ObservableObject` + `@Published` | `@Observable` 宏 | iOS 17 |
| `@StateObject` | `@State`（配合 `@Observable`） | iOS 17 |
| `@ObservedObject` | 直接使用 / `@Bindable`（需要绑定时） | iOS 17 |
| `@EnvironmentObject` | `@Environment`（配合 `@Observable`） | iOS 17 |

## 视图修饰符

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| `foregroundColor()` | `foregroundStyle()` | iOS 15 |
| `background(color)` | `background { }` 或 `background(in:)` | iOS 15 |
| `.animation(.easeInOut)` | `.animation(.easeInOut, value: someValue)` | iOS 15 |
| `onChange(of:perform:)` | `onChange(of:) { oldValue, newValue in }` | iOS 17 |
| `onAppear { Task {} }` | `task { }` | iOS 15 |

## UIHostingController

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| 手动计算 `sizeThatFits` + `intrinsicContentSize` | `sizingOptions: .intrinsicContentSize` | iOS 16 |

## Alert 与确认对话框

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| `Alert(title:message:)` 结构体 | `.alert(_:isPresented:actions:message:)` | iOS 15 |
| `ActionSheet` | `.confirmationDialog(_:isPresented:actions:)` | iOS 15 |
| `confirmationDialog` 的 `titleVisibility` | iOS 16+ 默认显示标题，iOS 15 需要 `.visible` | iOS 15/16 |

## 其他

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| `UIScreen.main.bounds` | `GeometryReader` / `WindowGroup` 的 `defaultSize` | iOS 16 |
| `UIApplication.shared.keyWindow` | `WindowScene` 获取 window | iOS 15 |
| `openURL(_:)` 同步方法 | `environment(\.openURL)` / `OpenURLAction` | iOS 15 |
| `@UIApplicationDelegateAdaptor` 处理生命周期 | `ScenePhase` environment | iOS 14 |

## 并发与异步

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| `DispatchQueue.main.async` 更新 UI | `@MainActor` | iOS 13 |
| GCD `DispatchQueue` 并发任务 | `TaskGroup` / `async let` | iOS 13 |
| `OperationQueue` 简单异步 | `Task` / `async/await` | iOS 13 |
| Completion handler 回调 | `async throws` 函数 | iOS 13 |

## 数据持久化

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| 手动 Core Data stack 配置 | `NSPersistentContainer` / SwiftData | iOS 10 / iOS 17 |
| `NSUserActivity` 手动编解码 | `Transferable` 协议 | iOS 16 |

## 测试

| 废弃 API | 现代替代 | 最低版本 |
|---------|---------|---------|
| `XCTestCase` + `func test_xxx()` | `@Suite` + `@Test` (Swift Testing) | Xcode 16 |
| `XCTAssertEqual` / `XCTAssertTrue` | `#expect()` | Xcode 16 |
| 手动参数化测试循环 | `@Test(arguments:)` | Xcode 16 |

## 审查建议

```
看到废弃 API 时的判断流程：

1. 项目最低支持版本是否 >= 现代 API 的最低版本？
   → 是 → 🟡 Should Fix，建议迁移
   → 否 → 📝 Info，提醒未来可迁移

2. 是新增代码还是修改已有代码？
   → 新增 → 🟡 Should Fix，新代码应使用现代 API
   → 修改 → 🟢 Nit，可在后续重构中统一迁移

3. 迁移是否涉及大范围改动？
   → 是 → 建议单独 PR 迁移，不在本次 PR 中混入
   → 否 → 可在本次 PR 中一并修改
```

### 版本兼容性审查策略

#### `#available` 检查规范

```
- [ ] #available 检查是否覆盖了所有新 API 调用点？
      → 不要只检查入口方法，内部调用链也需要确认
- [ ] else 分支（fallback）是否提供了等价功能？
      → 不能只是空实现或 fatalError
      → fallback 功能可以简化，但核心体验必须保留
- [ ] @available 标注是否用于整个类型/方法？
      → 当整个 View/VC 仅支持特定版本时，使用 @available 而非逐行 #available
      → @available(iOS 17, *) struct ModernView: View { ... }
```

#### 多版本支持策略表

| 场景 | 策略 | 示例 |
|------|------|------|
| 新 API 仅影响样式/动画 | 有则用，无则忽略 | `.symbolEffect()` 在 iOS 17+ 生效，低版本无动效但不影响功能 |
| 新 API 提供核心功能 | 必须提供 fallback | `NavigationStack` → 低版本使用 `NavigationView` |
| 新 API 替代废弃 API | 条件编译 + 双实现 | `onChange(of:)` 新旧两种签名 |
| 新框架（SwiftData/WidgetKit） | 独立模块 + canImport | `#if canImport(SwiftData)` |
| 系统权限新增项 | 版本检查 + 优雅降级 | iOS 17+ 需要新权限，低版本跳过相关功能 |

#### 渐进式迁移原则

```
1. 新代码优先使用最新 API（配合 #available 检查）
2. 旧代码保持现状，不在功能 PR 中混入大规模迁移
3. 当最低支持版本提升时，批量清理过时的 #available 检查
4. 保持 fallback 代码的可维护性 — 当 fallback 过于复杂时，考虑是否值得支持旧版本
5. 使用 TODO 标记未来可清理的兼容代码：// TODO: Remove when min deployment target >= iOS 17
```
