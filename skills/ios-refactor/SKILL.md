---
name: ios-refactor
description: Guides iOS code refactoring through a systematic five-phase methodology that ensures behavioral equivalence throughout the process. Use when migrating from ObjC to Swift, UIKit to SwiftUI, refactoring architecture patterns, modularizing codebases, or cleaning up technical debt in iOS projects.
---

# iOS 代码重构

## 核心原则

**先证明行为正确，再安全地改变结构，每一步都可验证，改完必须证明等价。**

重构不是重写。重构是在**不改变外部可观察行为**的前提下，改善代码的内部结构。在动手之前，必须先回答：

1. **现在行为是什么？** — 当前代码的输入、输出、副作用是什么
2. **安全网在哪？** — 有什么手段能证明行为没变
3. **怎么一步步改？** — 每一步都能编译通过、测试通过
4. **怎么证明等价？** — 重构后的行为与重构前完全一致

## 绝对禁止

- 边重构边加功能（重构 PR 和功能 PR 必须分开）
- 跳过测试直接重构（没有安全网的重构是赌博）
- 一次性大爆炸重构（必须小步迭代，每步可回滚）
- 重构完不验证就提交（必须证明行为等价）
- 在没有理解现有代码设计意图的情况下重构

---

## 复杂度速判

在开始五阶段流程之前，先判断重构复杂度，选择合适的流程路径：

| 级别 | 特征 | 示例 | 流程路径 |
|------|------|------|---------|
| **S（小）** | 原子操作，改动 ≤ 3 个文件，无跨模块影响 | 重命名、提取方法、内联变量、消除魔法数 | 阶段一 → 阶段四 → 阶段五 |
| **M（中）** | 单一模块内的架构调整，4-10 个文件 | 从 VC 提取 ViewModel、引入 Protocol 解耦、单文件 ObjC→Swift | 完整五阶段 |
| **L（大）** | 跨模块重构或技术栈迁移，> 10 个文件 | 整体架构迁移、ObjC→Swift 批量迁移、UIKit→SwiftUI 页面迁移 | 完整五阶段 + 方案评审 + 分阶段交付 |

> **S 级简化说明：** 跳过阶段二（安全网建立）和阶段三（策略设计），但仍需在阶段四确认现有测试通过，在阶段五做基本的等价性验证。若实现过程中发现影响超出预期，立即升级为 M 级。

---

## 五阶段流程

### 阶段一：重构范围评估

**目标：** 理解现有代码 + 确定重构目标 + 建立行为基线。

1. **理解现有代码**
   - 阅读待重构代码，理解其设计意图和职责边界
   - 梳理依赖关系（谁调用了它？它调用了谁？）
   - 记录当前代码的所有外部可观察行为（输入/输出/副作用）

2. **明确重构目标**
   - 重构要解决什么问题？（可读性 / 可维护性 / 性能 / 技术栈升级）
   - 重构的边界在哪？（哪些要改、哪些不改）
   - 重构后的预期结构是什么样的？

3. **建立行为基线**
   - 记录当前测试覆盖情况
   - 手动验证关键路径的当前行为
   - 收集性能基准（如涉及性能敏感代码）

> 详细流程见 `references/safe-refactoring-techniques.md`

**门控输出：** 重构范围文档（含现有行为清单、重构目标、边界定义、依赖关系图）。无法描述"现有代码为什么这样设计"则不通过。

### 阶段二：安全网建立

**目标：** 在动手重构之前，补充足够的测试覆盖，确保有手段验证行为等价。

1. **评估现有测试**
   - 待重构代码的测试覆盖率如何？
   - 现有测试是否覆盖了所有外部可观察行为？
   - 哪些关键路径缺少测试？

2. **补充测试**
   - 为缺少覆盖的行为编写特征测试（Characterization Test）
   - 为关键路径编写金丝雀测试
   - 考虑快照测试（UI 重构场景）

3. **确认安全网就绪**
   - 所有现有测试通过
   - 新增测试覆盖了待重构代码的核心行为
   - 明确验证手段（自动化测试 / 手动验证 checklist）

> 详细流程见 `references/legacy-code-strategies.md` 和 `references/refactoring-verification.md`

**门控输出：** 测试安全网报告（含覆盖率、新增测试清单、验证手段确认）。关键行为无测试覆盖则不通过。

### 阶段三：重构策略设计

**目标：** 选择合适的重构模式，制定可逐步执行的步骤序列。

1. **选择重构模式**
   - 原子重构操作：提取方法、内联、移动、重命名等
   - 结构性重构模式：Strangler Fig、Branch by Abstraction
   - 技术栈迁移模式：渐进迁移、文件级迁移

2. **制定步骤序列**
   - 将大重构分解为可独立验证的小步骤
   - 每步都必须：编译通过 + 测试通过 + 可独立提交
   - 步骤之间的依赖关系要清晰

3. **风险评估**
   - 哪些步骤风险最高？
   - 每步的回滚方案是什么？
   - 是否需要 Feature Flag 或灰度发布？

> 详细流程见对应的重构类型参考文件：
> - ObjC → Swift：`references/objc-to-swift-migration.md`
> - UIKit → SwiftUI：`references/uikit-to-swiftui-migration.md`
> - 架构重构：`references/architecture-refactoring.md`
> - 原子操作：`references/safe-refactoring-techniques.md`

**门控输出：** 重构步骤序列（含每步的操作、验证标准、回滚方案）。步骤不可独立验证则不通过。

### 阶段四：逐步执行重构

**目标：** 按步骤序列执行，每步都编译通过 + 测试通过。

**执行原则：**
- **一步一提交**：每个原子操作完成后立即提交，commit message 说明做了什么
- **先编译后测试**：每步先确保编译通过，再运行测试
- **发现问题立即停**：如果某步导致测试失败，立即回滚该步骤并分析原因
- **不混入功能变更**：重构过程中发现需要加功能或修 Bug，记录下来但不在本次重构中处理
- **保持代码始终可工作**：任何时刻 checkout 任一提交，代码都应该能编译和运行

**执行流程：**

```
对每个步骤：
  1. 执行重构操作
  2. 编译 → 失败则修复或回滚
  3. 运行测试 → 失败则分析原因
     - 测试本身需要更新（行为未变，只是内部结构变了）→ 更新测试
     - 行为发生了改变 → 回滚，重新分析
  4. 提交（描述清楚做了什么重构操作）
  5. 进入下一步
```

**门控输出：** 每步都有独立的提交，编译通过 + 测试通过。任何步骤失败则停止并回退到上一个安全状态。

### 阶段五：等价性验证

**目标：** 全面证明重构后的代码与重构前行为完全一致。

1. **回归测试**
   - 运行完整测试套件，确保零失败
   - 对比测试覆盖率（不应下降）

2. **行为对比**
   - 对照阶段一建立的行为基线，逐项验证
   - UI 重构：前后截图对比
   - API 重构：前后请求/响应对比

3. **性能对比**
   - 对比阶段一收集的性能基准
   - 使用 Instruments 对比关键指标
   - 确保无性能回退

4. **影响范围验证**
   - 验证所有调用方行为正常
   - 验证关联页面/模块正常
   - 验证边界条件和异常路径

> 详细流程见 `references/refactoring-verification.md`

**门控输出：** 等价性验证报告（含测试结果、行为对比、性能对比）。任何行为差异或性能回退则不通过。

---

## Topic Router

| 话题 | 引用文件 |
|------|---------|
| 原子重构操作、安全步骤、版本控制策略 | `references/safe-refactoring-techniques.md` |
| ObjC→Swift 迁移策略、类型桥接、混编管理 | `references/objc-to-swift-migration.md` |
| UIKit→SwiftUI 迁移、Hosting/Representable、状态迁移 | `references/uikit-to-swiftui-migration.md` |
| 架构模式迁移、模块化拆分、依赖管理 | `references/architecture-refactoring.md` |
| 遗留代码处理、安全区域、补充测试、解耦技巧 | `references/legacy-code-strategies.md` |
| 行为等价验证、性能对比、回归测试、PR 规范 | `references/refactoring-verification.md` |
| Swift Concurrency 迁移、async/await、Actor、Sendable | `references/concurrency-migration.md` |

---

## 红旗信号

| 违规行为 | 正确做法 |
|---------|---------|
| 边重构边加功能 | 重构 PR 和功能 PR 严格分开 |
| 没有测试就开始重构 | 先补充测试建立安全网 |
| 一次性大范围修改 | 小步迭代，每步可验证可回滚 |
| 重构完不验证等价性 | 对照行为基线逐项验证 |
| 不理解现有代码就重构 | 先读懂设计意图再动手 |
| 重构导致测试失败继续推进 | 立即停止，分析原因后再决定 |
| 只改结构不跑测试 | 每步都必须编译通过 + 测试通过 |
| 跳过性能对比 | 性能敏感代码必须前后对比 |

---

## 输出格式规范

### 重构范围文档

```
## 重构范围评估

**重构目标：** [要解决什么问题]
**重构类型：** [原子操作 / 架构调整 / 技术栈迁移]
**复杂度：** [S / M / L]

**现有行为清单：**
1. [输入 X → 输出 Y]
2. [操作 A → 副作用 B]
...

**重构边界：**
- 要改：[范围]
- 不改：[范围]

**依赖关系：**
- 调用方：[列表]
- 被调用方：[列表]

**风险评估：**
- [风险点] → [应对方案]
```

### 重构步骤序列

```
## 重构步骤

### 步骤 1：[操作描述]
- 操作：[具体做什么]
- 验证：[编译通过 + 哪些测试通过]
- 回滚：[如何回滚]

### 步骤 2：[操作描述]
...
```

### 等价性验证报告

```
## 等价性验证报告

**测试结果：** [全部通过 / 失败 N 个]
**覆盖率对比：** [重构前 X% → 重构后 Y%]

**行为对比：**
| 行为 | 重构前 | 重构后 | 等价 |
|------|--------|--------|------|
| ... | ... | ... | Y/N |

**性能对比：**
| 指标 | 重构前 | 重构后 | 差异 |
|------|--------|--------|------|
| ... | ... | ... | ... |

**结论：** [等价验证通过 / 存在差异需处理]
```

### PR 提交规范

标题：`refactor: [一句话描述重构内容]`

> 完整的 PR 提交规范（含描述模板、特殊规则）见 `references/refactoring-verification.md`

---

## 与其他 Skill 的协作

- **ios-feature-impl**：重构完成后新增功能时，切换到需求实现流程
  - 切换条件：重构 PR 已合并，且新功能需求已明确
- **ios-code-review**：重构 PR 提交后的代码审查
  - 切换条件：重构 PR 已创建，需要 review 等价性验证和代码质量
- **ios-issue-fix**：重构过程中发现 Bug 时，记录下来另开 PR 修复
  - 切换条件：重构过程中发现了非重构引入的既有 Bug
- **ios-testing**：安全网建立阶段需要补充测试时
  - 切换条件：阶段二发现测试覆盖不足，需要系统性补充测试
- **ios-performance-optimize**：阶段五等价性验证发现性能回退时
  - 切换条件：重构后性能基准测试指标超出可接受范围

---

## 端到端示例：从 ViewController 提取 ViewModel

以下用精简示例演示五阶段流程在"提取 ViewModel"场景下的实际执行过程。

### 阶段一：范围评估

```
重构目标：从 ProfileViewController 中提取业务逻辑到 ProfileViewModel
重构类型：架构调整
复杂度：M

现有行为清单：
1. viewDidLoad → 调用 loadProfile() → 显示用户信息
2. 点击保存 → 调用 saveProfile() → 显示成功/失败提示
3. 网络失败 → 显示错误弹窗

重构边界：
- 要改：ProfileViewController（提取逻辑）
- 不改：API 层、Model 层、导航逻辑
```

### 阶段二：安全网建立

```swift
// 为现有行为编写特征测试
func testLoadProfile_displaysUserName() {
    let vc = ProfileViewController()
    vc.loadViewIfNeeded()
    // 模拟网络返回
    mockAPI.stubProfile(name: "Alice")
    vc.viewDidLoad()
    XCTAssertEqual(vc.nameLabel.text, "Alice")
}
```

### 阶段三：策略设计

```
步骤序列：
1. 创建 ProfileViewModel，移入状态属性 → 编译 → 测试
2. 移入 loadProfile() 逻辑 → 编译 → 测试
3. 移入 saveProfile() 逻辑 → 编译 → 测试
4. VC 通过 Combine 绑定 ViewModel → 编译 → 测试
5. 删除 VC 中冗余代码 → 编译 → 测试
```

### 阶段四：逐步执行

```swift
// 步骤 1 commit: refactor: 创建 ProfileViewModel 并移入状态属性
@MainActor
class ProfileViewModel: ObservableObject {
    @Published var userName: String = ""
    @Published var email: String = ""
    @Published var errorMessage: String?
}

// 步骤 2 commit: refactor: 将 loadProfile 逻辑移至 ViewModel
func loadProfile() async {
    do {
        let profile = try await profileService.fetchProfile()
        userName = profile.name
        email = profile.email
    } catch {
        errorMessage = error.localizedDescription
    }
}

// 步骤 3 commit: refactor: 将 saveProfile 逻辑移至 ViewModel
// 步骤 4 commit: refactor: VC 通过 Combine 绑定 ViewModel
// 步骤 5 commit: refactor: 删除 VC 中冗余的状态属性和逻辑方法
```

### 阶段五：等价性验证

```
测试结果：全部通过（原有 8 个 + 新增 5 个 ViewModel 测试）
覆盖率对比：重构前 45% → 重构后 72%

行为对比：
| 行为         | 重构前            | 重构后            | 等价 |
|-------------|-------------------|-------------------|------|
| 加载用户信息  | 显示 name + email | 显示 name + email | Y    |
| 保存成功      | Toast 提示        | Toast 提示        | Y    |
| 网络错误      | Alert 弹窗        | Alert 弹窗        | Y    |

结论：等价验证通过
```
