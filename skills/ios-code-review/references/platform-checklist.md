# 平台规范与检查清单

> 核心平台规范检查。深入的无障碍和隐私审查见 `accessibility-privacy.md`，测试审查见 `testing-review.md`，UIKit 专项审查见 `uikit-review.md`。

## 生命周期

```
UIViewController：
- [ ] 生命周期使用是否正确？
      → 初始化逻辑在 viewDidLoad（不是 init）
      → 布局更新在 viewDidLayoutSubviews
      → 出现/消失逻辑在 viewWillAppear / viewDidDisappear

SwiftUI：
- [ ] 生命周期 modifier 使用是否正确？
      → .onAppear / .onDisappear / .task / .onChange
      → .task 自动管理取消，优于 .onAppear + Task {}
```

## UI 规范（精简版）

```
基础规范：
- [ ] 使用语义化颜色？（.label / .systemBackground，非硬编码）
- [ ] 适配 Safe Area？（非硬编码状态栏/底部高度）
- [ ] 触控区域 ≥ 44x44pt？
- [ ] 支持 Dynamic Type？（preferredFont / .font(.body)）
- [ ] 适配 Dark Mode？
- [ ] 图片加载是否异步且带缓存？

→ Auto Layout 详细审查见 uikit-review.md
→ Cell 复用详细审查见 uikit-review.md
→ 无障碍详细审查见 accessibility-privacy.md
```

## API 兼容性

```
- [ ] 使用的 API 在最低支持版本上是否可用？
- [ ] #available 检查是否正确？
      → if #available(iOS 17, *) { ... } else { ... }
      → else 分支的 fallback 是否功能完整？
- [ ] deprecated API 是否有替代方案？
      → 详细替代方案见 modern-api.md
- [ ] 条件编译（#if canImport）是否正确使用？
      → 用于可选框架依赖（如 WidgetKit）
```

## 架构一致性审查

```
项目约定遵循：
- [ ] 架构模式是否一致？（不要在 MVC 项目中突然引入 MVVM）
- [ ] 异步模式是否一致？（不要在 Combine 项目中突然用 async/await）
- [ ] 错误处理是否一致？（不要在 Result 项目中突然 throw）
- [ ] 导航方式是否一致？（Coordinator / 直接跳转 / Router）
- [ ] 命名规范是否一致？

职责划分：
- [ ] View 层是否只负责展示？（不含业务逻辑/网络请求）
- [ ] ViewModel 是否不依赖 UIKit/SwiftUI 类型？
- [ ] Service 层职责是否单一？
- [ ] 各层之间是否通过协议/接口解耦？

代码组织：
- [ ] 文件是否放在正确的目录下？
- [ ] 新建的文件是否遵循项目的命名规范？
- [ ] 是否重复实现了已有的功能？（项目中是否有可复用的组件）
- [ ] 公共代码是否放到了正确的共享位置？

Extension / Widget / App Clip 检查：
- [ ] App Extension 的内存限制是否考虑？（通常 ≤ 120MB）
- [ ] Extension 与主 App 的数据共享是否通过 App Group？
- [ ] Widget 的 Timeline 刷新策略是否合理？
- [ ] Extension 中是否误用了主 App 专属的 API？
```
