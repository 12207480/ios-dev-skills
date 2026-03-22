# 修复模式与测试

## 修复前确认清单

开始改代码前再确认一次：
- [ ] 根因已明确，能用一句话解释
- [ ] 已理解原有代码的设计意图
- [ ] 已梳理影响范围（上游调用方 + 下游依赖）
- [ ] 已确认修复方案不会破坏原有设计意图
- [ ] 已从最新主分支创建修复分支

## 修复原则

### 1. 最小修改原则
- 只改与 Bug 直接相关的代码
- 不顺手重构，不顺手优化，不顺手加功能
- 修复和重构分开提交

### 2. 在根因处修复
- 数据源错了 → 修数据源，不要在展示层做特殊处理
- 逻辑分支遗漏 → 补全分支，不要在下游兜底
- 时序不对 → 修正时序，不要加 delay

### 3. 保持原有代码风格
- 遵循项目现有的命名、缩进、架构模式
- 不要在修 Bug 时引入新的编码风格

### 4. Swift 安全修复模式

| 问题类型 | 修复模式 |
|---------|---------|
| 强制解包 Bug | 替换为 `guard let` / `if let` / `??` 默认值 |
| 数组越界 Bug | 添加 bounds 检查 |
| 类型转换 Bug | 使用 `as?` 安全转换 |
| 线程安全 Bug | 使用 `@MainActor` / `actor` / 串行队列 |
| 循环引用 Bug | 添加 `[weak self]` |

### 5. Swift 6 并发修复模式

| 问题类型 | 修复模式 |
|---------|---------|
| 非主线程更新 UI | 给 ViewModel/方法添加 `@MainActor` 标注 |
| Sendable 编译报错 | 使值类型遵循 `Sendable`，或使用 `actor` 封装引用类型 |
| Actor reentrancy 数据不一致 | `await` 后重新验证状态，不要假设 await 前后状态不变 |
| Task 未取消导致页面销毁后回调 | 存储 `Task` 引用，`deinit`/`onDisappear` 中调用 `cancel()` |
| 全局 var 并发警告 | 标注 `@MainActor` 或改为 `actor` 封装 |
| `nonisolated(unsafe)` 绕过检查 | 优先修正设计（改为 `let` / `actor` / `@MainActor`），绕过是最后手段 |

```swift
// 常见修复：Task 取消
class MyViewController: UIViewController {
    private var loadTask: Task<Void, Never>?

    func startLoading() {
        loadTask = Task {
            let data = try? await service.fetch()
            guard !Task.isCancelled else { return }  // 检查取消
            updateUI(with: data)
        }
    }

    deinit { loadTask?.cancel() }
}
```

### 6. SwiftUI 常见修复原则

> SwiftUI 问题的详细排查与修复见 `swiftui-debugging.md`，此处仅列修复原则。

- **状态所有权**：外部传入值用 `let` / `@Binding`，自身管理状态用 `@State`，绝不用 `@State` 接收外部值
- **Identity 稳定性**：`ForEach` / `List` 使用稳定的业务 ID，不使用数组索引
- **动画精确绑定**：使用 `.animation(.spring, value: specificState)` 指定触发值，避免无参数的 `.animation()`
- **导航类型匹配**：`NavigationStack(path:)` 的路径类型与 `navigationDestination(for:)` 注册类型严格一致
- **Task 生命周期**：`.task {}` 自动管理取消，但内部长操作需检查 `Task.isCancelled`
- **观察机制正确性**：`@Observable` 对象的属性在 `body` 中直接访问，不通过方法间接访问

---

## 编写测试

### 先红后绿流程

1. **先写一个会失败的测试（证明 Bug 存在）**
   - 用 Bug 的输入条件作为测试输入
   - 用预期行为作为断言条件
   - 运行 → 测试应该失败（红色）

2. **实现修复代码**

3. **再运行测试** → 测试应该通过（绿色）

4. **运行完整测试套件** → 全部通过（无回归）

### 测试类型选择表

| Bug 类型 | 推荐测试类型 | 工具 |
|---------|------------|------|
| 逻辑 Bug | 单元测试 | Swift Testing `@Test` / XCTest |
| 数据 Bug | Mock 数据的单元测试 | Swift Testing + Mock |
| UI Bug | 快照测试 | swift-snapshot-testing |
| 交互 Bug | UI 自动化测试 | XCUITest |
| 性能 Bug | 性能基准测试 | `measure {}` |

### Swift Testing 写法（Xcode 16+）

```swift
import Testing

@Test("loadData 网络失败时状态变为 error")
func loadData_networkFailure() async {
    let service = MockService(shouldFail: true)
    let viewModel = MyViewModel(service: service)
    await viewModel.loadData()
    #expect(viewModel.state == .error)
}

// 参数化测试：一次覆盖多个输入
@Test("价格格式化", arguments: [
    (100, "¥100.00"),
    (0, "¥0.00"),
    (9999, "¥9,999.00"),
])
func priceFormat(input: Int, expected: String) {
    #expect(formatPrice(input) == expected)
}
```
