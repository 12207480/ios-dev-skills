# 分类型排查策略

## UI 显示异常

### 布局错乱 / 元素位置不对
1. View Debugger（Cmd+Shift+D）查看实际布局层级
2. Console 搜索 "Unable to simultaneously satisfy constraints"
3. 检查约束优先级冲突
4. 检查 Safe Area / layoutMargins 在不同设备上的差异
5. 检查 `translatesAutoresizingMaskIntoConstraints`

### 文字截断 / 显示不全
1. 检查 UILabel 的 `numberOfLines` 和 `lineBreakMode`
2. 检查 SwiftUI Text 的 `lineLimit` 和 `truncationMode`
3. 检查容器的宽度约束是否允许文字展开
4. 检查 Dynamic Type 大字体下是否溢出

### 图片不显示 / 显示错误
1. 检查图片 URL 是否正确（打印/抓包确认）
2. 检查图片加载库的缓存策略（旧缓存？）
3. 检查 Asset Catalog 中图片名称是否匹配
4. 检查 @2x / @3x 资源是否齐全

### 颜色/样式不对
1. 检查 Dark Mode 适配（是否使用了硬编码颜色）
2. 检查颜色值与设计稿是否一致（注意色彩空间 sRGB vs Display P3）
3. 检查条件样式逻辑（选中/未选中、启用/禁用状态）

---

## 功能逻辑错误

### 按钮点击无反应 / 操作无效果
1. 断点确认 Action 方法是否被调用
2. **如果未调用：**
   - 检查 IBAction 连接是否断开
   - 检查 `isUserInteractionEnabled` 和 `isEnabled`
   - 检查是否有透明视图遮挡了点击区域（View Debugger 确认）
   - 检查手势冲突（GestureRecognizer 优先级）
3. **如果已调用但无效果：**
   - 在方法内部逐行断点，定位执行到哪一步中断
   - 检查条件判断是否走进了错误的分支
   - 检查异步操作是否正确回调

### 流程走不通 / 状态流转错误
1. 列出完整的状态流转路径
2. 在每个状态切换点添加日志
3. 比对实际日志与预期流转路径
4. 定位在哪一步偏离了预期
5. 检查边界条件：空数据/首次/末尾/最大值/最小值

### 计算结果不对
1. 准备一组已知输入→预期输出的测试数据
2. 在关键计算节点添加日志打印中间值
3. 逐步对比中间值与预期值
4. 定位首次出现偏差的计算步骤
5. 检查：类型转换精度丢失 / 整除取整 / 浮点比较 / 时区 / 本地化数字格式

---

## 数据问题

### 数据不展示
1. 抓包确认 API 是否返回了数据
2. 断点检查 Model 解析结果是否为 nil / 空数组
3. 检查 Codable 字段映射（CodingKeys / 可选值 / 日期格式）
4. 检查数据从 Service → ViewModel → View 的传递链路
5. 检查 UI 绑定是否正确（`@Published` / delegate / `reloadData`）

### 数据不更新 / 显示旧数据
1. 检查缓存策略：是否优先读取了本地缓存？
2. 检查 URLSession 的 `cachePolicy`
3. 检查 UI 刷新机制：`reloadData` / `setNeedsLayout` / `objectWillChange`
4. 检查 SwiftUI 的 `@State` / `@Binding` 是否正确触发刷新
5. 检查 `identifiable` id 是否唯一（相同 id 不触发 diff）

### 数据错乱 / 张冠李戴
1. **列表场景：** 检查 Cell 复用（`prepareForReuse` 是否重置）
2. **异步场景：** 请求发出后页面已切换，回调更新了错误的目标
3. **多线程场景：** 共享数据被并发修改
4. **缓存场景：** 缓存 Key 冲突导致读取了错误数据

---

## 交互不符合预期

### 导航方向/方式不对
1. 检查 push vs present 使用是否正确
2. 检查 `modalPresentationStyle` / `modalTransitionStyle`
3. 检查 NavigationStack / NavigationLink 配置
4. 检查返回时的数据回传逻辑（delegate / closure / Combine）

### 动画异常
1. 检查动画是否在主线程执行
2. 检查动画参数（duration / delay / curve）
3. 检查动画嵌套是否导致冲突
4. 检查 `layoutIfNeeded()` 的调用时机

### 滚动/手势异常
1. 检查 ScrollView 嵌套导致的手势冲突
2. 检查 UIGestureRecognizer 的 delegate 方法
3. 检查 `contentInset` / `contentOffset` 设置
4. 检查 keyboard avoidance 的实现方式

---

## 兼容性问题

### 特定 iOS 版本出现
1. 检查使用的 API 在目标版本上是否可用
2. 检查 `#available` 分支逻辑是否正确
3. 查阅 Apple Release Notes 确认是否有行为变更

### API 可用性排查
1. 在 Xcode 中将 Deployment Target 设为最低支持版本，编译检查警告
2. 全局搜索 `#available` / `@available`，逐一确认分支逻辑
3. 检查是否遗漏了 `if #available` 的 else 分支（降级方案）
4. 检查第三方库的最低版本要求是否与项目 Deployment Target 一致
5. 使用旧版本模拟器实际运行验证

### 特定设备出现
1. 检查 Safe Area（刘海屏 vs 非刘海屏）
2. 检查屏幕尺寸适配（SE 小屏 / Pro Max 大屏 / iPad）
3. 检查性能差异（低端机内存/CPU 不足）

### 深色模式问题
1. 检查是否使用了硬编码颜色（`UIColor(red:...)` / `Color(hex:...)`）
2. 检查 Asset Catalog 中颜色资源是否配置了 Dark Appearance
3. 检查 `traitCollectionDidChange` 是否正确响应
4. 检查图片资源是否提供了深色模式变体

### 无障碍问题

#### VoiceOver 问题
1. 打开 VoiceOver（设置 → 辅助功能 → VoiceOver）实际体验操作流程
2. 检查交互元素是否设置了 `accessibilityLabel`（UIKit）/ `.accessibilityLabel()`（SwiftUI）
3. 检查自定义控件的 `accessibilityTraits`（如按钮应有 `.button` trait）
4. 检查焦点顺序是否符合逻辑（`accessibilityElements` 排序 / `accessibilitySortPriority`）
5. 检查图片是否正确标记为装饰性（`isAccessibilityElement = false`）或提供了描述

#### Dynamic Type 问题
1. 检查文字是否使用了系统文字样式（`UIFont.preferredFont` / `.font(.body)`）
2. 在设置中将字体调至最大，检查 UI 是否溢出/截断/重叠
3. 检查固定高度的约束是否阻止了文字展开（使用 `>= height` 替代 `== height`）
4. 检查 ScrollView 是否在大字体下仍可滚动查看完整内容

#### Reduce Motion 问题
1. 检查是否响应了 `UIAccessibility.isReduceMotionEnabled`（UIKit）/ `@Environment(\.accessibilityReduceMotion)`（SwiftUI）
2. 开启 Reduce Motion 后检查是否有必要的动画替代方案（如淡入淡出替代滑动）
3. 检查自动播放的动画是否在 Reduce Motion 下停止

---

## 崩溃（按信号分类）

### EXC_BAD_ACCESS — 访问已释放内存
- 开启 Zombie Objects 排查
- 检查 delegate / closure 的引用关系
- 检查 Unmanaged / UnsafePointer 使用

### SIGABRT — 强制解包 nil / 断言失败
- 设置 Exception Breakpoint
- 检查可选值处理链路
- 检查 `fatalError` / `precondition` / `assert` 调用

### SIGKILL — 系统强杀
- 启动超时（> 20s）/ 内存超限 / 后台任务超时
- 检查 Watchdog 日志
- 检查 `beginBackgroundTask` 是否正确调用 `endBackgroundTask`

### EXC_BAD_INSTRUCTION — Swift 运行时错误
- enum 未匹配 / 数组越界 / 非法操作
- 检查 `switch` 是否覆盖所有 case
- 检查数组/字典的索引访问

> 详细的崩溃日志分析与符号化流程见 `crash-analysis.md`
