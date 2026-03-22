# 架构重构

## 架构模式对比决策表

| 维度 | MVC | MVVM | TCA | Clean Architecture |
|------|-----|------|-----|-------------------|
| **复杂度** | 低 | 中 | 高 | 高 |
| **可测试性** | 差（逻辑在 VC） | 好（ViewModel 可测试） | 优秀（纯函数 Reducer） | 优秀（分层隔离） |
| **学习成本** | 低 | 低-中 | 高 | 中-高 |
| **适用规模** | 小型项目 | 中大型项目 | 复杂状态管理 | 大型项目/团队 |
| **状态管理** | 散乱 | @Observable/Combine | 集中式 Store | 领域模型 |
| **依赖管理** | 无约束 | 协议注入 | Environment/Dependency | 分层依赖 |
| **团队协作** | 容易冲突 | 职责清晰 | 统一模式 | 模块独立 |

### 何时考虑架构迁移

```
信号：
├── ViewController > 500 行 → 考虑提取 ViewModel
├── 业务逻辑散落在 View 层 → 需要逻辑层分离
├── 测试覆盖率无法提升 → 需要可测试的架构
├── 团队协作频繁冲突 → 需要更好的模块边界
├── 状态同步 Bug 频发 → 需要集中式状态管理
└── 模块间循环依赖 → 需要依赖方向管理
```

---

## MVC → MVVM 迁移步骤

### 第一步：识别 ViewController 中的逻辑

```swift
// === 迁移前：典型的 Massive ViewController ===
class OrderListViewController: UIViewController {
    // UI 属性
    @IBOutlet weak var tableView: UITableView!
    @IBOutlet weak var loadingView: UIActivityIndicatorView!

    // 数据（不应在 VC 中）
    var orders: [Order] = []
    var isLoading = false
    var errorMessage: String?

    // 业务逻辑（不应在 VC 中）
    func loadOrders() {
        isLoading = true
        updateUI()
        APIService.shared.fetchOrders { [weak self] result in
            self?.isLoading = false
            switch result {
            case .success(let orders):
                self?.orders = orders
            case .failure(let error):
                self?.errorMessage = error.localizedDescription
            }
            self?.updateUI()
        }
    }

    func deleteOrder(_ order: Order) {
        // 删除逻辑...
    }

    // UI 更新逻辑
    func updateUI() {
        loadingView.isHidden = !isLoading
        tableView.reloadData()
        // ...
    }
}
```

### 第二步：提取 ViewModel

```swift
// === 迁移后：ViewModel 持有状态和逻辑 ===
// 使用 ObservableObject + @Published 以兼容 UIKit Combine 绑定和 SwiftUI
// 若项目最低支持 iOS 17 且仅用 SwiftUI，可改用 @Observable
class OrderListViewModel: ObservableObject {
    private let orderService: OrderServiceProtocol

    // 状态（@Published 支持 Combine 订阅）
    @Published var orders: [Order] = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    init(orderService: OrderServiceProtocol) {
        self.orderService = orderService
    }

    // 业务逻辑
    func loadOrders() async {
        isLoading = true
        defer { isLoading = false }

        do {
            orders = try await orderService.fetchOrders()
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func deleteOrder(_ order: Order) async {
        do {
            try await orderService.deleteOrder(order.id)
            orders.removeAll { $0.id == order.id }
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

### 第三步：简化 ViewController

```swift
// === 迁移后：VC 只负责 UI 绑定 ===
class OrderListViewController: UIViewController {
    @IBOutlet weak var tableView: UITableView!
    @IBOutlet weak var loadingView: UIActivityIndicatorView!

    private let viewModel: OrderListViewModel
    private var cancellables = Set<AnyCancellable>()

    init(viewModel: OrderListViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        bindViewModel()
        Task { await viewModel.loadOrders() }
    }

    private func bindViewModel() {
        // 方案一：Combine 绑定（推荐用于 UIKit，iOS 13+）
        viewModel.$orders
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)

        viewModel.$isLoading
            .receive(on: DispatchQueue.main)
            .sink { [weak self] isLoading in
                self?.loadingView.isHidden = !isLoading
            }
            .store(in: &cancellables)

        viewModel.$errorMessage
            .compactMap { $0 }
            .receive(on: DispatchQueue.main)
            .sink { [weak self] message in
                self?.showErrorAlert(message: message)
            }
            .store(in: &cancellables)
    }

    private func showErrorAlert(message: String) {
        let alert = UIAlertController(
            title: "错误", message: message, preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "确定", style: .default))
        present(alert, animated: true)
    }
}
```

### 迁移安全步骤

```
1. 在 VC 文件中标记哪些是"状态"、哪些是"逻辑"、哪些是"UI"
2. 创建 ViewModel 文件
3. 将"状态"属性移到 ViewModel（先不删除 VC 中的）
4. 将"逻辑"方法移到 ViewModel
5. 在 VC 中通过 ViewModel 访问状态和逻辑
6. 编译 → 测试
7. 删除 VC 中多余的状态属性
8. 编译 → 测试
9. 为 ViewModel 编写单元测试
```

---

## 绑定方式选择

| 绑定方式 | 适用场景 | iOS 最低版本 |
|---------|---------|-------------|
| **@Observable** | SwiftUI + iOS 17+ | iOS 17 |
| **Combine (@Published)** | UIKit / SwiftUI + iOS 13+ | iOS 13 |
| **Closure 回调** | 简单场景，无框架依赖 | 任意 |
| **Delegate** | 保持与现有代码一致 | 任意 |

```swift
// Combine 绑定示例（UIKit）
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false
}

// ViewController 中
viewModel.$items
    .receive(on: DispatchQueue.main)
    .sink { [weak self] _ in
        self?.tableView.reloadData()
    }
    .store(in: &cancellables)

// Closure 绑定示例
class ViewModel {
    var onItemsUpdated: (([Item]) -> Void)?
    var items: [Item] = [] {
        didSet { onItemsUpdated?(items) }
    }
}
```

---

## 依赖注入引入

### 从直接依赖到协议依赖

```swift
// === Before：直接依赖具体类型 ===
class OrderViewModel {
    func loadOrders() {
        APIService.shared.fetchOrders { ... }  // 单例直接调用
    }
}

// === After：通过协议注入 ===
protocol OrderServiceProtocol {
    func fetchOrders() async throws -> [Order]
    func deleteOrder(_ id: String) async throws
}

class OrderService: OrderServiceProtocol {
    func fetchOrders() async throws -> [Order] { ... }
    func deleteOrder(_ id: String) async throws { ... }
}

class OrderViewModel {
    private let orderService: OrderServiceProtocol

    init(orderService: OrderServiceProtocol) {
        self.orderService = orderService
    }
}

// 测试中可以注入 Mock
class MockOrderService: OrderServiceProtocol {
    var mockOrders: [Order] = []
    func fetchOrders() async throws -> [Order] { mockOrders }
    func deleteOrder(_ id: String) async throws {}
}
```

---

## 模块化拆分策略

### 识别模块边界

```
划分原则：
├── 按业务领域划分（用户、订单、支付、消息）
├── 按技术层划分（网络层、持久化层、UI 组件库）
├── 按变化频率划分（稳定基础设施 vs 频繁变化的业务）
└── 核心标准：模块间的依赖应该是单向的

识别方法：
1. 画出当前代码的依赖图
2. 找出依赖环（循环依赖）
3. 识别高内聚的代码组（经常一起修改的文件）
4. 确定模块边界（接口最少化原则）
```

### 提取 Protocol 接口

```swift
// 模块间通过 Protocol 通信
// --- ModuleA (不依赖 ModuleB) ---
protocol PaymentProcessing {
    func processPayment(amount: Decimal) async throws -> PaymentResult
}

class OrderService {
    private let paymentProcessor: PaymentProcessing

    init(paymentProcessor: PaymentProcessing) {
        self.paymentProcessor = paymentProcessor
    }
}

// --- ModuleB (实现 Protocol) ---
class StripePaymentProcessor: PaymentProcessing {
    func processPayment(amount: Decimal) async throws -> PaymentResult {
        // Stripe SDK 调用
    }
}
```

### 创建 SPM Package

```
项目结构：
├── App/                          # 主工程（组装层）
├── Packages/
│   ├── CoreKit/                  # 基础工具（Extension、常量、通用协议）
│   ├── NetworkKit/               # 网络层
│   ├── UserModule/               # 用户模块
│   │   ├── Sources/
│   │   │   ├── UserModule/       # 公共接口
│   │   │   └── UserModuleImpl/   # 内部实现
│   │   └── Tests/
│   ├── OrderModule/              # 订单模块
│   └── UIComponents/             # 共享 UI 组件
└── Package.swift
```

### Package.swift 完整示例

```swift
// swift-tools-version: 5.9

import PackageDescription

let package = Package(
    name: "AppModules",
    platforms: [.iOS(.v16)],
    products: [
        // 公共接口模块（Protocol 定义、领域模型）
        .library(name: "CoreKit", targets: ["CoreKit"]),
        .library(name: "NetworkKit", targets: ["NetworkKit"]),
        // 业务模块
        .library(name: "UserModule", targets: ["UserModule"]),
        .library(name: "OrderModule", targets: ["OrderModule"]),
        // UI 组件库
        .library(name: "UIComponents", targets: ["UIComponents"]),
    ],
    dependencies: [
        // 第三方依赖统一在此管理
    ],
    targets: [
        // === 基础层 ===
        .target(
            name: "CoreKit",
            dependencies: [],
            path: "Sources/CoreKit"
        ),

        // === 网络层 ===
        .target(
            name: "NetworkKit",
            dependencies: ["CoreKit"],
            path: "Sources/NetworkKit"
        ),

        // === 业务模块：用户 ===
        .target(
            name: "UserModule",
            dependencies: ["CoreKit", "NetworkKit"],
            path: "Sources/UserModule"
        ),
        .testTarget(
            name: "UserModuleTests",
            dependencies: ["UserModule"],
            path: "Tests/UserModuleTests"
        ),

        // === 业务模块：订单 ===
        .target(
            name: "OrderModule",
            dependencies: ["CoreKit", "NetworkKit", "UserModule"],
            path: "Sources/OrderModule"
        ),
        .testTarget(
            name: "OrderModuleTests",
            dependencies: ["OrderModule"],
            path: "Tests/OrderModuleTests"
        ),

        // === UI 组件 ===
        .target(
            name: "UIComponents",
            dependencies: ["CoreKit"],
            path: "Sources/UIComponents"
        ),
    ]
)
```

### 依赖方向管理

```
正确的依赖方向（单向依赖）：

App → Feature Modules → Domain → Core
  │         │              │       │
  │         │              │       └── 基础类型、扩展、工具
  │         │              └── 业务模型、Protocol 定义
  │         └── 具体功能实现（UI + ViewModel + Service 实现）
  └── 组装、依赖注入、App 生命周期

禁止：
  ✗ Core → Domain（底层不应依赖上层）
  ✗ ModuleA → ModuleB → ModuleA（循环依赖）
  ✗ Domain → Feature（领域层不应依赖实现）

解决循环依赖的方法：
  1. 依赖反转（Dependency Inversion）
     → 提取 Protocol 到更底层的模块
  2. 事件总线（Notification/Combine）
     → 通过事件解耦，不直接引用
  3. 中介者模式（Mediator）
     → 在上层模块中协调
```

---

## CoreData → SwiftData 迁移

### 对应关系

| Core Data | SwiftData | 说明 |
|-----------|-----------|------|
| `NSManagedObject` | `@Model class` | SwiftData 使用宏自动生成模型元数据 |
| `NSPersistentContainer` | `ModelContainer` | 配置存储位置、模型架构 |
| `NSManagedObjectContext` | `ModelContext` | 操作数据的上下文 |
| `NSFetchRequest` | `#Predicate` + `FetchDescriptor` | 类型安全的查询 |
| `@FetchRequest` (SwiftUI) | `@Query` | SwiftUI 中的数据绑定 |
| `NSEntityDescription` | 自动从 `@Model` 推断 | 无需 xcdatamodeld 文件 |
| `NSMigrationManager` | `VersionedSchema` + `SchemaMigrationPlan` | 声明式迁移 |

### 迁移示例

```swift
// === Core Data ===
// User.xcdatamodeld + NSManagedObject 子类
class CDUser: NSManagedObject {
    @NSManaged var name: String
    @NSManaged var email: String
    @NSManaged var createdAt: Date
    @NSManaged var orders: NSSet?
}

// 查询
let request: NSFetchRequest<CDUser> = CDUser.fetchRequest()
request.predicate = NSPredicate(format: "name CONTAINS[cd] %@", keyword)
request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
let users = try context.fetch(request)

// === SwiftData ===
@Model
class User {
    var name: String
    var email: String
    var createdAt: Date
    @Relationship(deleteRule: .cascade)
    var orders: [Order] = []

    init(name: String, email: String) {
        self.name = name
        self.email = email
        self.createdAt = Date()
    }
}

// 查询（类型安全）
let predicate = #Predicate<User> { $0.name.localizedStandardContains(keyword) }
let descriptor = FetchDescriptor<User>(
    predicate: predicate,
    sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
)
let users = try modelContext.fetch(descriptor)
```

### 共存策略

```
推荐的迁移路径：
├── 阶段一：新模型使用 SwiftData，旧模型保留 Core Data
│   └── 两套持久化栈可以共存于同一个 App 中
├── 阶段二：逐步将旧模型迁移到 SwiftData
│   ├── 使用 Core Data 读取旧数据
│   ├── 转换为 SwiftData @Model 对象写入
│   └── 验证数据完整性
├── 阶段三：删除 Core Data 栈和 xcdatamodeld 文件
└── 注意事项：
    ├── SwiftData 底层仍使用 Core Data（SQLite）
    ├── 可以让 SwiftData 的 ModelContainer 读取已有的 Core Data SQLite 文件
    ├── 字段名和关系名必须与原 Core Data 模型匹配
    └── 最低支持 iOS 17
```

---

## 技术债评估与优先级排序

### 评估矩阵

| 维度 | 权重 | 评估标准 |
|------|------|---------|
| **修改频率** | 高 | 过去 3 个月该文件/模块被修改了多少次？ |
| **Bug 密度** | 高 | 该区域产生了多少 Bug？ |
| **理解难度** | 中 | 新人上手需要多长时间理解？ |
| **测试覆盖** | 中 | 测试覆盖率如何？ |
| **耦合程度** | 中 | 与其他模块的耦合是否过紧？ |

### 优先级排序公式

```
重构优先级 = 修改频率 × Bug 密度 × (1 - 测试覆盖率)

高优先级：经常改、经常出 Bug、测试少 → 立即重构
中优先级：偶尔改、偶尔出问题 → 计划性重构
低优先级：稳定运行、很少修改 → 暂不重构
```

---

## 架构重构 Checklist

```
重构前：
- [ ] 当前架构痛点已明确记录
- [ ] 目标架构已确定（团队共识）
- [ ] 迁移路径已规划（分阶段）
- [ ] 现有测试覆盖充分
- [ ] 团队已了解目标架构模式

迁移执行：
- [ ] 每步只做一个架构变更（提取 ViewModel / 引入 Protocol / 拆分模块）
- [ ] 每步编译通过 + 测试通过
- [ ] 未混入功能变更
- [ ] 依赖方向正确（单向、无循环）
- [ ] 新代码遵循目标架构模式

迁移后：
- [ ] 所有外部行为等价
- [ ] 代码可测试性提升（ViewModel/Service 可独立测试）
- [ ] 模块边界清晰，依赖关系明确
- [ ] 团队能理解和维护新架构
- [ ] 文档已更新（架构图、模块说明）
```
