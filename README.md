# ios-dev-skills

让 Claude Code 成为你的 iOS 高级工程师。6 个 Skills 覆盖实现、审查、修复、性能优化、重构、测试全链路，支持 Swift 6 / SwiftUI / UIKit / ObjC 混编项目。

## 概览

这是一套为 [Claude Code](https://claude.com/claude-code) 打造的 iOS 开发 Skills 套件，覆盖 iOS 开发全生命周期：

```
需求 → 实现 → 测试 → 审查 → 修复 → 优化 → 重构
       ↓      ↓      ↓      ↓      ↓      ↓
  feature  testing  code   issue   perf  refactor
   impl            review   fix  optimize
```

| 指标 | 数值 |
|------|------|
| Skills 数量 | 6 |
| 文件总数 | 58 |
| 代码行数 | 15,500+ |
| 技术栈 | Swift 6 · SwiftUI · UIKit · Objective-C |

## Skills 一览

### ios-feature-impl — 功能实现

从需求分析到代码提交的五阶段方法论。

- 需求理解与分析（类型判断、设计稿审阅、API 审阅）
- 上下文分析（代码定位、设计意图理解）
- 关联分析（数据/UI/状态/导航/业务规则五维度）
- SwiftUI 现代模式 + UIKit 实现模式
- 网络层模式、持久化模式、测试策略

### ios-code-review — 代码审查

PR 审查的五阶段方法论，从理解意图到给出反馈。

- Swift 安全性（可选值、错误处理、Swift 6 并发）
- Objective-C 安全性（nil/集合/Block/KVC-KVO/混编）
- 内存与线程安全（循环引用、结构化并发）
- SwiftUI / UIKit 审查清单
- 无障碍与隐私审查
- 审查沟通技巧（提问法、建议而非命令）

### ios-issue-fix — 问题修复

从读 Issue 到提 PR 的五阶段 Bug 修复方法论。

- 7 大问题分类（UI/逻辑/数据/交互/兼容/崩溃/性能）
- 分类型排查策略 + SwiftUI 专项排查
- 崩溃分析与符号化（dSYM、Crashlytics）
- 网络问题排查（HTTP 状态码、Codable 解析、SSL）
- Xcode 调试工具速查（LLDB/View Debugger/Instruments）

### ios-performance-optimize — 性能优化

系统化的五阶段性能优化方法论：量化 → 定位 → 分析 → 优化 → 验证。

- 启动优化（pre-main/post-main、二进制重排）
- 内存优化（OOM 分析、图片降采样、泄漏排查）
- 渲染优化（离屏渲染、列表滚动、SwiftUI 重绘）
- 包体积优化（资源瘦身、代码瘦身、App Thinning）
- Swift Concurrency 性能（Actor 重入、Task 开销）
- 数据库性能（Core Data 批量操作、索引优化）
- Instruments 深度使用指南

### ios-refactor — 代码重构

以"行为等价性"为核心的五阶段安全重构方法论。

- 安全重构技法（提取/内联/移动 + Strangler Fig Pattern）
- ObjC → Swift 渐进式迁移
- UIKit → SwiftUI 渐进式迁移
- Swift Concurrency 迁移（completion handler → async/await、Swift 6）
- 架构重构（MVC → MVVM、模块化拆分、SPM Package）
- 遗留代码处理策略（安全区域划定、特征测试）

### ios-testing — 测试编写

从测试策略设计到质量审查的五阶段测试方法论。

- Swift Testing 深度指南（@Test/@Suite/#expect/参数化/Trait）
- Mock/Stub/Spy/Fake 模式（Protocol-based、URLProtocol）
- 单元测试设计（等价类/边界值/决策表/状态转换）
- 异步测试（Combine/AsyncSequence/Actor/Task 取消）
- UI 测试（XCUITest + Page Object Pattern）
- 快照测试（swift-snapshot-testing + CI 配置）
- 集成测试（多层协作、端到端数据流）

## 统一设计模式

所有 6 个 Skills 遵循相同的设计模式：

| 设计元素 | 说明 |
|---------|------|
| **五阶段流程** | 每个 Skill 都有清晰的五阶段工作流 |
| **复杂度速判** | S/M/L 三级分类，简单任务走精简流程 |
| **Topic Router** | 按话题路由到对应的 reference 文件，按需加载 |
| **门控输出** | 每个阶段有明确的"通过条件"，防止跳步 |
| **红旗信号表** | 列出常见反模式及正确做法 |
| **Checklist** | 每个 reference 文件都有可执行的检查清单 |

## 安装

### 方式一：克隆到 Claude Code Skills 目录

```bash
# 克隆仓库
git clone https://github.com/12207480/ios-dev-skills.git

# 复制到 Claude Code skills 目录
cp -r ios-dev-skills/skills/ios-* ~/.claude/skills/
```

### 方式二：手动安装单个 Skill

```bash
# 只安装你需要的 Skill
cp -r ios-dev-skills/skills/ios-code-review ~/.claude/skills/
```

### 方式三：项目级安装

```bash
# 复制到项目的 .claude/skills/ 目录下，仅对当前项目生效
cp -r ios-dev-skills/skills/ios-* your-project/.claude/skills/
```

安装后 Claude Code 会自动检测并加载这些 Skills。

## 目录结构

```
ios-dev-skills/
├── skills/
│   ├── ios-feature-impl/          # 功能实现（11 文件）
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── requirement-analysis.md
│   │       ├── context-analysis.md
│   │       ├── impact-analysis.md
│   │       ├── coding-standards.md
│   │       ├── swiftui-patterns.md
│   │       ├── uikit-patterns.md
│   │       ├── networking-patterns.md
│   │       ├── persistence-patterns.md
│   │       ├── testing-strategy.md
│   │       └── scenario-quickref.md
│   ├── ios-code-review/           # 代码审查（11 文件）
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── swift-safety.md
│   │       ├── objc-safety.md
│   │       ├── memory-thread.md
│   │       ├── swiftui-review.md
│   │       ├── uikit-review.md
│   │       ├── modern-api.md
│   │       ├── platform-checklist.md
│   │       ├── accessibility-privacy.md
│   │       ├── testing-review.md
│   │       └── review-techniques.md
│   ├── ios-issue-fix/             # 问题修复（10 文件）
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── issue-classification.md
│   │       ├── root-cause-analysis.md
│   │       ├── debug-by-type.md
│   │       ├── swiftui-debugging.md
│   │       ├── crash-analysis.md
│   │       ├── network-debugging.md
│   │       ├── xcode-tools.md
│   │       ├── fix-patterns.md
│   │       └── verification.md
│   ├── ios-performance-optimize/  # 性能优化（9 文件）
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── launch-optimization.md
│   │       ├── memory-optimization.md
│   │       ├── rendering-optimization.md
│   │       ├── app-size-optimization.md
│   │       ├── network-battery-optimization.md
│   │       ├── concurrency-optimization.md
│   │       ├── database-optimization.md
│   │       └── instruments-guide.md
│   ├── ios-refactor/              # 代码重构（8 文件）
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── safe-refactoring-techniques.md
│   │       ├── objc-to-swift-migration.md
│   │       ├── uikit-to-swiftui-migration.md
│   │       ├── concurrency-migration.md
│   │       ├── architecture-refactoring.md
│   │       ├── legacy-code-strategies.md
│   │       └── refactoring-verification.md
│   └── ios-testing/               # 测试编写（9 文件）
│       ├── SKILL.md
│       └── references/
│           ├── swift-testing-guide.md
│           ├── mock-stub-patterns.md
│           ├── unit-testing-patterns.md
│           ├── async-testing-patterns.md
│           ├── integration-testing-guide.md
│           ├── ui-testing-guide.md
│           ├── snapshot-testing-guide.md
│           └── test-quality-checklist.md
└── README.md
```

## 技术覆盖

| 领域 | 覆盖内容 |
|------|---------|
| **语言** | Swift 6、Objective-C |
| **UI 框架** | SwiftUI（iOS 17+ @Observable）、UIKit、混编 |
| **并发** | async/await、Actor、Sendable、structured concurrency |
| **架构** | MVVM、MVC、Coordinator、SPM 模块化 |
| **持久化** | Core Data、SwiftData、UserDefaults、Keychain |
| **网络** | URLSession、Codable、分页加载、SSL Pinning |
| **测试** | Swift Testing、XCTest、XCUITest、swift-snapshot-testing |
| **工具** | Instruments、LLDB、View Debugger、Memory Graph |
| **平台** | iOS 15+、Widget、App Extension |

## 工作流示例

```
你：实现一个带分页加载的用户列表页面
Claude：[自动加载 ios-feature-impl] 按五阶段流程执行...

你：审查这个 PR
Claude：[自动加载 ios-code-review] 按审查流程逐层检查...

你：用户反馈列表页滑动卡顿
Claude：[自动加载 ios-performance-optimize] 先建立性能基线...

你：这个页面有个崩溃需要修复
Claude：[自动加载 ios-issue-fix] 先分析崩溃日志...

你：把这个 ViewController 重构为 SwiftUI
Claude：[自动加载 ios-refactor] 先评估范围，建立安全网...

你：给 UserViewModel 写单元测试
Claude：[自动加载 ios-testing] 先分析被测代码的行为边界...
```

## 适用场景

- 独立 iOS 开发者，希望 Claude Code 提供高级工程师级别的协助
- iOS 团队，希望统一开发规范和质量标准
- 处理 Swift/ObjC 混编的大型遗留项目
- 从 UIKit 向 SwiftUI 迁移的项目
- 需要系统化性能优化的项目

## 贡献

欢迎提交 Issue 和 Pull Request。贡献时请遵循：

1. 保持与现有 Skills 一致的设计模式（五阶段流程、Topic Router、门控输出等）
2. 所有内容使用中文
3. 代码示例使用 Swift，ObjC 仅在混编场景中使用
4. 每个 reference 文件必须包含总结 Checklist

## 许可证

[MIT](LICENSE)
