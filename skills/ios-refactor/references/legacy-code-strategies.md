# 遗留代码处理策略

## 遗留代码定义与识别

### 什么是遗留代码

> 遗留代码 = 没有测试覆盖的代码。 — Michael Feathers, Working Effectively with Legacy Code

不仅仅是"旧代码"。即使是上周写的代码，如果没有测试、难以理解、难以修改，就是遗留代码。

### 识别信号

```
代码层面：
├── 方法超过 100 行
├── 类超过 500 行
├── 深层嵌套（> 4 层 if/for）
├── 大量注释但仍然难以理解
├── 全局状态和单例泛滥
├── 复制粘贴的重复代码
├── 已弃用的 API 仍在使用
└── 无法为其编写单元测试

项目层面：
├── "别碰那块代码"的共识
├── 修改一处产生意外的连锁反应
├── 新人需要数周才能理解某个模块
├── Bug 修复后频繁引入新 Bug
└── 编译时间异常长的模块
```

---

## "安全区域"划定

### 风险分级

| 区域 | 特征 | 改动策略 |
|------|------|---------|
| **安全区（绿色）** | 有测试覆盖、职责清晰、依赖简单 | 可以直接重构 |
| **谨慎区（黄色）** | 部分测试、逻辑较复杂、有一些依赖 | 先补测试再重构 |
| **危险区（红色）** | 无测试、逻辑不清、被大量依赖 | 先隔离、建立安全网、小步修改 |
| **禁区（黑色）** | 无人理解、涉及核心基础设施、修改影响全局 | 不动，除非有充分的理由和完整的方案 |

### 如何划定安全边界

```
1. 运行测试覆盖率报告 → 找出无覆盖区域
2. 查看 Git blame → 找出长期无人维护的文件
3. 查看 Bug 追踪系统 → 找出高 Bug 密度区域
4. 与团队沟通 → 了解哪些区域"大家都怕改"
5. 画出依赖图 → 找出被大量依赖的核心代码
```

---

## 为遗留代码补充测试的策略

### 特征测试（Characterization Test）

**目的：** 不是验证"应该怎样"，而是记录"现在怎样"。

```swift
// 对一个复杂的遗留方法编写特征测试
// 我们不知道它"应该"返回什么，但我们记录它"实际"返回什么
func testCalculatePrice_characterization() {
    let calculator = LegacyPriceCalculator()

    // 记录当前行为（即使这个结果可能不"正确"）
    let result1 = calculator.calculate(quantity: 1, unitPrice: 100)
    XCTAssertEqual(result1, 100)  // 记录现状

    let result2 = calculator.calculate(quantity: 5, unitPrice: 100)
    XCTAssertEqual(result2, 475)  // 原来有 5% 折扣？记录下来

    let result3 = calculator.calculate(quantity: 0, unitPrice: 100)
    XCTAssertEqual(result3, 0)    // 边界情况

    let result4 = calculator.calculate(quantity: -1, unitPrice: 100)
    XCTAssertEqual(result4, -100) // 负数情况，可能是 Bug，但先记录
}
```

**编写步骤：**
1. 调用遗留代码，传入各种输入
2. 先不写断言，运行一次看实际输出
3. 将实际输出作为期望值写入断言
4. 覆盖正常路径、边界值、异常输入
5. 这些测试就是重构时的"安全网"

### 金丝雀测试（关键路径覆盖）

**目的：** 对最关键的业务路径提供最基本的保护。

```swift
// 不需要覆盖所有分支，只覆盖最关键的路径
func testOrderFlow_happyPath() {
    // 准备
    let order = Order(items: [testItem], user: testUser)

    // 执行完整的订单流程
    let result = orderProcessor.process(order)

    // 验证关键结果
    XCTAssertTrue(result.isSuccess)
    XCTAssertNotNil(result.orderId)
    XCTAssertEqual(result.status, .confirmed)
}

func testOrderFlow_emptyCart() {
    let order = Order(items: [], user: testUser)
    let result = orderProcessor.process(order)
    XCTAssertTrue(result.isFailure)
}
```

### 集成测试作为安全网

```swift
// 当单元测试太难写时（依赖太多），写集成测试
func testUserLoginFlow_integration() async throws {
    // 使用真实的（或沙盒环境的）服务
    let loginService = LoginService(environment: .staging)
    let result = try await loginService.login(
        email: "test@example.com",
        password: "testpassword"
    )

    XCTAssertNotNil(result.token)
    XCTAssertTrue(result.token.count > 0)
}
```

---

## 解耦技巧

### 提取接口（Extract Interface）

**场景：** 遗留代码直接依赖具体类型，无法替换为测试替身。

> 详细的 Protocol 提取步骤和代码示例见 `safe-refactoring-techniques.md` 的"提取协议"部分。

核心思路：
1. 识别消费方实际使用的方法子集
2. 定义 Protocol 包含这些方法
3. 让原类型遵循 Protocol
4. 消费方的类型声明从具体类型改为 Protocol
5. 编译 → 测试

### 依赖注入（Dependency Injection）

```swift
// 渐进式引入 DI：先提供默认值，不破坏现有调用方
class ReportGenerator {
    private let dataFetcher: DataFetching
    private let formatter: DataFormatting

    // 保持现有调用方兼容
    convenience init() {
        self.init(
            dataFetcher: DatabaseManager.shared,
            formatter: ExcelExporter.shared
        )
    }

    // 新的可测试的初始化方法
    init(dataFetcher: DataFetching, formatter: DataFormatting) {
        self.dataFetcher = dataFetcher
        self.formatter = formatter
    }
}
```

### Seam 技术（接缝技术）

**Seam = 可以在不修改代码的情况下改变行为的地方。**

```swift
// === 对象 Seam：通过子类覆盖改变行为 ===
class LegacyOrderProcessor {
    func processOrder(_ order: Order) {
        // ... 大量遗留逻辑 ...
        sendNotification(order)  // 我们想在测试中避免真实发送
        // ... 更多逻辑 ...
    }

    // 这是一个 Seam：子类可以覆盖此方法
    func sendNotification(_ order: Order) {
        NotificationService.shared.send(...)
    }
}

// 测试中使用子类覆盖
class TestableOrderProcessor: LegacyOrderProcessor {
    var notificationSent = false
    override func sendNotification(_ order: Order) {
        notificationSent = true  // 不真实发送，只记录
    }
}
```

---

## 渐进式现代化路线图模板

```
## 模块现代化路线图：[模块名]

### 现状评估
- 代码行数：[N 行]
- 测试覆盖率：[X%]
- 技术栈：[ObjC/Swift/混编]
- 架构模式：[MVC/无明确架构]
- 主要痛点：[列表]

### 目标状态
- 技术栈：[Swift]
- 架构模式：[MVVM]
- 测试覆盖率：[> 80%]
- 模块化：[独立 SPM Package]

### 分阶段计划

#### 阶段 1：建立安全网（1-2 周）
- [ ] 为关键路径编写特征测试
- [ ] 为公共接口编写集成测试
- [ ] 记录当前行为基线

#### 阶段 2：解耦依赖（1-2 周）
- [ ] 提取核心协议（Protocol）
- [ ] 引入依赖注入
- [ ] 消除全局单例的直接使用

#### 阶段 3：逐步迁移（2-4 周）
- [ ] 模型层迁移（ObjC → Swift）
- [ ] 服务层迁移
- [ ] ViewModel 提取
- [ ] 视图层迁移（按需）

#### 阶段 4：清理验证（1 周）
- [ ] 删除旧代码
- [ ] 补充单元测试
- [ ] 性能基准测试
- [ ] 文档更新

### 风险与缓解
| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| [风险描述] | [影响范围] | [应对方案] |
```

---

## 与团队协作：重构优先级沟通

### 向管理层说明重构价值

```
不说：
  "这段代码太烂了，需要重构。"

要说：
  "过去 3 个月，这个模块产生了 15 个 Bug，
   每个 Bug 平均需要 2 天修复。
   重构后预计 Bug 率降低 70%，
   每月节省约 3 天修复时间。"
```

### 重构 PR 的沟通规范

```
PR 描述模板：

## 重构目标
[为什么要重构？解决什么问题？]

## 重构策略
[使用了什么方法？（提取方法 / 引入协议 / 分层等）]

## 行为变更声明
本 PR 不包含任何行为变更。所有外部可观察行为与重构前完全一致。

## 验证方式
- [x] 原有测试全部通过
- [x] 新增 N 个测试覆盖重构区域
- [x] 手动验证关键路径
```

---

## 总结 Checklist

```
遗留代码重构准备：
- [ ] 识别并分级了代码区域（安全/谨慎/危险/禁区）
- [ ] 为关键路径编写了特征测试
- [ ] 为核心行为编写了金丝雀测试
- [ ] 记录了当前行为基线（输入/输出/副作用）

解耦执行：
- [ ] 提取了必要的 Protocol 接口
- [ ] 引入了依赖注入（渐进式，保持向后兼容）
- [ ] 利用了 Seam 技术使代码可测试
- [ ] 每步编译通过 + 测试通过

团队协作：
- [ ] 重构计划已与团队沟通
- [ ] 重构 PR 描述清晰（动机 + 策略 + 行为声明 + 验证）
- [ ] 未在重构 PR 中混入功能变更
- [ ] 渐进式路线图已记录并跟踪
```
