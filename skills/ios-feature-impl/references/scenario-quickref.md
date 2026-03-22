# 常见场景速查

## 在已有列表页新增功能

关注点：
- Cell 是否需要新增 UI 元素？影响已有 Cell 布局？
- 数据模型是否需要新增字段？影响其他用到该模型的页面？
- 新增的交互（如滑动删除）是否与已有手势冲突？
- 列表的筛选/排序/搜索条件是否需要调整？
- DiffableDataSource / LazyVStack 的数据源是否需要调整？

**UIKit 关注点：**
- Cell 使用 DiffableDataSource 还是传统 DataSource？保持与现有代码一致
- 自定义 Cell 的 `prepareForReuse()` 是否需要重置新增的 UI 元素
- Compositional Layout 的 Section 配置是否需要调整
- `UITableView` 的 self-sizing cell 约束链是否完整
- 预估行高（`estimatedRowHeight`）设置是否合理

## 在已有表单页新增字段

关注点：
- 新字段插入位置是否影响现有布局？
- 校验规则是否需要更新？
- 提交接口的参数是否需要变更？影响后端？
- 草稿保存/恢复逻辑是否需要适配？
- 键盘遮挡问题是否因新字段位置而变化？

**UIKit 关注点：**
- 静态表单用 `UITableView` static cells 还是动态构建？保持一致
- 表单验证逻辑是否集中管理（ViewModel 统一校验 vs Cell 各自校验）
- `UITextField` / `UITextView` 的 delegate 链是否正确设置
- 键盘工具栏（上一个/下一个/完成）的 tag 顺序是否连续
- `IQKeyboardManager` 等第三方库的兼容性

## 在已有详情页增加操作按钮

关注点：
- 按钮放在哪？Navigation Bar / Toolbar / 页面内？
- 操作成功后，返回列表页需要刷新吗？
- 操作需要确认弹窗吗？
- 操作有权限限制吗？无权限时按钮隐藏还是置灰？
- 操作的 loading/成功/失败 状态如何展示？

## 新增一个独立页面

关注点：
- 入口在哪？从哪些页面可以进入？
- 导航方式和返回方式
- 是否需要接收参数？参数从哪来？
- 操作后是否需要向上一个页面回传结果？
- 是否需要支持 Deep Link / 推送跳转？
- 页面的目录结构和命名是否与项目一致？

## 适配 API 变更

关注点：
- 新旧字段的映射关系
- 旧客户端（未升级用户）是否仍能正常工作？
- Codable 模型修改后，影响多少个页面？
- 本地缓存的旧数据是否需要清理或迁移？
- 是否需要按 API 版本做兼容处理？

## 需求变更处理

### 变更评估流程

1. **确认变更内容** — 新增 / 修改 / 删除？
2. **影响分析** — 已实现的代码哪些受影响？
3. **关联分析** — 变更是否波及其他功能？
4. **工时评估** — 需要多少额外时间？
5. **与产品/PM 同步** — 确认排期调整
6. **更新实现方案** — 记录变更历史

### 变更分级

| 级别 | 描述 | 处理方式 |
|------|------|---------|
| S 级 | 核心流程重设计 | 回到阶段二重新分析 |
| A 级 | 新增/删除核心交互 | 补充关联分析 |
| B 级 | 调整文案/颜色/间距 | 直接修改 |
| C 级 | 修复设计细节 | 直接修改 |

## Widget 开发

关注点：
- Widget 类型选择：`StaticConfiguration`（固定内容）vs `IntentConfiguration`（用户可配置）vs `AppIntentConfiguration`（iOS 17+）
- Timeline 刷新策略：`atEnd`（时间线结束后刷新）/ `after(date)`（指定时间刷新）/ `never`（仅手动刷新）
- 数据共享：通过 App Group 的 `UserDefaults(suiteName:)` 或共享文件与主 App 共享数据
- Widget 支持的尺寸：`systemSmall` / `systemMedium` / `systemLarge` / `systemExtraLarge`（iPad）
- 深度链接：Widget 点击跳转到主 App 的特定页面（`widgetURL` / `Link`）
- 性能约束：Widget 的 Timeline 更新频率受系统限制，不要依赖实时更新
- 预览调试：使用 `#Preview` 宏预览不同尺寸和状态

```swift
// Widget 基本结构
struct MyWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(
            kind: "MyWidget",
            provider: MyTimelineProvider()
        ) { entry in
            MyWidgetView(entry: entry)
                .containerBackground(.fill, for: .widget)
        }
        .configurationDisplayName("我的小组件")
        .description("显示最新数据")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}
```

## App Extension 开发

关注点：
- Extension 类型确认：Share Extension / Today Extension / Notification Content / Action Extension / 等
- 与主 App 的代码共享：通过 Framework 或 Swift Package 共享业务逻辑
- 数据共享：App Group 是 Extension 与主 App 共享数据的唯一方式
- 内存限制：Extension 的内存限制远低于主 App（通常约 120 MB），注意内存占用
- 生命周期：Extension 随时可能被系统终止，不要依赖长时间运行
- Bundle ID：Extension 的 Bundle ID 必须以主 App 的 Bundle ID 为前缀
- 权限继承：Extension 的 Entitlements 需要单独配置，不继承主 App
- 调试：Xcode 中选择 Extension Target 运行，附加到宿主 App 调试

```
// 项目结构示例
MyApp/
├── MyApp/                    # 主 App Target
├── MyAppExtension/           # Extension Target
├── SharedKit/                # 共享 Framework / Package
│   ├── Models/               # 共享数据模型
│   ├── Services/             # 共享业务逻辑
│   └── Utilities/            # 共享工具类
```

**Extension 常见陷阱：**
- 忘记在 Extension Target 中添加 App Group capability
- 共享的 UserDefaults 使用了 `.standard` 而非 `UserDefaults(suiteName:)`
- Extension 中使用了主 App 才有权限的 API（如后台定位）
- 共享 Framework 引入了不必要的依赖导致 Extension 包体积过大
