# 安全重构技术

## 原子重构操作清单

### 重命名（Rename）

**适用场景：** 变量、方法、类型、文件名不再准确反映其职责。

**安全步骤：**
1. 全局搜索当前名称，记录所有使用位置
2. 使用 Xcode → Refactor → Rename（确保跨文件生效）
3. 编译 → 确认无报错
4. 运行测试 → 确认无失败
5. 检查字符串中的硬编码引用（如 `#selector`、`NSStringFromClass`、Storyboard ID）

```swift
// Before
class DataManager {
    func getData() -> [Item] { ... }
}

// After — 名称更准确地反映职责
class ItemRepository {
    func fetchAllItems() -> [Item] { ... }
}
```

**注意事项：**
- Xcode Rename 不会修改注释和字符串字面量中的引用
- ObjC 方法的重命名需要同时更新 `.h` 和 `.m` 文件
- 涉及 `@objc(name)` 标注的重命名需要特别注意序列化兼容性

### 提取方法（Extract Method）

**适用场景：** 方法过长，包含可独立命名的逻辑块。

**安全步骤：**
1. 识别可提取的代码块（有明确的输入和输出）
2. 确认代码块内的局部变量依赖
3. 使用 Xcode → Refactor → Extract Method（或手动提取）
4. 新方法的参数 = 代码块需要的外部变量
5. 新方法的返回值 = 代码块对外产生的结果
6. 编译 → 测试

```swift
// Before
func processOrder(_ order: Order) {
    // 验证逻辑（20行）
    guard order.items.count > 0 else { return }
    guard order.totalAmount > 0 else { return }
    guard order.shippingAddress != nil else { return }
    // ... 更多验证

    // 计算逻辑（15行）
    let subtotal = order.items.reduce(0) { $0 + $1.price }
    let tax = subtotal * 0.1
    let total = subtotal + tax
    // ... 更多计算
}

// After
func processOrder(_ order: Order) {
    guard validateOrder(order) else { return }
    let total = calculateTotal(for: order)
    // ...
}

private func validateOrder(_ order: Order) -> Bool {
    guard order.items.count > 0 else { return false }
    guard order.totalAmount > 0 else { return false }
    guard order.shippingAddress != nil else { return false }
    return true
}

private func calculateTotal(for order: Order) -> Decimal {
    let subtotal = order.items.reduce(0) { $0 + $1.price }
    let tax = subtotal * 0.1
    return subtotal + tax
}
```

### 内联（Inline）

**适用场景：** 方法/变量只被使用一次，或名称不比内容更清晰。

**安全步骤：**
1. 确认只有一个调用点（或所有调用点都要内联）
2. 将方法体/变量值替换到调用位置
3. 删除原方法/变量声明
4. 编译 → 测试

```swift
// Before — 变量只是中转，没有增加可读性
let isValid = item != nil
if isValid {
    process(item!)
}

// After
if let item = item {
    process(item)
}
```

### 移动（Move）

**适用场景：** 方法或类型放错了位置，应该属于另一个类/文件/模块。

**安全步骤：**
1. 确认目标位置（该方法的数据依赖在哪里更集中）
2. 在目标位置创建新方法
3. 更新所有调用方指向新位置
4. 编译 → 测试
5. 删除原位置的方法
6. 编译 → 测试

### 提取变量（Extract Variable）

**适用场景：** 复杂表达式需要命名以提高可读性。

```swift
// Before
if user.age >= 18 && user.hasVerifiedEmail && !user.isBanned && user.subscriptionEndDate > Date() {
    grantAccess()
}

// After
let isEligibleForAccess = user.age >= 18
    && user.hasVerifiedEmail
    && !user.isBanned
    && user.subscriptionEndDate > Date()

if isEligibleForAccess {
    grantAccess()
}
```

### 提取协议（Extract Protocol）

**适用场景：** 需要解耦依赖关系，便于测试或模块化。

**安全步骤：**
1. 识别消费方实际使用的方法子集
2. 定义 Protocol，包含这些方法
3. 让原类型遵循该 Protocol
4. 将消费方的类型声明从具体类型改为 Protocol
5. 编译 → 测试

```swift
// Before — 直接依赖具体类型
class OrderViewModel {
    let networkService: NetworkService // 具体类型

    func loadOrders() {
        networkService.fetchOrders { ... }
    }
}

// After — 依赖抽象
protocol OrderFetching {
    func fetchOrders(completion: @escaping (Result<[Order], Error>) -> Void)
}

extension NetworkService: OrderFetching {}

class OrderViewModel {
    let orderFetcher: OrderFetching // 抽象类型

    func loadOrders() {
        orderFetcher.fetchOrders { ... }
    }
}
```

### 消除魔法数/字符串

```swift
// Before
cell.layer.cornerRadius = 8
label.font = .systemFont(ofSize: 14)
if status == "completed" { ... }

// After
enum DesignConstants {
    static let cardCornerRadius: CGFloat = 8
    static let bodyFontSize: CGFloat = 14
}

enum OrderStatus: String {
    case completed
    case pending
    case cancelled
}
```

---

## Xcode 重构工具

| 操作 | 快捷路径 | 注意事项 |
|------|---------|---------|
| Rename | Editor → Refactor → Rename | 跨文件重命名，但不处理字符串字面量 |
| Extract Method | Editor → Refactor → Extract Method | 自动推断参数和返回值 |
| Extract Variable | Editor → Refactor → Extract Expression | 提取为局部常量 |
| Generate Memberwise Initializer | Editor → Refactor → Generate | 自动生成构造器 |

**Xcode 重构的局限性：**
- 不支持跨 ObjC/Swift 边界的重命名
- 不支持字符串字面量中的引用更新
- 有时在复杂泛型场景下不够准确
- 建议：用 Xcode 工具做初步重构，手动验证边界情况

---

## 手动重构的安全模式

### Strangler Fig Pattern（绞杀者模式）

**适用场景：** 替换一个大型模块/系统，不能一步到位。

**步骤：**
1. 在旧系统旁边创建新实现
2. 新功能走新实现，旧功能仍走旧实现
3. 逐步将旧功能迁移到新实现
4. 旧实现不再有调用方时，删除它

```swift
// 阶段 1：新旧共存
protocol ImageLoading {
    func loadImage(url: URL) async -> UIImage?
}

class LegacyImageLoader: ImageLoading { ... } // 旧实现
class ModernImageLoader: ImageLoading { ... } // 新实现

// 阶段 2：通过配置切换
let imageLoader: ImageLoading = useModernLoader
    ? ModernImageLoader()
    : LegacyImageLoader()

// 阶段 3：全部切换后删除 LegacyImageLoader
```

### Branch by Abstraction（抽象分支模式）

**适用场景：** 替换一个被广泛依赖的组件。

**步骤：**
1. 创建抽象层（Protocol）
2. 让所有消费方依赖抽象而非具体实现
3. 在抽象层后面实现新版本
4. 切换抽象层的实现
5. 删除旧实现

---

## 重构的版本控制策略

### 小步提交原则

```
每个原子重构操作 = 一个独立的 commit

好的 commit 序列：
  refactor: 重命名 DataManager 为 ItemRepository
  refactor: 提取 validateOrder 方法
  refactor: 将价格计算逻辑移至 PriceCalculator
  refactor: 引入 OrderFetching 协议解耦网络依赖

坏的 commit：
  refactor: 重构订单模块（包含 20 个不同操作）
```

### 可回滚策略

- 每步提交后用 `git stash` 或新建分支保存当前状态
- 复杂重构建议在独立分支进行
- 使用 `git revert` 回滚单个步骤时，确保不破坏后续步骤的依赖

### 分批提交 PR 的策略

对于 L 级重构，按以下方式分批提交：

| 批次 | 内容 | 目的 |
|------|------|------|
| PR 1 | 补充测试 | 建立安全网 |
| PR 2 | 引入抽象层（Protocol/Adapter） | 解耦依赖 |
| PR 3 | 新实现 | 实际的代码替换 |
| PR 4 | 清理旧代码 | 删除不再使用的旧实现 |

---

## 总结 Checklist

```
- [ ] 重构操作可被命名（重命名 / 提取方法 / 移动 / 内联等）
- [ ] 每步操作后编译通过
- [ ] 每步操作后测试通过
- [ ] 每步操作有独立的 commit
- [ ] 使用 Xcode 工具时手动验证了边界情况
- [ ] 手动重构时遵循了安全模式（Strangler Fig / Branch by Abstraction）
- [ ] commit message 清晰描述了重构操作
- [ ] 没有在重构中混入功能变更
```
