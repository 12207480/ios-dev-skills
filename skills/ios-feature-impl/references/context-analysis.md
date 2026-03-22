# 上下文分析

## 定位相关代码

根据需求类型，使用不同策略：

### 全新功能

1. 找到项目中相似功能的模块，参考其目录结构和架构模式
2. 确认项目的架构模式（MVC / MVVM / VIPER / TCA / Clean）
3. 确认项目的技术选型（UIKit / SwiftUI / 混编）
4. 确认项目的依赖管理方式和已有的第三方库

### 增量需求（最常见）

1. 从 UI 入口定位到对应的 View / ViewController
2. 找到对应的 ViewModel / Presenter / Interactor
3. 找到对应的 Service / API 调用
4. 找到对应的 Model 定义
5. 全局搜索相关的 class / protocol / enum，了解完整的依赖图

### 技术栈判断与参考

定位到相关代码后，根据技术栈选择对应的参考文件：

| 技术栈 | 参考文件 | 重点关注 |
|-------|---------|---------|
| UIKit | `uikit-patterns.md` | Auto Layout、DiffableDataSource、VC 生命周期、Coordinator |
| SwiftUI | `swiftui-patterns.md` | @Observable、NavigationStack、性能优化 |
| 混编（UIKit + SwiftUI） | 两者都参考 | UIHostingController、UIViewRepresentable、导航衔接、样式一致性 |

## 理解现有代码的设计意图

**改代码之前，必须先理解它为什么这样写。**

### View 层
- 为什么用这种布局方式？（Auto Layout / StackView / SwiftUI）
- 状态变体怎么处理的？（enum switch / if-else / 隐藏显示）
- 数据怎么绑定到 UI 的？（delegate / closure / Combine / @Published / @Observable）
- 有没有复用/公共组件？

**UIKit 视角补充：**
- ViewController 的职责是否清晰？（是否承担了过多逻辑）
- Cell 复用是否正确实现？（`prepareForReuse` 是否重置状态）
- 自定义 View 是用代码还是 XIB/Storyboard？项目的统一方式是什么？
- 生命周期管理：`viewDidLoad` / `viewWillAppear` / `viewDidAppear` 中分别做了什么？

### ViewModel / 逻辑层
- 状态管理用的什么模式？（@Published / delegate / closure）
- 业务规则是怎么实现的？（校验、转换、计算）
- 错误处理策略是什么？
- 依赖注入方式是什么？（init 参数 / protocol）

### 数据层
- 网络请求怎么组织的？（集中式 Service / 分散式）
- 数据模型怎么设计的？（Response 模型 vs 业务模型 分不分离）
- 本地存储用了什么？（UserDefaults / Core Data / 文件 / Keychain）
- 缓存策略是什么？

### 通用模式
- 项目的错误处理约定是什么？（Result / throw / 回调）
- 项目的异步模式是什么？（async/await / Combine / GCD / delegate）
- 项目的导航方式是什么？（Coordinator / 直接跳转 / Router）
- 项目的命名规范是什么？

## 阅读相关的 Git 历史

理解代码的演变：

1. `git log -10 --oneline -- <相关文件>` — 最近改动
2. `git log --author=<谁写的> -- <文件>` — 原作者的其他改动
3. 查看相关 PR 的描述和 Review 评论 — 理解设计决策
4. 查看相关的技术方案文档（如有） — 理解架构意图

## 阅读现有测试

检查已有的测试覆盖：

1. 找到相关模块的 Tests 文件
2. 阅读测试用例 → 理解预期行为的边界
3. 哪些场景有测试覆盖？哪些没有？
4. 新增/修改功能是否会导致现有测试失败？
   - 如果会，是因为行为确实变了（需要更新测试），还是因为破坏了不该变的逻辑（需要重新审视方案）？

## 输出模板：上下文分析摘要

```
项目上下文摘要：

架构模式：[MVC / MVVM / VIPER / TCA / Clean / 混合]
技术栈：[UIKit / SwiftUI / 混编，异步模式，依赖管理]

相关文件清单：
  [文件路径] — [职责] — [需要修改 / 仅需理解]
  ...

现有设计意图：
  [用自己的话概括现有代码为什么这样设计]

扩展点 / 修改点：
  [在哪里新增代码？在哪里修改代码？]

需要遵循的项目约定：
  [命名风格 / 错误处理方式 / 异步模式 / ...]
```

**关键原则：** 如果写不出"现有设计意图"这一项，说明代码还没读懂，不要开始下一步。
