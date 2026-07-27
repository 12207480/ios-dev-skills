<div align="center">

# ios-dev-skills

**Turn Claude Code into your Senior iOS Engineer**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Swift](https://img.shields.io/badge/Swift-6.0-orange.svg)](https://swift.org)
[![Platform](https://img.shields.io/badge/Platform-iOS%2015+-blue.svg)](https://developer.apple.com/ios/)
[![Skills](https://img.shields.io/badge/Skills-6-green.svg)](#skills-overview)
[![Files](https://img.shields.io/badge/Files-58-lightgrey.svg)](#directory-structure)
[![Lines](https://img.shields.io/badge/Lines-15%2C500%2B-brightgreen.svg)](#technical-coverage)

6 Skills covering the full lifecycle of **Implementation · Review · Fix · Performance Optimization · Refactoring · Testing**

Supports Swift 6 / SwiftUI / UIKit / Objective-C mixed-language projects

</div>

---

## Table of Contents

- [Why this project?](#why-this-project)
- [Quick Start](#quick-start)
- [Skills Overview](#skills-overview)
- [Unified Design Pattern](#unified-design-pattern)
- [Technical Coverage](#technical-coverage)
- [Workflow Examples](#workflow-examples)
- [Directory Structure](#directory-structure)
- [Installation](#installation)
- [Applicable Scenarios](#applicable-scenarios)
- [FAQ](#faq)
- [Contribution](#contribution)
- [License](#license)

## Why this project?

General AI coding assistants often encounter the following issues in iOS development:

| Pain Point | Manifestation |
|------|------|
| **Lack of Methodology** | Writes code directly, skipping requirement analysis and impact assessment |
| **Unaware of iOS Specs** | Ignores memory management, thread safety, and Swift 6 concurrency requirements |
| **Superficial Fixes** | Applies temporary patches instead of locating the root cause |
| **Intuitive Optimization** | "Optimizes" without quantification or baseline data |
| **Unsafe Refactoring** | Adds features while refactoring, lacking equivalence verification |

**ios-dev-skills** provides 6 structured Skills to ensure Claude Code works with **discipline, standards, and verification**, just like a senior iOS engineer.

## Quick Start

**30-second installation:**

```bash
git clone https://github.com/12207480/ios-dev-skills.git
cp -r ios-dev-skills/skills/ios-* ~/.claude/skills/
```

**Use immediately — just describe your needs in natural language:**

```
You: Implement a user list with pull-to-refresh and pagination
You: Review this PR
You: Users reported the home screen starts slowly, please optimize it
You: There is a crash on this page that needs to be fixed
You: Refactor this ViewController to SwiftUI
You: Write unit tests for UserViewModel
```

Claude Code will automatically load the corresponding Skill and execute it according to the five-stage methodology.

## Skills Overview

### 1. ios-feature-impl — Feature Implementation

> A five-stage methodology from requirement analysis to code commit

- Requirement understanding and analysis (Type determination, design review, API review)
- Context analysis (Code localization, design intent understanding)
- Correlation analysis (Five dimensions: Data / UI / State / Navigation / Business Rules)
- Modern SwiftUI patterns + UIKit implementation patterns
- Networking patterns, persistence patterns, and testing strategies

### 2. ios-code-review — Code Review

> A five-stage methodology for PR reviews, from understanding intent to providing feedback

- Swift Safety (Optionals, error handling, Swift 6 concurrency)
- Objective-C Safety (nil / Collections / Blocks / KVC-KVO / Interop)
- Memory & Thread Safety (Strong reference cycles, structured concurrency)
- SwiftUI / UIKit review checklists
- Accessibility & Privacy reviews, review communication techniques

### 3. ios-issue-fix — Issue Fixing

> A five-stage bug-fixing methodology from reading an Issue to submitting a PR

- 7 major issue classifications (UI / Logic / Data / Interaction / Compatibility / Crash / Performance)
- Classification-based troubleshooting + SwiftUI specific debugging
- Crash analysis and symbolication (dSYM, Crashlytics)
- Network troubleshooting (HTTP status codes, Codable parsing, SSL)
- Xcode debugging tool quick-ref (LLDB / View Debugger / Instruments)

### 4. ios-performance-optimize — Performance Optimization

> A systematic five-stage methodology: Quantify → Locate → Analyze → Optimize → Verify

- Launch optimization (pre-main / post-main, binary reordering)
- Memory optimization (OOM analysis, image downsampling, leak detection)
- Rendering optimization (Off-screen rendering, list scrolling, SwiftUI redraws)
- App size optimization (Resource slimming, code slimming, App Thinning)
- Swift Concurrency performance, database performance, and deep Instruments guide

### 5. ios-refactor — Code Refactoring

> A five-stage safe refactoring methodology centered on "Behavioral Equivalence"

- Safe refactoring techniques (Extract / Inline / Move + Strangler Fig Pattern)
- Progressive migration from ObjC → Swift
- Progressive migration from UIKit → SwiftUI
- Swift Concurrency migration (completion handler → async/await, Swift 6)
- Architecture refactoring (MVC → MVVM, modularization, SPM Package)
- Legacy code strategies (Safety zones, characterization tests)

### 6. ios-testing — Test Writing

> A five-stage testing methodology from test strategy design to quality review

- Deep guide to Swift Testing (@Test / @Suite / #expect / Parameterization / Trait)
- Mock / Stub / Spy / Fake patterns (Protocol-based, URLProtocol)
- Unit test design (Equivalence partitioning / Boundary values / Decision tables / State transition)
- Asynchronous testing (Combine / AsyncSequence / Actor / Task cancellation)
- UI Testing (XCUITest + Page Object), Snapshot testing, Integration testing

<p align="right"><a href="#table-of-contents">⬆ Back to Top</a></p>

## Unified Design Pattern

All 6 Skills follow the same design pattern:

| Design Element | Description |
|---------|------|
| **Five-Stage Process** | Each Skill has a clear five-stage workflow |
| **Complexity Assessment** | S / M / L classification; simple tasks follow a streamlined process |
| **Topic Router** | Routes to corresponding reference files based on topic, loading on demand |
| **Gated Output** | Each stage has clear "Pass Conditions" to prevent skipping steps |
| **Red Flag Table** | Lists common anti-patterns and the correct approach |
| **Checklist** | Every reference file contains an executable checklist |

```
               ┌─────────────────────────────────┐
               │         User Input Request       │
               └──────────┬──────────────────────┘
                          ▼
               ┌─────────────────────┐
               │ Complexity Check S/M/L │
               └──────────┬──────────┘
                          ▼
    ┌──────────┬──────────┬──────────┬──────────┐
    ▼          ▼          ▼          ▼          ▼
 Stage 1    Stage 2    Stage 3    Stage 4    Stage 5
 Understand  Analyze    Design    Execute    Verify
    │          │          │          │          │
    ▼          ▼          ▼          ▼          ▼
 [Gate]     [Gate]      [Gate]     [Gate]     [Gate]
  Pass?      Pass?       Pass?      Pass?      Pass?
```

<p align="right"><a href="#table-of-contents">⬆ Back to Top</a></p>

## Technical Coverage

| Domain | Coverage |
|------|---------|
| **Languages** | Swift 6, Objective-C |
| **UI Frameworks** | SwiftUI (iOS 17+ @Observable), UIKit, Mixed-language |
| **Concurrency** | async/await, Actor, Sendable, structured concurrency |
| **Architecture** | MVVM, MVC, Coordinator, SPM modularization |
| **Persistence** | Core Data, SwiftData, UserDefaults, Keychain |
| **Network** | URLSession, Codable, Pagination, SSL Pinning |
| **Testing** | Swift Testing, XCTest, XCUITest, swift-snapshot-testing |
| **Tools** | Instruments, LLDB, View Debugger, Memory Graph |
| **Platform** | iOS 15+, Widget, App Extension |

<p align="right"><a href="#table-of-contents">⬆ Back to Top</a></p>

## Workflow Examples

<table>
<tr>
<td width="50%">

**Feature Implementation**

```
You: Implement a user list page with pagination

Claude:
 → [Load ios-feature-impl]
 → Stage 1: Analyze requirement type and designs
 → Stage 2: Locate existing code and patterns
 → Stage 3: Assess data/UI/state impact
 → Stage 4: Code according to SwiftUI patterns
 → Stage 5: Self-review + Testing strategy
```

</td>
<td width="50%">

**Code Review**

```
You: Review this PR

Claude:
 → [Load ios-code-review]
 → Stage 1: Understand PR intent and context
 → Stage 2: Swift/ObjC safety check
 → Stage 3: Memory and thread safety check
 → Stage 4: UI framework specialized review
 → Stage 5: Output structured review feedback
```

</td>
</tr>
<tr>
<td width="50%">

**Performance Optimization**

```
You: Users reported the list page stutters during scrolling

Claude:
 → [Load ios-performance-optimize]
 → Stage 1: Establish performance baseline (FPS/CPU)
 → Stage 2: Precise localization using Instruments
 → Stage 3: Analyze root cause of rendering bottleneck
 → Stage 4: Implement optimization plan
 → Stage 5: Verify and compare improvement
```

</td>
<td width="50%">

**Issue Fixing**

```
You: There is a crash on this page that needs to be fixed

Claude:
 → [Load ios-issue-fix]
 → Stage 1: Analyze crash logs and classify
 → Stage 2: Locate crash code context
 → Stage 3: Root cause localization and impact assessment
 → Stage 4: Fix + Red-to-Green testing
 → Stage 5: Verify fix and submit PR
```

</td>
</tr>
<tr>
<td width="50%">

**Code Refactoring**

```
You: Refactor this ViewController to SwiftUI

Claude:
 → [Load ios-refactor]
 → Stage 1: Assess refactoring scope and risk
 → Stage 2: Establish behavioral safety net (tests)
 → Stage 3: Design migration strategy
 → Stage 4: Progressive refactoring execution
 → Stage 5: Equivalence verification
```

</td>
<td width="50%">

**Test Writing**

```
You: Write unit tests for UserViewModel

Claude:
 → [Load ios-testing]
 → Stage 1: Analyze behavior boundaries of code under test
 → Stage 2: Design test strategy and cases
 → Stage 3: Select Mock/Stub patterns
 → Stage 4: Write high-quality test code
 → Stage 5: Test quality review
```

</td>
</tr>
</table>

<p align="right"><a href="#table-of-contents">⬆ Back to Top</a></p>

## Directory Structure

```
ios-dev-skills/
├── skills/
│   ├── ios-feature-impl/              # Feature Implementation (11 files)
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
│   ├── ios-code-review/               # Code Review (11 files)
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
│   ├── ios-issue-fix/                 # Issue Fixing (10 files)
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
│   ├── ios-performance-optimize/      # Performance Optimization (9 files)
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
│   ├── ios-refactor/                  # Code Refactoring (8 files)
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── safe-refactoring-techniques.md
│   │       ├── objc-to-swift-migration.md
│   │       ├── uikit-to-swiftui-migration.md
│   │       ├── concurrency-migration.md
│   │       ├── architecture-refactoring.md
│   │       ├── legacy-code-strategies.md
│   │       └── refactoring-verification.md
│   └── ios-testing/                   # Test Writing (9 files)
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

<p align="right"><a href="#table-of-contents">⬆ Back to Top</a></p>

## Installation

### Option 1: Global Installation (Recommended)

```bash
# Clone repository
git clone https://github.com/12207480/ios-dev-skills.git

# Copy to Claude Code skills directory (affects all projects)
cp -r ios-dev-skills/skills/ios-* ~/.claude/skills/
```

### Option 2: Install Specific Skills Only

```bash
# For example, install only Code Review
cp -r ios-dev-skills/skills/ios-code-review ~/.claude/skills/
```

### Option 3: Project-level Installation

```bash
# Copy to project's .claude/skills/ directory, affects only the current project
cp -r ios-dev-skills/skills/ios-* your-project/.claude/skills/
```

> After installation, Claude Code will automatically detect and load these Skills without further configuration.

<p align="right"><a href="#table-of-contents">⬆ Back to Top</a></p>

## Applicable Scenarios

- Independent iOS developers who want Claude Code to provide senior-level assistance.
- iOS teams looking to unify development standards and quality benchmarks.
- Large legacy projects with a mix of Swift and Objective-C.
- Projects migrating from UIKit to SwiftUI.
- Projects requiring systematic performance optimization.

## FAQ

<details>
<summary><b>Claude Code didn't automatically load the Skills after installation?</b></summary>

Confirm that the files were correctly copied to the `~/.claude/skills/` directory and that each skill directory contains a `SKILL.md` file. Restart Claude Code for changes to take effect.

</details>

<details>
<summary><b>Can I install only some of the Skills?</b></summary>

Yes. Each Skill is completely independent. It is recommended to at least install `ios-feature-impl` (for daily development) and `ios-code-review` (for quality assurance).

</details>

<details>
<summary><b>Which iOS versions are supported?</b></summary>

The patterns and best practices in these Skills cover iOS 15+. SwiftUI content primarily focuses on the `@Observable` pattern (iOS 17+), while maintaining compatibility with `ObservableObject`.

</details>

<details>
<summary><b>Does it support pure Objective-C projects?</b></summary>

Yes. `ios-code-review` includes a comprehensive ObjC safety review, and `ios-refactor` includes a guide for progressive ObjC → Swift migration. While implementations primarily use Swift, ObjC is used for mixed-language scenarios.

</details>

<details>
<summary><b>How can I customize or extend the Skills?</b></summary>

The `SKILL.md` of each Skill defines the main process, and the `references/` directory holds detailed references. You can edit these Markdown files directly to adjust processes or add team-specific standards.

</details>

<p align="right"><a href="#table-of-contents">⬆ Back to Top</a></p>

## Contribution

Issues and Pull Requests are welcome! When contributing, please follow these guidelines:

1. Maintain the design pattern consistent with existing Skills (Five-stage process, Topic Router, Gated output, etc.).
2. All content should be in English.
3. Use Swift for code examples; use ObjC only in mixed-language scenarios.
4. Every reference file must include a summary Checklist.

## License

[MIT](LICENSE) &copy; 2026
