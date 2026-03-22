# 测试质量审查清单

## 测试代码质量标准

### 可读性

**Arrange-Act-Assert 结构**

```swift
@Test("添加商品后购物车总价更新")
func addItemUpdatesTotal() {
    // Arrange — 准备测试数据和环境
    let cart = Cart()
    let item = Product(name: "iPhone", price: 999)

    // Act — 执行被测操作
    cart.add(item, quantity: 2)

    // Assert — 验证结果
    #expect(cart.totalPrice == 1998)
    #expect(cart.items.count == 1)
}
```

**描述性命名**

```swift
// 好：读测试名就知道测什么
@Test("空购物车结算时显示提示")
@Test("过期优惠券不可使用")
@Test("网络超时后自动重试一次")

// 差：不知道在测什么
@Test("test1")
@Test("购物车测试")
@Test("验证功能")
```

**一个测试一个行为**

```swift
// 错误：一个测试验证多个不相关行为
@Test("购物车功能")
func cartFeatures() {
    let cart = Cart()
    cart.add(item)
    #expect(cart.items.count == 1)    // 行为 1：添加
    cart.remove(item.id)
    #expect(cart.items.isEmpty)        // 行为 2：删除
    cart.add(item)
    cart.add(item2)
    #expect(cart.totalPrice == 1500)   // 行为 3：计算
}

// 正确：拆分为独立测试
@Test("添加商品后数量增加")
func addItem() { ... }

@Test("删除商品后列表为空")
func removeItem() { ... }

@Test("多商品总价计算正确")
func totalPrice() { ... }
```

### 独立性

| 检查项 | 通过标准 |
|-------|---------|
| 无共享可变状态 | 每个测试有独立的 setUp，不共享 `var` 属性 |
| 无执行顺序依赖 | 随机顺序运行（Xcode: Edit Scheme → Test → Randomize execution order）通过 |
| 无文件系统依赖 | 不读写真实文件，使用内存 Mock |
| 无网络依赖 | 不调用真实 API，使用 Protocol Mock |
| 无数据库残留 | 使用 in-memory 存储，tearDown 中清理 |

### 可维护性

```swift
// 错误：过度抽象，测试难以理解
@Test func test() {
    verifyBehavior(input: .type1, expected: .result1, transform: .standard)
}

// 错误：完全不抽象，大量重复代码
@Test func test1() {
    let service = MockService()
    service.stubbedResult = .success(User(id: "1", name: "A"))
    let vm = ViewModel(service: service)
    // ... 20 行重复代码
}

@Test func test2() {
    let service = MockService()
    service.stubbedResult = .success(User(id: "2", name: "B"))
    let vm = ViewModel(service: service)
    // ... 20 行几乎相同的代码
}

// 正确：适度抽象，使用 Factory 减少重复但保持可读
@Test func loadUserSuccess() {
    let vm = makeViewModel(stubbedUser: UserFactory.make(name: "张三"))
    await vm.loadUser()
    #expect(vm.userName == "张三")
}

// 辅助方法放在测试 Suite 内部
private func makeViewModel(
    stubbedUser: User? = nil,
    stubbedError: Error? = nil
) -> UserViewModel {
    let mock = MockUserService()
    mock.stubbedUser = stubbedUser
    mock.stubbedError = stubbedError
    return UserViewModel(service: mock)
}
```

### 稳定性

| 不稳定来源 | 解决方案 |
|-----------|---------|
| `Date()` | 注入 `DateProvider` Mock |
| `UUID()` | 注入 `UUIDProvider` Mock 或用固定值 |
| `Int.random()` | 注入随机数生成器 |
| 网络请求 | Protocol Mock |
| 文件系统 | In-memory Fake |
| `Task.sleep` / `DispatchQueue.asyncAfter` | 使用 `await` + `XCTestExpectation` |
| 动画 / 渲染时机 | 禁用动画 |

### 速度

| 测试类型 | 目标时间 |
|---------|---------|
| 单个单元测试 | < 0.1 秒 |
| 单个集成测试 | < 1 秒 |
| 单个 UI 测试 | < 10 秒 |
| 整个单元测试套件 | < 30 秒 |
| 整个 UI 测试套件 | < 5 分钟 |

**慢测试排查：**
- Xcode → Product → Test → 查看每个测试执行时间
- 超时的测试通常是等待异步操作或文件 I/O
- 优先优化最慢的 5 个测试

---

## 测试覆盖率分析

### Xcode 覆盖率工具使用

1. Edit Scheme → Test → Options → 勾选 "Code Coverage"
2. 运行测试后：Report Navigator → 选择最近的测试报告 → Coverage
3. 查看每个文件/方法的覆盖率百分比
4. 点击文件可看到逐行覆盖情况（绿色=已覆盖，红色=未覆盖）

### 覆盖率目标设定

| 代码类型 | 推荐覆盖率 | 说明 |
|---------|-----------|------|
| 业务逻辑（ViewModel/Service） | > 80% | 核心逻辑必须高覆盖 |
| 数据模型（Codable） | > 90% | 解析错误影响面广 |
| 工具类/扩展 | > 70% | 通用代码应稳定可靠 |
| UI 层（View） | 不设硬性目标 | 用快照测试替代 |
| 网络层封装 | > 60% | 核心路径覆盖即可 |
| 全项目 | > 60% | 追求有价值的覆盖 |

**不追求 100% 的原因：**
- 简单的 getter/setter 不需要测试
- UI 布局代码用快照测试更有效
- 某些防御性代码难以触发且价值低
- 追求数字会导致写无价值的测试

### 有价值的覆盖 vs 无价值的覆盖

```swift
// 有价值：测试了业务规则
@Test("VIP 折扣计算正确")
func vipDiscount() {
    let price = PriceCalculator.finalPrice(
        original: 100, userType: .vip, coupon: nil
    )
    #expect(price == 80)  // VIP 打 8 折
}

// 无价值：只为凑覆盖率
@Test("User 初始化")
func userInit() {
    let user = User(name: "test")
    #expect(user.name == "test")  // 测试了 Swift 的 struct init，没有业务价值
}
```

---

## 遗留代码的测试补充策略

### 特征测试（先记录现有行为）

对于没有测试的遗留代码，第一步不是"测试它是否正确"，而是"记录它现在的行为"。

```swift
// 步骤 1：运行遗留代码，记录当前输出
@Test("记录当前折扣计算行为")
func characterizeDiscountCalculation() {
    // 这些期望值来自实际运行结果，不是需求文档
    #expect(LegacyPriceEngine.discount(for: 100, level: 1) == 5)
    #expect(LegacyPriceEngine.discount(for: 100, level: 2) == 12)
    #expect(LegacyPriceEngine.discount(for: 0, level: 1) == 0)
    #expect(LegacyPriceEngine.discount(for: -10, level: 1) == 0)  // 发现：负数返回0
}
```

**特征测试的目的：**
- 不是验证正确性，而是防止无意修改行为
- 重构时作为安全网
- 逐步理解遗留代码的实际行为

### 关键路径优先

```
优先级排序：
1. 支付/交易相关 — 出错直接损失金钱
2. 用户认证 — 出错影响安全
3. 核心业务流程 — 出错影响主要功能
4. 数据持久化 — 出错可能丢失数据
5. 其他辅助功能 — 出错影响体验
```

### 逐步提高覆盖率

```
第一阶段（1-2 周）：
  - 为最关键的 3-5 个类写特征测试
  - 建立测试基础设施（Mock 基类、Factory、Helper）
  - 目标：覆盖率从 0% → 20%

第二阶段（3-4 周）：
  - 新增/修改的代码必须有测试（规则执行）
  - 为高频修改的文件补充测试
  - 目标：覆盖率 20% → 40%

第三阶段（持续）：
  - 修 Bug 时补充回归测试
  - 重构时先写特征测试
  - 逐步覆盖更多模块
  - 目标：覆盖率 40% → 60%+
```

---

## 测试审查 Checklist

在 Code Review 中审查测试代码时使用：

### 基本结构
- [ ] 测试遵循 Arrange-Act-Assert 结构
- [ ] 每个测试只验证一个行为
- [ ] 测试命名描述了场景和预期结果
- [ ] 新文件使用 Swift Testing（除非有特殊原因）

### 测试内容
- [ ] 覆盖了正常路径
- [ ] 覆盖了关键的边界条件
- [ ] 覆盖了主要的错误路径
- [ ] 断言验证的是行为结果，不是实现细节

### 测试质量
- [ ] 没有 `sleep` 或 `asyncAfter` 等待
- [ ] 没有测试间的共享可变状态
- [ ] Mock 只用于外部依赖，没有 Mock 被测对象
- [ ] 期望值是硬编码的，不是从生产代码计算的
- [ ] 没有永远通过的空断言

### 可维护性
- [ ] 测试数据使用 Factory/Builder
- [ ] 没有过度抽象（读测试能理解在测什么）
- [ ] 没有大量重复代码（适度提取辅助方法）
- [ ] 新增 Mock 与现有 Mock 风格一致

---

## LLM 常犯测试错误清单

LLM（包括 AI 编程助手）在生成测试代码时常犯以下错误：

| 错误 | 说明 | 正确做法 |
|------|------|---------|
| **测试了 Mock 而非真实代码** | Mock 了被测对象本身，测试等于测 Mock | 只 Mock 外部依赖 |
| **在测试中重复生产逻辑** | 复制计算公式作为期望值 | 用硬编码的已知正确值 |
| **永真断言** | `#expect(true)`、`#expect(result != nil)` 对所有输入都成立 | 验证具体的值 |
| **忽略异步时序** | 不 await 就检查结果 | 正确使用 `await` |
| **过度测试 getter/setter** | 测试 `user.name == "test"` | 只测试有逻辑的代码 |
| **测试与实现 1:1 对应** | 每个私有方法一个测试 | 通过公开接口测试行为 |
| **忽略边界和异常** | 只测试 happy path | 覆盖空值、极值、错误 |
| **共享 Mock 状态** | 多个测试共用同一个 Mock 实例的 var 状态 | 每个测试创建新 Mock |
| **生成不能编译的测试** | 使用不存在的 API 或错误的类型 | 确保测试可编译运行 |
| **不检查副作用** | 只检查返回值，忽略保存/通知等副作用 | 通过 Mock 验证副作用 |

**使用 LLM 生成测试后的必须操作：**
1. 确认测试能编译通过
2. 确认测试能正确失败（改一下期望值，看是否报错）
3. 检查是否有上表中的错误
4. 删除无价值的测试（纯 getter 测试等）

---

## 速查 Checklist

- [ ] 测试遵循 Arrange-Act-Assert 结构
- [ ] 测试独立运行，无顺序依赖
- [ ] 测试稳定，无 Flaky Test
- [ ] 单个测试 < 1s，套件 < 30s
- [ ] 覆盖率关注业务逻辑而非数字
- [ ] 遗留代码先写特征测试再重构
- [ ] Code Review 时使用测试审查 Checklist
- [ ] LLM 生成的测试必须人工审查
- [ ] 测试行为而非实现
- [ ] 期望值硬编码，不从生产代码计算
