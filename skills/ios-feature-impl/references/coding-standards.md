# 编码规范

## 编码前检查清单

开始写代码前确认：

- [ ] 需求理解卡片已完成，疑问已关闭
- [ ] 现有代码已读懂，设计意图已理解
- [ ] 关联关系已梳理，影响范围已明确
- [ ] 实现方案已确定（复杂需求需团队评审通过）
- [ ] 开发分支已从最新主分支创建
- [ ] API 已就绪或 Mock 方案已确定

## 实现顺序

### 增量需求（在现有代码上改）

1. 修改/新增数据模型 → 编写解析测试
2. 修改/新增 Service 层 → 编写 Mock 测试
3. 修改/新增 ViewModel 逻辑 → 编写状态流转测试
4. 修改/新增 UI → 预览验证各状态
5. 集成验证 → 与现有功能联合测试

### 全新功能

1. 参考项目中相似模块的目录结构和模式
2. 按 Model → Service → ViewModel → View 顺序搭建
3. 每层完成后编写测试再进入下一层
4. 最后与现有页面集成（添加入口/导航）

## 编码原则

### 1. 遵循现有代码风格
- 命名规范、缩进风格、文件组织方式与项目一致
- 不要在实现需求时引入新的编码风格
- 查看相邻代码怎么写的，保持一致

### 2. 最小改动原则
- 只改与需求直接相关的代码
- 不在需求 PR 中混入重构/优化
- 如发现需要重构，单独提 PR

### 3. 安全编码
- 可选值：`guard let` / `if let` / `??`，禁止 `!`
- 错误处理：`do-catch` 并记录日志，禁止空 `catch {}`
- 线程安全：`@MainActor` 标注 UI 操作
- 内存管理：闭包中 `[weak self]`
- 数组访问：先检查 bounds

### 4. 扩展而非修改
- 优先通过扩展（extension / 新方法）添加功能
- 修改现有方法签名前，评估所有调用方
- 修改 protocol 前，检查所有 conformance
- 修改公共组件前，确认影响范围

### 5. 版本兼容性
- 使用新 API 前检查最低部署版本，必要时使用 `if #available` 保护
- 避免在 `@available` 分支中重复大量代码，将差异提取为最小单元
- Xcode 编译警告中的 deprecation 应及时处理，不要积累
- 优先使用条件编译 `#if canImport(xxx)` 判断模块可用性
- 同一功能在不同 iOS 版本上的行为差异需文档记录

```swift
// 推荐：最小化 #available 分支的差异
func applyStyle(to label: UILabel) {
    label.font = preferredFont()
    // 仅在必要处做版本分支
    if #available(iOS 17.0, *) {
        label.textColor = .secondaryLabel
    } else {
        label.textColor = .gray
    }
}

// 避免：整个方法在两个版本分支中重复
```

## Swift 6 并发编码规范

### `@MainActor`
- ViewModel 和所有 UI 操作必须标注 `@MainActor`
- 避免在非主线程更新 UI 状态

```swift
@MainActor
@Observable
final class ProfileViewModel {
    var profile: Profile?
    var isLoading = false

    func loadProfile() async {
        isLoading = true
        defer { isLoading = false }
        profile = try? await profileService.fetch()
    }
}
```

### `@Sendable` 闭包约束
- 跨隔离域传递的闭包必须标记 `@Sendable`
- `@Sendable` 闭包中只能捕获 `Sendable` 类型

```swift
func fetchData(completion: @Sendable @escaping (Result<Data, Error>) -> Void) {
    Task {
        // ...
    }
}
```

### `actor` 隔离共享可变状态
- 多线程共享的可变状态使用 `actor` 封装
- 避免手动锁（`NSLock` / `DispatchQueue`），优先用 `actor`

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]

    func image(for url: URL) -> UIImage? {
        cache[url]
    }

    func store(_ image: UIImage, for url: URL) {
        cache[url] = image
    }
}
```

### `Task` 和 `TaskGroup`
- 使用 `Task` 启动异步工作，注意取消传播
- `TaskGroup` 用于并行执行多个独立任务
- 始终检查 `Task.isCancelled` 或使用 `Task.checkCancellation()`

```swift
func loadDashboard() async throws -> Dashboard {
    async let profile = fetchProfile()
    async let notifications = fetchNotifications()
    return Dashboard(
        profile: try await profile,
        notifications: try await notifications
    )
}
```

### 避免 `nonisolated(unsafe)`
- 除非绝对必要（如与 C/ObjC 互操作），不要使用 `nonisolated(unsafe)`
- 如果编译器报隔离错误，优先修正设计而非绕过检查

## 代码模板参考

```swift
// 数据模型：安全的 Codable 实现
struct XXXResponse: Codable {
    let id: String
    let title: String
    let subtitle: String?           // 可能不返回的字段用 Optional
    let status: Status?

    enum Status: String, Codable {
        case active, inactive
    }
}

// ViewModel：清晰的状态管理（现代 @Observable 模式）
@MainActor
@Observable
final class XXXViewModel {
    enum ViewState {
        case idle, loading
        case loaded([XXXModel])
        case empty
        case error(String)
    }

    private(set) var state: ViewState = .idle
    private let service: XXXServiceProtocol

    init(service: XXXServiceProtocol = XXXService()) {
        self.service = service
    }

    func loadData() async {
        state = .loading
        do {
            let items = try await service.fetchItems()
            state = items.isEmpty ? .empty : .loaded(items)
        } catch {
            state = .error(error.localizedDescription)
        }
    }
}
```

> **注意：** 以上为参考模板，实际编码时必须遵循项目现有的架构模式和代码风格。如果项目用 MVC，就不要自作主张引入 MVVM。如果项目用 delegate，就不要混入 Combine。
