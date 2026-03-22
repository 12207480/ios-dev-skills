# UIKit 审查

> UIKit 特有的审查要点。涵盖 Auto Layout、列表视图、ViewController 生命周期、混编场景。

## Auto Layout 审查

```
约束完整性：
- [ ] translatesAutoresizingMaskIntoConstraints = false 是否设置？
      → 使用代码布局时必须设置，否则自动约束冲突
      → Storyboard/XIB 中系统自动处理，无需手动设置
- [ ] 约束是否完整定义了视图的 x/y/width/height？
      → 缺少约束 → 视图位置或大小不确定（Ambiguous Layout）
      → 使用 Xcode 的 Debug View Hierarchy 检查
- [ ] 约束是否冲突？
      → 控制台出现 "Unable to simultaneously satisfy constraints"
      → 设置 priority < 1000 解决可打破约束
- [ ] 使用 Safe Area 而非硬编码状态栏/底部高度？
      → view.safeAreaLayoutGuide ✅
      → 硬编码 44pt / 34pt ❌

性能与最佳实践：
- [ ] 避免在 updateConstraints() 中频繁创建/删除约束
      → 预创建约束，通过 isActive 切换
      → 或使用 NSLayoutConstraint.activate / deactivate 批量操作
- [ ] 复杂布局是否使用 UIStackView 简化？
      → StackView 自动管理子视图约束
      → 减少手动约束数量，降低出错概率
- [ ] intrinsicContentSize 是否正确实现？
      → 自定义视图需要覆写以支持自适应大小
      → invalidateIntrinsicContentSize() 在内容变化时调用
- [ ] compressionResistance / huggingPriority 是否合理？
      → 多个 Label 水平排列时，谁先被压缩？
      → 通常内容重要的设高 resistance，可拉伸的设低 hugging
```

## UITableView / UICollectionView 审查

```
Cell 复用：
- [ ] prepareForReuse() 是否重置所有状态？
      → 图片、文字、选中状态、背景色等必须重置
      → 未重置 → 滚动时出现"残影"或错误状态
- [ ] dequeueReusableCell 是否使用注册机制？
      → register(UINib/Class, forCellReuseIdentifier:) ✅
      → 手动 if cell == nil 创建 ❌（过时写法）
- [ ] Cell 中的异步操作（网络图片加载）是否处理了复用？
      → 在 prepareForReuse 中取消未完成的请求
      → 回调中验证 cell 是否仍显示对应数据

性能：
- [ ] cellForRowAt 中是否有耗时操作？
      → 数据格式化、图片处理应在 Model 层预处理
      → 避免在 cellForRowAt 中创建 DateFormatter
- [ ] Cell 高度计算是否高效？
      → 优先使用 UITableView.automaticDimension + 正确约束
      → 复杂 Cell 可缓存计算后的高度
- [ ] 使用 UICollectionViewCompositionalLayout 替代 UICollectionViewFlowLayout？
      → 复杂布局（瀑布流、多 Section 不同布局）优先使用 Compositional Layout
      → 避免在 delegate 方法中计算布局

Diffable DataSource（推荐）：
- [ ] 新代码是否使用 UITableViewDiffableDataSource？
      → 替代传统 delegate 数据源方法
      → 自动 diff + 动画，避免 reloadData / performBatchUpdates 崩溃
- [ ] NSDiffableDataSourceSnapshot 的 apply 是否在主线程？
      → apply(snapshot, animatingDifferences:) 默认主线程
      → 后台线程 apply 需注意线程安全
```

## UIViewController 生命周期审查

```
生命周期顺序：
  init → loadView → viewDidLoad → viewWillAppear → viewDidAppear
  → viewWillDisappear → viewDidDisappear → deinit

- [ ] 初始化逻辑是否在 viewDidLoad 中？（不是 init）
      → viewDidLoad 保证视图层级已加载
      → init 中 self.view 会触发提前加载（loadView）
- [ ] viewWillAppear 中的操作是否幂等？
      → 可能被多次调用（返回时再次触发）
      → 不要在此进行一次性操作
- [ ] viewDidDisappear 中是否停止了不必要的操作？
      → 停止 Timer、暂停动画、取消网络请求
- [ ] deinit 是否被正确调用？
      → 添加 print("deinit") 验证是否释放
      → 不释放 → 检查循环引用（闭包、delegate、NotificationCenter）

容器 VC：
- [ ] addChild / removeFromParent 是否配对使用？
      → addChild → didMove(toParent: self)
      → willMove(toParent: nil) → removeFromParent
- [ ] 子 VC 的生命周期方法是否正确转发？
      → 系统默认转发，自定义容器需确认

Present / Dismiss：
- [ ] present 的 VC 是否正确处理了 dismiss？
      → 避免在 presented VC 上再 present（需先 dismiss）
- [ ] modalPresentationStyle 是否符合预期？
      → iOS 13+ 默认 .automatic（非全屏）
      → 需要全屏时显式设置 .fullScreen
```

## 混编审查（Swift + Objective-C）

```
桥接文件：
- [ ] Bridging Header 是否只包含必要的头文件？
      → 过多头文件增加编译时间
      → 使用 @import Module; 替代 #import（在可能时）
- [ ] Generated Interface Header（ProductName-Swift.h）是否正确暴露？
      → Swift 类必须继承 NSObject 或标注 @objc 才能暴露给 OC
      → @objcMembers 暴露所有成员 vs @objc 精确控制

类型桥接：
- [ ] OC 的 nullability 标注是否完整？
      → 未标注 → Swift 侧为隐式解包可选值（!），有崩溃风险
      → 使用 NS_ASSUME_NONNULL_BEGIN/END + 逐个标注 nullable
- [ ] OC 枚举使用 NS_ENUM / NS_OPTIONS？
      → 普通 C 枚举 → Swift 无法正确桥接
- [ ] OC 泛型是否标注？
      → NSArray<NSString *> → Swift 得到 [String]
      → NSArray → Swift 得到 [Any]，需要强转

Swift → OC 调用注意：
- [ ] Swift 的 String/Array/Dictionary 在 OC 侧是否自动桥接？
      → Swift String → NSString（自动）
      → Swift 值类型（struct/enum）无法暴露给 OC
- [ ] @objc 方法的参数/返回值是否为 OC 兼容类型？
      → 元组、泛型、Swift 独有类型不可暴露
- [ ] Swift 的错误处理在 OC 侧是否变为 NSError **？
      → throws → (NSError **) 参数
      → OC 调用方需要检查 error 参数
```

## 常见问题速查

### 看到 `translatesAutoresizingMaskIntoConstraints` 未设置

```
判断场景：
  代码创建的视图 → 必须设置为 false
  XIB/Storyboard 中的视图 → 系统自动处理，不需要
  UIStackView 的 arrangedSubviews → 不需要设置

建议 → 🔴 Must Fix（不设置会导致约束冲突，布局错乱）
```

### 看到 `reloadData()` 在数据局部变化时

```
判断影响：
  数据全量替换 → reloadData() 合理
  数据局部变化（增/删/改少量行） → 应使用 performBatchUpdates 或 Diffable DataSource

建议 → 🟡 Should Fix
  → 使用 DiffableDataSource + apply(snapshot) 替代
  → 或至少使用 insertRows/deleteRows/reloadRows
```

### 看到 Cell 中直接网络请求

```
判断 Y → 违反职责分离，Cell 不应直接持有网络逻辑
建议 Z → 网络请求应在 ViewModel/Controller 中发起，Cell 只负责展示
  → 图片加载可使用 SDWebImage/Kingfisher 等库或自建图片加载器
  → Cell 中只保留 configure(with:) 接收预处理好的数据
```

### 看到 `viewDidLoad` 中大量初始化代码

```
判断 Y → viewDidLoad 职责过重，难以维护和测试
建议 Z → 拆分为独立方法
  → setupUI()：视图层级和布局
  → setupBindings()：数据绑定
  → loadData()：初始数据加载
  → 每个方法职责单一，便于子类覆写
```

### 看到 `UIColor` 硬编码

```
判断 Y → 不支持 Dark Mode，不同主题下显示不正确
建议 Z →
  → 使用语义化颜色：.label / .systemBackground / .secondaryLabel
  → 自定义颜色使用 Asset Catalog 的 Color Set（支持 Light/Dark）
  → UIColor { traitCollection in ... } 动态适配
```
