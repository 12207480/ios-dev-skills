# 测试审查

> 测试覆盖率与测试质量审查。基础平台规范见 `platform-checklist.md`。

## 测试覆盖

```
- [ ] 核心业务逻辑是否有单元测试？
- [ ] 新增的分支条件是否有对应测试？
- [ ] Bug 修复是否有复现该 Bug 的测试用例？
- [ ] 修改了公共代码，现有测试是否仍通过？
- [ ] 边界值是否覆盖？（空数组、nil、最大值、负数）
```

## 测试质量

```
- [ ] 测试是否真正验证了行为？（不是只调用了方法就算过）
      → 有断言且断言验证了关键结果
- [ ] 测试的断言是否足够具体？
      → XCTAssertEqual(result, expected) ✅
      → XCTAssertNotNil(result) ❌（太宽泛，任何非 nil 值都通过）
- [ ] 测试命名是否清晰？（test_方法_条件_预期）
      → test_login_withInvalidPassword_shouldReturnError ✅
      → testLogin ❌
- [ ] Mock / Stub 是否合理？（不要 Mock 被测对象本身）
      → Mock 外部依赖（网络、数据库），不要 Mock 内部实现细节
- [ ] 测试之间是否独立？（无共享可变状态）
      → setUp/tearDown 正确重置状态
      → 测试执行顺序不影响结果
- [ ] 测试是否易于维护？
      → 避免过度使用 Mock 导致测试与实现细节耦合
      → 修改内部实现不应导致大量测试失败
```

## Swift Testing（Xcode 16+）

```
- [ ] 新测试是否使用 @Test / @Suite 而非 XCTestCase？
      → 新代码应优先使用 Swift Testing 框架
- [ ] 使用 #expect() 替代 XCTAssert 系列断言？
      → #expect(result == expected) ✅
      → #expect(array.isEmpty) ✅
- [ ] 重复逻辑测试是否用 @Test(arguments:) 参数化？
      → 替代多个几乎相同的测试方法
      → @Test(arguments: ["", " ", "  "])
        func emptyInput(input: String) { ... }
- [ ] 异步测试是否直接使用 async 而非 XCTestExpectation？
      → @Test func fetchData() async throws { ... } ✅
- [ ] 使用 #require() 替代 XCTUnwrap？
      → let value = try #require(optionalValue)
- [ ] Tag 是否用于测试分组和筛选？
      → @Test(.tags(.networking)) 替代 XCTestCase 类分组
```

## LLM 常犯测试错误

> AI 辅助生成的测试代码中高频出现的问题，审查时需特别留意。

### 错误一：测试只调用不断言

```
看到 X → 测试方法只调用被测方法，没有任何断言
判断 Y → 无效测试，永远通过，不验证任何行为
建议 Z → 要求添加具体断言，验证返回值/状态变化/副作用

❌ 典型无效测试：
@Test func testFetch() async throws {
    let service = UserService()
    _ = try await service.fetchUser(id: "123")  // 只调用，不断言
}

✅ 有效测试：
@Test func testFetch() async throws {
    let service = UserService(client: MockClient())
    let user = try await service.fetchUser(id: "123")
    #expect(user.name == "Alice")
    #expect(user.id == "123")
}
```

### 错误二：断言过于宽泛

```
看到 X → 断言只检查 != nil 或 .isEmpty == false
判断 Y → 宽泛断言掩盖了实际 Bug，返回任何值都能通过
建议 Z → 断言应验证具体值或具体状态

❌ 宽泛断言：
#expect(result != nil)
#expect(!items.isEmpty)

✅ 具体断言：
#expect(result == .success("expected_value"))
#expect(items.count == 3)
#expect(items.first?.name == "Alice")
```

### 错误三：Mock 泄漏实现细节

```
看到 X → Mock 对象精确模拟了内部调用顺序和参数
判断 Y → 测试与实现细节耦合，重构时大量测试失败
建议 Z → Mock 应基于接口行为，不关心内部调用顺序

❌ 过度 Mock：
mock.verify(callOrder: [.fetchToken, .refreshIfNeeded, .fetchUser])

✅ 行为验证：
#expect(result.user.name == "Alice")  // 关心结果，不关心过程
```

### 错误四：异步测试使用 sleep 等待

```
看到 X → 测试中使用 Task.sleep / Thread.sleep / XCTWaiter 等待固定时间
判断 Y → 不可靠（CI 环境可能更慢）、拖慢测试执行速度
建议 Z → 使用 async/await 直接等待结果，或用 expectation 的 fulfill

❌ 固定等待：
try await Task.sleep(for: .seconds(2))
XCTAssertEqual(viewModel.state, .loaded)

✅ 直接 await：
await viewModel.loadData()
#expect(viewModel.state == .loaded)
```

### 错误五：测试数据硬编码且无语义

```
看到 X → 测试中充斥 "test"、"abc"、123 等无意义数据
判断 Y → 难以理解测试意图，无法判断边界值是否覆盖
建议 Z → 使用语义化测试数据，或抽取 Factory 方法

❌ 无语义数据：
let user = User(name: "test", age: 1, email: "a@b.c")

✅ 语义化数据：
let adultUser = User(name: "张三", age: 25, email: "zhangsan@example.com")
let minorUser = User(name: "李四", age: 15, email: "lisi@example.com")
```

### 错误六：忽略错误路径测试

```
看到 X → 只有 happy path 测试，没有错误/异常路径
判断 Y → 实际 Bug 大多发生在错误处理路径
建议 Z → 要求补充网络错误、无效输入、权限拒绝等异常测试

应覆盖的错误路径：
  → 网络请求超时/失败
  → 无效输入（空字符串、越界值）
  → 解码失败（字段缺失、类型不匹配）
  → 权限被拒绝
  → 磁盘空间不足
```

## 常见问题速查

### 看到测试文件但覆盖不足

```
判断方法：
  1. 检查核心业务逻辑（Service/ViewModel）是否有对应测试文件
  2. 对比新增代码的分支数与测试用例数
  3. Bug 修复是否有回归测试

建议标签：
  核心逻辑无测试 → 🟡 Should Fix
  Bug 修复无回归测试 → 🟡 Should Fix
  工具方法无测试 → 🟢 Nit
```

### 看到 `XCTestCase` 在新项目中

```
判断标准：
  项目使用 Xcode 16+ 且最低部署目标支持 → 建议迁移到 Swift Testing
  项目有大量现有 XCTest → 新测试用 Swift Testing，不强求迁移旧测试
  需要 UI 测试 → 仍使用 XCUITest（Swift Testing 不支持 UI 测试）
```
