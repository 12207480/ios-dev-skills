---
name: ios-testing
description: Guides iOS test creation through a systematic five-phase methodology from test strategy design to comprehensive coverage verification. Use when writing unit tests, UI tests, snapshot tests, or integration tests for iOS applications, or when establishing testing practices for existing codebases.
---

# iOS 测试

## 核心原则

**先理解被测行为，再设计测试策略，写有价值的测试，测试本身也需要质量审查。**

测试不是"写完代码补一下"的后续任务。测试是对代码行为的精确描述——如果你写不出测试，说明你还没完全理解被测代码的行为。

### 测试前必须回答的 4 个问题

1. **被测行为是什么？** — 这段代码在什么输入下应该产出什么结果
2. **边界在哪里？** — 空值、极值、异常输入时应该怎样
3. **依赖有哪些？** — 网络、数据库、系统服务等外部依赖如何隔离
4. **值得测什么？** — 哪些测试真正防止回归，哪些是浪费时间

## 绝对禁止

- 测试实现细节而非行为（如：验证内部方法调用顺序）
- 测试之间相互依赖（如：测试 B 依赖测试 A 创建的数据）
- Mock 被测对象本身（如：Mock 了要测试的 ViewModel）
- 复制粘贴生产代码逻辑到测试中作为"期望值"
- 写只能通过不能失败的测试（永真断言）
- 用 `sleep` / `asyncAfter` 等待异步结果（使用正确的等待机制）

---

## 复杂度速判

| 级别 | 特征 | 示例 | 流程路径 |
|------|------|------|---------|
| **S（小）** | 为单个方法写单元测试，被测代码无外部依赖 | 纯函数、数据模型、工具方法 | 阶段一 → 阶段四 → 阶段五 |
| **M（中）** | 为一个模块建立测试覆盖，需要 Mock 依赖 | ViewModel 测试、Service 层测试、数据层测试 | 完整五阶段 |
| **L（大）** | 为遗留项目建立测试体系，涉及测试基础设施搭建 | 全项目测试策略、CI 测试集成、测试框架迁移 | 完整五阶段 + 测试策略评审 |

> **S 级简化说明：** 跳过阶段二（测试策略设计）和阶段三（测试基础设施搭建），直接编写测试。但仍需在阶段五做测试质量审查。若发现依赖复杂度超出预期，立即升级为 M 级。

---

## 五阶段流程

### 阶段一：被测代码分析

**目标：** 完全理解被测代码的接口、依赖和行为边界。

1. **接口分析** — 公开方法签名、输入参数类型和范围、返回值/副作用
2. **依赖分析** — 外部依赖列表（网络/持久化/系统服务）、依赖注入方式
3. **行为边界** — 正常路径、边界条件、异常路径、并发场景
4. **现有测试** — 已有测试覆盖了什么、遗漏了什么

> 详细分析方法见 `references/unit-testing-patterns.md` → 测试用例设计方法

**门控输出：** 被测代码分析摘要（含接口清单、依赖列表、行为边界矩阵）。写不出行为边界说明代码还没读懂，不进入下一阶段。

### 阶段二：测试策略设计

**目标：** 选择合适的测试类型，确定覆盖范围，设计测试用例。

1. **测试类型选择** — 单元测试 / 集成测试 / UI 测试 / 快照测试
2. **框架选择** — Swift Testing（推荐）vs XCTest（兼容旧项目）
3. **覆盖范围** — 按优先级排序：核心逻辑 > 错误处理 > 边界条件 > UI 状态
4. **测试用例清单** — 每个用例包含：场景描述、输入、预期输出

> 测试框架选择见 `references/swift-testing-guide.md`
> UI 测试策略见 `references/ui-testing-guide.md`
> 快照测试策略见 `references/snapshot-testing-guide.md`

**门控输出：** 测试策略文档（含测试类型、用例清单、优先级排序）。用例清单为空或无法说明优先级理由则不通过。

### 阶段三：测试基础设施搭建

**目标：** 准备 Mock/Stub、测试数据和辅助工具，为编写测试扫清障碍。

1. **Mock/Stub 准备** — 为每个外部依赖创建测试替身
2. **测试数据准备** — Factory / Builder / JSON fixture
3. **辅助工具** — 通用断言封装、等待工具、Page Object（UI 测试）
4. **测试目标配置** — 确保测试 Target 正确关联被测代码

**测试 Target 配置步骤（Xcode）：**
1. File → New → Target → 选择 "Unit Testing Bundle" 或 "UI Testing Bundle"
2. 设置 Target 名称（如 `MyAppTests`），确认 "Target to be Tested" 选择主 App Target
3. 在测试 Target 的 Build Settings 中确认 `Host Application` 设置正确
4. 在测试 Target 的 Build Phases → "Dependencies" 中添加主 App Target
5. 需要访问内部符号时在测试文件中使用 `@testable import MyApp`
6. SPM 依赖需在 Package.swift 的 `.testTarget` 中声明，或在 Xcode 的 Frameworks and Libraries 中添加

> Mock 模式详解见 `references/mock-stub-patterns.md`
> 测试数据管理见 `references/unit-testing-patterns.md` → 测试数据管理

**门控输出：** Mock/Stub 可用 + 测试数据就绪 + 测试 Target 可编译。基础设施不完备则不开始写测试。

### 阶段四：编写测试

**目标：** 按优先级实现测试用例，确保每个测试独立、可读、有明确断言。

1. **按优先级顺序** — 先写最高优先级的测试用例
2. **Arrange-Act-Assert** — 每个测试遵循三段式结构
3. **一个测试一个行为** — 不在单个测试中验证多个不相关行为
4. **描述性命名** — 测试名称说明场景和预期结果
5. **先红后绿**（Bug 修复场景）— 先写失败的测试，再实现修复

> 单元测试模式见 `references/unit-testing-patterns.md`
> Swift Testing 用法见 `references/swift-testing-guide.md`
> UI 测试编写见 `references/ui-testing-guide.md`
> 快照测试编写见 `references/snapshot-testing-guide.md`

**门控输出：** 所有测试通过 + 覆盖了用例清单中的高优先级项。测试失败或高优先级用例遗漏则不通过。

### 阶段五：测试质量审查

**目标：** 确保测试本身的质量——可读、稳定、可维护。

1. **可读性** — 测试意图一目了然、命名描述性强、结构清晰
2. **独立性** — 无测试间依赖、无执行顺序依赖、无共享可变状态
3. **稳定性** — 无 Flaky Test、确定性结果、不依赖时间/网络/文件系统
4. **可维护性** — DRY 但不过度抽象、修改被测代码时测试容易调整
5. **速度** — 单个测试 < 1s，整个测试套件 < 30s

> 质量标准详见 `references/test-quality-checklist.md`

**门控输出：** 测试质量审查清单全部通过。任一项不满足则修复后重新审查。

---

## Topic Router

| 话题 | 引用文件 |
|------|---------|
| Swift Testing vs XCTest、断言、参数化测试 | `references/swift-testing-guide.md` |
| Mock/Stub/Spy/Fake、依赖注入、Protocol Mock | `references/mock-stub-patterns.md` |
| 测试用例设计、ViewModel 测试、数据层测试 | `references/unit-testing-patterns.md` |
| async/await、Combine、AsyncStream、Actor、Task 取消 | `references/async-testing-patterns.md` |
| 集成测试、多层协作测试、端到端数据流 | `references/integration-testing-guide.md` |
| XCUITest、Page Object、UI 测试策略 | `references/ui-testing-guide.md` |
| swift-snapshot-testing、视图快照、CI 配置 | `references/snapshot-testing-guide.md` |
| 测试质量标准、覆盖率、遗留代码测试、审查清单 | `references/test-quality-checklist.md` |

---

## 红旗信号表

| 违规行为 | 为什么是错误的 | 正确做法 |
|---------|--------------|---------|
| 测试实现细节而非行为 | 重构时测试全部失败，但功能正常 | 测试公开接口的输入输出 |
| 测试之间有执行顺序依赖 | 单独运行时通过，一起运行时失败 | 每个测试独立 setUp/tearDown |
| 在测试中复制生产代码逻辑 | 生产代码有 Bug 测试也不会发现 | 用硬编码的期望值 |
| Mock 了被测对象本身 | 测试了 Mock 而不是真正的代码 | 只 Mock 外部依赖 |
| 用 `sleep()` 等待异步结果 | 慢且不稳定 | 用 `await` / `XCTestExpectation` |
| 测试中没有断言 | 测试永远通过，无任何价值 | 每个测试必须有明确断言 |
| 过度 Mock 导致测试与实现强耦合 | 重构时测试全部报错 | 只 Mock 必要的外部依赖 |
| 单个测试验证 10+ 个行为 | 失败时无法快速定位原因 | 一个测试一个行为 |
| 测试数据用 magic number 无说明 | 可读性差，改动困难 | 用命名常量或 Factory |
| 忽略 Flaky Test | 逐渐侵蚀团队对测试的信任 | 立即修复或隔离 |

---

## 输出格式规范

**被测代码分析摘要**（阶段一输出）
```
模块：[模块名/文件路径]
公开接口：[方法签名列表]
外部依赖：[依赖列表及注入方式]
行为边界：正常路径 [N] 个 | 边界条件 [N] 个 | 异常路径 [N] 个
```

**测试策略文档**（阶段二输出）
```
测试类型：[单元测试 / 集成测试 / UI 测试 / 快照测试]
测试框架：[Swift Testing / XCTest]
用例清单：
  P0（必须）：[用例列表]
  P1（应该）：[用例列表]
  P2（可选）：[用例列表]
```

**测试质量审查报告**（阶段五输出）
```
覆盖范围：P0 [N/N] | P1 [N/N] | P2 [N/N]
质量指标：可读性 ✓/✗ | 独立性 ✓/✗ | 稳定性 ✓/✗ | 速度 ✓/✗
遗留问题：[如有]
```

---

## 与其他 Skill 的协作

- **ios-feature-impl**：新功能实现的测试覆盖，在编码阶段调用测试流程
- **ios-issue-fix**：Bug 修复的"先红后绿"测试，验证修复有效性
- **ios-refactor**：重构前建立特征测试，重构后验证行为不变
