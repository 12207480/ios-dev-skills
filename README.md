<div align="center">

# ios-dev-skills

**让 Claude Code 成为你的 iOS 高级工程师**

*Turn Claude Code into your Senior iOS Engineer*

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Swift](https://img.shields.io/badge/Swift-6.0-orange.svg)](https://swift.org)
[![Platform](https://img.shields.io/badge/Platform-iOS%2015+-blue.svg)](https://developer.apple.com/ios/)
[![Skills](https://img.shields.io/badge/Skills-6-green.svg)](#skills-一览)
[![Files](https://img.shields.io/badge/Files-58-lightgrey.svg)](#目录结构)
[![Lines](https://img.shields.io/badge/Lines-15%2C500%2B-brightgreen.svg)](#技术覆盖)

6 个 Skills 覆盖 **实现 · 审查 · 修复 · 性能优化 · 重构 · 测试** 全链路

支持 Swift 6 / SwiftUI / UIKit / Objective-C 混编项目

</div>

---

## 目录

- [为什么需要这个项目](#为什么需要这个项目)
- [快速开始](#快速开始)
- [Skills 一览](#skills-一览)
- [统一设计模式](#统一设计模式)
- [技术覆盖](#技术覆盖)
- [工作流示例](#工作流示例)
- [目录结构](#目录结构)
- [安装](#安装)
- [适用场景](#适用场景)
- [常见问题](#常见问题)
- [贡献](#贡献)
- [许可证](#许可证)

## 为什么需要这个项目

通用 AI 编程助手在 iOS 开发中常出现以下问题：

| 痛点 | 表现 |
|------|------|
| **缺少方法论** | 直接写代码，跳过需求分析和影响评估 |
| **不懂 iOS 规范** | 忽略内存管理、线程安全、Swift 6 并发要求 |
| **修复浮于表面** | 临时补丁而非根因定位 |
| **优化凭感觉** | 未量化就"优化"，缺少基线数据 |
| **重构不安全** | 边重构边加功能，缺少等价性验证 |

**ios-dev-skills** 通过 6 个结构化 Skills，让 Claude Code 像资深 iOS 工程师一样**有章法、有标准、有验证**地工作。

## 快速开始

**30 秒安装：**

```bash
git clone https://github.com/12207480/ios-dev-skills.git
cp -r ios-dev-skills/skills/ios-* ~/.claude/skills/
```

**立即使用 — 只需自然语言描述你的需求：**

```
你：实现一个带下拉刷新和分页加载的用户列表
你：审查这个 PR
你：用户反馈首页启动慢，优化一下
你：这个页面有个崩溃需要修复
你：把这个 ViewController 重构为 SwiftUI
你：给 UserViewModel 写单元测试
```

Claude Code 会自动加载对应 Skill，按五阶段方法论执行。

## Skills 一览

### 1. ios-feature-impl — 功能实现

> 从需求分析到代码提交的五阶段方法论

- 需求理解与分析（类型判断、设计稿审阅、API 审阅）
- 上下文分析（代码定位、设计意图理解）
- 关联分析（数据 / UI / 状态 / 导航 / 业务规则五维度）
- SwiftUI 现代模式 + UIKit 实现模式
- 网络层模式、持久化模式、测试策略

### 2. ios-code-review — 代码审查

> PR 审查的五阶段方法论，从理解意图到给出反馈

- Swift 安全性（可选值、错误处理、Swift 6 并发）
- Objective-C 安全性（nil / 集合 / Block / KVC-KVO / 混编）
- 内存与线程安全（循环引用、结构化并发）
- SwiftUI / UIKit 审查清单
- 无障碍与隐私审查、审查沟通技巧

### 3. ios-issue-fix — 问题修复

> 从读 Issue 到提 PR 的五阶段 Bug 修复方法论

- 7 大问题分类（UI / 逻辑 / 数据 / 交互 / 兼容 / 崩溃 / 性能）
- 分类型排查策略 + SwiftUI 专项排查
- 崩溃分析与符号化（dSYM、Crashlytics）
- 网络问题排查（HTTP 状态码、Codable 解析、SSL）
- Xcode 调试工具速查（LLDB / View Debugger / Instruments）

### 4. ios-performance-optimize — 性能优化

> 系统化的五阶段方法论：量化 → 定位 → 分析 → 优化 → 验证

- 启动优化（pre-main / post-main、二进制重排）
- 内存优化（OOM 分析、图片降采样、泄漏排查）
- 渲染优化（离屏渲染、列表滚动、SwiftUI 重绘）
- 包体积优化（资源瘦身、代码瘦身、App Thinning）
- Swift Concurrency 性能、数据库性能、Instruments 深度指南

### 5. ios-refactor — 代码重构

> 以"行为等价性"为核心的五阶段安全重构方法论

- 安全重构技法（提取 / 内联 / 移动 + Strangler Fig Pattern）
- ObjC → Swift 渐进式迁移
- UIKit → SwiftUI 渐进式迁移
- Swift Concurrency 迁移（completion handler → async/await、Swift 6）
- 架构重构（MVC → MVVM、模块化拆分、SPM Package）
- 遗留代码处理策略（安全区域划定、特征测试）

### 6. ios-testing — 测试编写

> 从测试策略设计到质量审查的五阶段测试方法论

- Swift Testing 深度指南（@Test / @Suite / #expect / 参数化 / Trait）
- Mock / Stub / Spy / Fake 模式（Protocol-based、URLProtocol）
- 单元测试设计（等价类 / 边界值 / 决策表 / 状态转换）
- 异步测试（Combine / AsyncSequence / Actor / Task 取消）
- UI 测试（XCUITest + Page Object）、快照测试、集成测试

<p align="right"><a href="#目录">⬆ 回到目录</a></p>

## 统一设计模式

所有 6 个 Skills 遵循相同的设计模式：

| 设计元素 | 说明 |
|---------|------|
| **五阶段流程** | 每个 Skill 都有清晰的五阶段工作流 |
| **复杂度速判** | S / M / L 三级分类，简单任务走精简流程 |
| **Topic Router** | 按话题路由到对应的 reference 文件，按需加载 |
| **门控输出** | 每个阶段有明确的"通过条件"，防止跳步 |
| **红旗信号表** | 列出常见反模式及正确做法 |
| **Checklist** | 每个 reference 文件都有可执行的检查清单 |

```
               ┌─────────────────────────────────┐
               │         用户输入需求              │
               └──────────┬──────────────────────┘
                          ▼
               ┌─────────────────────┐
               │   复杂度速判 S/M/L   │
               └──────────┬──────────┘
                          ▼
    ┌──────────┬──────────┬──────────┬──────────┐
    ▼          ▼          ▼          ▼          ▼
 阶段 1     阶段 2     阶段 3     阶段 4     阶段 5
  理解        分析        设计       执行        验证
    │          │          │          │          │
    ▼          ▼          ▼          ▼          ▼
 [门控]     [门控]      [门控]     [门控]     [门控]
  通过？      通过？      通过？     通过？      通过？
```

<p align="right"><a href="#目录">⬆ 回到目录</a></p>

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

<p align="right"><a href="#目录">⬆ 回到目录</a></p>

## 工作流示例

<table>
<tr>
<td width="50%">

**功能实现**

```
你：实现一个带分页加载的用户列表页面

Claude：
 → [加载 ios-feature-impl]
 → 阶段1: 分析需求类型与设计稿
 → 阶段2: 定位现有代码与模式
 → 阶段3: 评估数据/UI/状态影响
 → 阶段4: 按 SwiftUI 模式编码
 → 阶段5: 自审 + 测试策略
```

</td>
<td width="50%">

**代码审查**

```
你：审查这个 PR

Claude：
 → [加载 ios-code-review]
 → 阶段1: 理解 PR 意图与上下文
 → 阶段2: Swift/ObjC 安全性检查
 → 阶段3: 内存与线程安全检查
 → 阶段4: UI 框架专项审查
 → 阶段5: 输出结构化审查意见
```

</td>
</tr>
<tr>
<td width="50%">

**性能优化**

```
你：用户反馈列表页滑动卡顿

Claude：
 → [加载 ios-performance-optimize]
 → 阶段1: 建立性能基线（FPS/CPU）
 → 阶段2: Instruments 精准定位
 → 阶段3: 分析渲染瓶颈根因
 → 阶段4: 实施优化方案
 → 阶段5: 对比验证提升效果
```

</td>
<td width="50%">

**问题修复**

```
你：这个页面有个崩溃需要修复

Claude：
 → [加载 ios-issue-fix]
 → 阶段1: 分析崩溃日志与分类
 → 阶段2: 定位崩溃代码上下文
 → 阶段3: 根因定位与影响评估
 → 阶段4: 修复 + 先红后绿测试
 → 阶段5: 验证修复并提 PR
```

</td>
</tr>
<tr>
<td width="50%">

**代码重构**

```
你：把这个 ViewController 重构为 SwiftUI

Claude：
 → [加载 ios-refactor]
 → 阶段1: 评估重构范围与风险
 → 阶段2: 建立行为安全网（测试）
 → 阶段3: 设计迁移策略
 → 阶段4: 渐进式重构执行
 → 阶段5: 等价性验证
```

</td>
<td width="50%">

**测试编写**

```
你：给 UserViewModel 写单元测试

Claude：
 → [加载 ios-testing]
 → 阶段1: 分析被测代码行为边界
 → 阶段2: 设计测试策略与用例
 → 阶段3: 选择 Mock/Stub 模式
 → 阶段4: 编写高质量测试代码
 → 阶段5: 测试质量审查
```

</td>
</tr>
</table>

<p align="right"><a href="#目录">⬆ 回到目录</a></p>

## 目录结构

```
ios-dev-skills/
├── skills/
│   ├── ios-feature-impl/              # 功能实现（11 文件）
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
│   ├── ios-code-review/               # 代码审查（11 文件）
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
│   ├── ios-issue-fix/                 # 问题修复（10 文件）
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
│   ├── ios-performance-optimize/      # 性能优化（9 文件）
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
│   ├── ios-refactor/                  # 代码重构（8 文件）
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── safe-refactoring-techniques.md
│   │       ├── objc-to-swift-migration.md
│   │       ├── uikit-to-swiftui-migration.md
│   │       ├── concurrency-migration.md
│   │       ├── architecture-refactoring.md
│   │       ├── legacy-code-strategies.md
│   │       └── refactoring-verification.md
│   └── ios-testing/                   # 测试编写（9 文件）
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
├── LICENSE
└── README.md
```

<p align="right"><a href="#目录">⬆ 回到目录</a></p>

## 安装

### 方式一：全局安装（推荐）

```bash
# 克隆仓库
git clone https://github.com/12207480/ios-dev-skills.git

# 复制到 Claude Code skills 目录（对所有项目生效）
cp -r ios-dev-skills/skills/ios-* ~/.claude/skills/
```

### 方式二：只安装需要的 Skill

```bash
# 例如只安装代码审查
cp -r ios-dev-skills/skills/ios-code-review ~/.claude/skills/
```

### 方式三：项目级安装

```bash
# 复制到项目的 .claude/skills/ 目录下，仅对当前项目生效
cp -r ios-dev-skills/skills/ios-* your-project/.claude/skills/
```

> 安装后 Claude Code 会自动检测并加载这些 Skills，无需额外配置。

<p align="right"><a href="#目录">⬆ 回到目录</a></p>

## 适用场景

- 独立 iOS 开发者，希望 Claude Code 提供高级工程师级别的协助
- iOS 团队，希望统一开发规范和质量标准
- 处理 Swift / ObjC 混编的大型遗留项目
- 从 UIKit 向 SwiftUI 迁移的项目
- 需要系统化性能优化的项目

## 常见问题

<details>
<summary><b>Skills 安装后 Claude Code 没有自动加载？</b></summary>

确认文件已正确复制到 `~/.claude/skills/` 目录下，每个 Skill 目录中包含 `SKILL.md` 文件。重启 Claude Code 后即可生效。

</details>

<details>
<summary><b>可以只安装部分 Skills 吗？</b></summary>

可以。每个 Skill 完全独立，按需安装即可。推荐至少安装 `ios-feature-impl`（日常开发）和 `ios-code-review`（质量保障）。

</details>

<details>
<summary><b>支持哪些 iOS 版本？</b></summary>

Skills 中的模式和最佳实践覆盖 iOS 15+。SwiftUI 相关内容以 iOS 17+ 的 `@Observable` 为主，同时兼顾 `ObservableObject` 的写法。

</details>

<details>
<summary><b>是否支持纯 Objective-C 项目？</b></summary>

支持。`ios-code-review` 包含完整的 ObjC 安全性审查，`ios-refactor` 包含 ObjC → Swift 渐进式迁移指南。代码实现以 Swift 为主，ObjC 在混编场景中使用。

</details>

<details>
<summary><b>如何自定义或扩展 Skills？</b></summary>

每个 Skill 的 `SKILL.md` 定义主流程，`references/` 目录存放详细参考。你可以直接编辑这些 Markdown 文件来调整流程或添加团队特定规范。

</details>

<p align="right"><a href="#目录">⬆ 回到目录</a></p>

## 贡献

欢迎提交 Issue 和 Pull Request！贡献时请遵循：

1. 保持与现有 Skills 一致的设计模式（五阶段流程、Topic Router、门控输出等）
2. 所有内容使用中文
3. 代码示例使用 Swift，ObjC 仅在混编场景中使用
4. 每个 reference 文件必须包含总结 Checklist

## 许可证

[MIT](LICENSE) &copy; 2026
