# 渲染优化

## 渲染管线基础

iOS 渲染管线的核心流程：

```
CPU 准备阶段          GPU 渲染阶段          显示
┌─────────────┐    ┌─────────────┐    ┌──────┐
│ 布局计算     │    │ 顶点处理     │    │      │
│ 文本测量     │ →  │ 光栅化       │ →  │ 屏幕  │
│ 图片解码     │    │ 像素合成     │    │      │
│ 图层打包提交  │    │ 帧缓冲写入   │    │      │
└─────────────┘    └─────────────┘    └──────┘
     ← 一帧 16.67ms (60fps) / 8.33ms (120fps ProMotion) →
```

- **掉帧原因：** CPU 或 GPU 在一帧时间内未完成工作
- **CPU 瓶颈：** 布局计算复杂、图片解码耗时、文本测量频繁
- **GPU 瓶颈：** 离屏渲染、大量透明度混合、过度绘制

---

## 离屏渲染

### 什么触发离屏渲染

GPU 需要在屏幕外开辟缓冲区先渲染再合成，代价是：额外的内存分配 + 上下文切换。

| 操作 | 是否离屏渲染 | 说明 |
|------|------------|------|
| `cornerRadius` + `masksToBounds` | **是** | 最常见的离屏渲染原因 |
| `shadow`（无 shadowPath） | **是** | 系统需遍历图层计算阴影形状 |
| `mask` | **是** | 需要离屏合成 |
| `groupOpacity`（多子图层 + opacity < 1） | **是** | 需要先合成再设透明度 |
| `shouldRasterize = true` | **首次是，后续缓存** | 缓存后反而提升性能 |
| `cornerRadius` 不设 `masksToBounds` | **否** | 仅背景裁剪 |
| `shadow` + `shadowPath` | **否** | 已提供阴影路径，无需计算 |

### 优化离屏渲染

```swift
// ❌ 圆角 + 裁剪 → 离屏渲染
imageView.layer.cornerRadius = 20
imageView.layer.masksToBounds = true  // 触发离屏渲染

// ✅ 方案一：预处理圆角图片（最优，完全避免离屏渲染）
extension UIImage {
    func roundedImage(radius: CGFloat) -> UIImage {
        let renderer = UIGraphicsImageRenderer(size: size)
        return renderer.image { ctx in
            let rect = CGRect(origin: .zero, size: size)
            UIBezierPath(roundedRect: rect, cornerRadius: radius).addClip()
            draw(in: rect)
        }
    }
}

// ✅ 方案二：UIBezierPath 直接绘制圆角（适合自定义 draw 的视图）
class RoundedImageView: UIView {
    var image: UIImage?

    override func draw(_ rect: CGRect) {
        guard let image = image else { return }
        let path = UIBezierPath(roundedRect: rect, cornerRadius: 20)
        path.addClip()
        image.draw(in: rect)
    }
}

// ✅ 方案三：cornerRadius + cornerCurve（iOS 13+，简单圆角最推荐）
// 仅设置 cornerRadius 而不设置 masksToBounds 时不会离屏渲染
// iOS 13+ 对 CALayer 做了优化，某些情况下 cornerRadius + masksToBounds 也能避免离屏渲染
imageView.layer.cornerRadius = 20
imageView.layer.cornerCurve = .continuous  // iOS 13+，连续曲率圆角
// 注意：如果图层有子图层或背景色以外的复杂内容，仍可能触发离屏渲染
// 建议用 Instruments Core Animation → Color Offscreen-Rendered 验证

// ⚠️ CAShapeLayer mask：仍然会触发离屏渲染
// layer.mask 本身就在离屏渲染触发列表中（见上方表格）
// 仅在需要不规则形状裁剪且无法预处理图片时使用
let maskLayer = CAShapeLayer()
maskLayer.path = UIBezierPath(roundedRect: imageView.bounds,
                               cornerRadius: 20).cgPath
imageView.layer.mask = maskLayer  // ⚠️ 触发离屏渲染，适用场景有限

// ❌ 阴影无 shadowPath → 离屏渲染
view.layer.shadowColor = UIColor.black.cgColor
view.layer.shadowOffset = CGSize(width: 0, height: 2)
view.layer.shadowOpacity = 0.3

// ✅ 提供 shadowPath → 避免离屏渲染
view.layer.shadowPath = UIBezierPath(roundedRect: view.bounds,
                                      cornerRadius: view.layer.cornerRadius).cgPath
```

---

## Core Animation 调优

### shouldRasterize

```swift
// ✅ 适用于内容不频繁变化的复杂图层（如带阴影的卡片）
cardView.layer.shouldRasterize = true
cardView.layer.rasterizationScale = UIScreen.main.scale  // 必须设置，否则模糊

// ⚠️ 注意事项：
// - 缓存有效期约 100ms，超时重新渲染
// - 内容频繁变化的图层不适用（如动画中的视图）
// - 会增加内存占用（缓存位图）
// - Instruments 中绿色 = 缓存命中，红色 = 缓存失效
```

### drawsAsynchronously

```swift
// ✅ 适用于自定义 draw(_:) 方法的视图
customView.layer.drawsAsynchronously = true
// Core Animation 会在后台线程准备绘制内容
// 适用场景：复杂的自定义绘制、图表、波形图
```

---

## UITableView / UICollectionView 滚动优化

### Cell 高度缓存

```swift
// ❌ 每次滚动都重新计算高度
func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
    // 复杂的高度计算...
    return calculateHeight(for: data[indexPath.row])
}

// ✅ 缓存已计算的高度
private var heightCache: [IndexPath: CGFloat] = [:]

func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
    if let cached = heightCache[indexPath] {
        return cached
    }
    let height = calculateHeight(for: data[indexPath.row])
    heightCache[indexPath] = height
    return height
}

// ✅ 更好的方案：使用 UITableView.automaticDimension + estimatedRowHeight
tableView.rowHeight = UITableView.automaticDimension
tableView.estimatedRowHeight = 80  // 提供合理估算值，减少布局计算
```

### 图片异步解码

```swift
// ❌ 在主线程解码大图片（UIImage 延迟解码，滚动时才触发）
func configure(with url: URL) {
    imageView.image = UIImage(contentsOfFile: url.path)  // 首次显示时在主线程解码
}

// ✅ 在后台线程预解码
func configure(with url: URL) {
    DispatchQueue.global(qos: .userInitiated).async {
        guard let image = UIImage(contentsOfFile: url.path),
              let cgImage = image.cgImage else { return }

        // 强制解码
        let renderer = UIGraphicsImageRenderer(size: image.size)
        let decodedImage = renderer.image { ctx in
            ctx.cgContext.draw(cgImage, in: CGRect(origin: .zero, size: image.size))
        }

        DispatchQueue.main.async { [weak self] in
            self?.imageView.image = decodedImage
        }
    }
}
```

### Cell 复用优化

```swift
// ✅ 在 prepareForReuse 中重置状态，取消进行中的任务
class ProductCell: UITableViewCell {
    private var imageTask: Task<Void, Never>?

    override func prepareForReuse() {
        super.prepareForReuse()
        imageTask?.cancel()  // 取消上一个图片加载任务
        imageView?.image = nil
        titleLabel.text = nil
    }

    func configure(with product: Product) {
        titleLabel.text = product.title
        imageTask = Task {
            let image = await ImageLoader.load(product.imageURL)
            guard !Task.isCancelled else { return }
            imageView?.image = image
        }
    }
}
```

---

## SwiftUI 渲染优化

### 避免不必要重绘

```swift
// ❌ body 中包含不依赖状态变化的昂贵计算
struct ProductListView: View {
    @State private var searchText = ""

    var body: some View {
        VStack {
            TextField("搜索", text: $searchText)
            ExpensiveChartView()  // 每次 searchText 变化都重建
        }
    }
}

// ✅ 提取为独立子视图，减少重绘范围
struct ProductListView: View {
    @State private var searchText = ""

    var body: some View {
        VStack {
            SearchBar(text: $searchText)
            ChartContainer()  // 独立视图，不受 searchText 影响
        }
    }
}

struct ChartContainer: View, Equatable {
    var body: some View {
        ExpensiveChartView()
    }

    static func == (lhs: ChartContainer, rhs: ChartContainer) -> Bool {
        return true  // 内容不变，跳过重绘
    }
}
```

### LazyVStack / LazyHStack

```swift
// ❌ VStack 在 ScrollView 中一次性创建所有视图
ScrollView {
    VStack {
        ForEach(items) { item in
            ItemView(item: item)  // 1000 个 item 全部创建
        }
    }
}

// ✅ LazyVStack 只创建可见区域的视图
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ItemView(item: item)  // 只创建屏幕可见的
        }
    }
}
```

### 调试 SwiftUI 重绘

```swift
// 在 body 中添加，查看触发重绘的原因
var body: some View {
    let _ = Self._printChanges()  // Xcode 15+，打印触发重绘的属性
    // ...
}
```

---

## 帧率监控

### CADisplayLink

```swift
// 开发阶段帧率监控
class FPSMonitor {
    private var displayLink: CADisplayLink?
    private var lastTimestamp: CFTimeInterval = 0
    private var frameCount: Int = 0

    func start() {
        displayLink = CADisplayLink(target: self, selector: #selector(tick))
        displayLink?.add(to: .main, forMode: .common)
    }

    @objc private func tick(_ link: CADisplayLink) {
        frameCount += 1
        let elapsed = link.timestamp - lastTimestamp
        if elapsed >= 1.0 {
            let fps = Double(frameCount) / elapsed
            print("FPS: \(Int(fps))")
            frameCount = 0
            lastTimestamp = link.timestamp
        }
    }

    func stop() {
        displayLink?.invalidate()
        displayLink = nil
    }
}
```

### MetricKit 线上帧率监控

```swift
// iOS 14+ 通过 MetricKit 获取卡顿数据
func didReceive(_ payloads: [MXMetricPayload]) {
    for payload in payloads {
        if let animationMetrics = payload.animationMetrics {
            let scrollHitchRate = animationMetrics.scrollHitchTimeRatio
            // scrollHitchTimeRatio < 5ms/s 为优秀
        }
    }
}
```

---

## Instruments Core Animation 模板使用

### 调试选项

| 选项 | 用途 | 优化目标 |
|------|------|---------|
| **Color Blended Layers** | 绿色=不透明，红色=透明混合 | 减少红色区域 |
| **Color Offscreen-Rendered** | 黄色=离屏渲染 | 消除不必要的离屏渲染 |
| **Color Misaligned Images** | 黄色=像素未对齐 | 确保图片尺寸匹配显示尺寸 |
| **Color Copied Images** | 蓝色=图片被拷贝 | 避免不必要的图片拷贝 |

---

## 渲染优化 checklist

```
- [ ] 是否在真机上使用 Instruments Core Animation 检测掉帧？
- [ ] 离屏渲染是否已优化（圆角预处理、shadowPath）？
- [ ] Cell 是否正确复用且在 prepareForReuse 中重置？
- [ ] 大图片是否异步解码而非主线程延迟解码？
- [ ] Cell 高度是否有缓存或使用 estimatedRowHeight？
- [ ] 透明视图是否设置了 isOpaque = true（非透明时）？
- [ ] backgroundColor 是否设置为非透明色（避免 blended layers）？
- [ ] SwiftUI 列表是否使用 LazyVStack/LazyHStack？
- [ ] SwiftUI 是否避免了不必要的视图重绘（Equatable / 提取子视图）？
- [ ] 复杂静态图层是否使用 shouldRasterize 缓存？
- [ ] 滚动过程中是否避免了同步 I/O 和复杂计算？
```

## 常见问题速查

```
看到列表滚动掉帧
  → 原因：Cell 内图片主线程解码 / 高度频繁计算 / 离屏渲染
  → 优化：异步解码 + 高度缓存 + 消除离屏渲染

看到 Color Offscreen-Rendered 大面积黄色
  → 原因：圆角 + masksToBounds / 无 shadowPath 的阴影
  → 优化：预处理圆角图片 + 设置 shadowPath

看到 Color Blended Layers 大面积红色
  → 原因：视图背景透明，GPU 需要混合多层像素
  → 优化：设置不透明背景色 + isOpaque = true

看到 SwiftUI 列表卡顿
  → 原因：使用 VStack 而非 LazyVStack / 不必要的视图重绘
  → 优化：LazyVStack + 提取子视图 + Equatable

看到 CPU 占用高但 GPU 正常
  → 原因：主线程布局计算过重 / Auto Layout 约束过多
  → 优化：简化视图层级 / 手动计算 frame / 预计算布局

看到图片显示时短暂卡顿
  → 原因：UIImage 懒解码，首次显示时在主线程解码
  → 优化：后台预解码或使用 ImageIO 降采样
```

---

## 动画性能优化

### CAAnimation / Core Animation

```swift
// ✅ 使用 Core Animation 而非逐帧手动更新
// Core Animation 动画在 Render Server（独立进程）执行，不阻塞主线程
let animation = CABasicAnimation(keyPath: "position.y")
animation.fromValue = 0
animation.toValue = 200
animation.duration = 0.3
animation.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
layer.add(animation, forKey: "moveY")

// ⚠️ 注意事项：
// - 动画期间 layer 的模型值不会自动更新，需手动设置最终值
// - 同时添加多个动画时使用 CAAnimationGroup 以减少提交次数
// - 避免在动画过程中频繁修改 layer 属性（会导致隐式动画叠加）
```

### UIView.animate

```swift
// ✅ UIView.animate 底层也是 Core Animation，性能良好
UIView.animate(withDuration: 0.3, delay: 0, options: .curveEaseInOut) {
    view.transform = CGAffineTransform(translationX: 0, y: 200)
}

// ❌ 在动画闭包中执行布局计算
UIView.animate(withDuration: 0.3) {
    self.complexView.isHidden = false
    self.view.layoutIfNeeded()  // 触发大量 Auto Layout 计算
}

// ✅ 提前计算好布局，动画中只做属性变更
complexView.alpha = 0
complexView.isHidden = false
view.layoutIfNeeded()  // 先完成布局
UIView.animate(withDuration: 0.3) {
    self.complexView.alpha = 1  // 动画中只做简单属性变更
}

// ⚠️ spring 动画注意：
// - UIView.animate(withDuration:delay:usingSpringWithDamping:...)
// - damping < 1.0 时动画更长，帧数更多，性能开销更大
// - 复杂视图上使用弹簧动画前先验证帧率
```

### SwiftUI 动画

```swift
// ✅ 使用 .animation 修饰符绑定到特定值
struct AnimatedView: View {
    @State private var isExpanded = false

    var body: some View {
        RoundedRectangle(cornerRadius: 10)
            .frame(height: isExpanded ? 200 : 50)
            .animation(.easeInOut(duration: 0.3), value: isExpanded)  // 仅 isExpanded 变化时触发
    }
}

// ❌ 使用无绑定的 .animation（已废弃，会导致不必要的动画）
// .animation(.easeInOut)  // 任何状态变化都触发动画

// ✅ withAnimation 控制动画范围
Button("Toggle") {
    withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) {
        isExpanded.toggle()
    }
}

// ⚠️ SwiftUI 动画性能注意：
// - 避免在 .animation 中使用复杂的自定义 Animatable 计算
// - Canvas / TimelineView 比大量 SwiftUI 视图动画更适合粒子效果等复杂场景
// - .drawingGroup() 可将 SwiftUI 视图合并为单个 Metal 纹理，提升复杂视图动画性能
```

### 通用动画性能原则

```
- 优先使用系统动画 API（Core Animation / UIView.animate / .animation）
  → 它们在 Render Server 或 GPU 上执行，不阻塞主线程
- 避免使用 Timer/CADisplayLink 手动逐帧更新（除非需要自定义绘制动画）
- 动画属性选择：
  → transform / opacity / position → GPU 执行，性能好
  → frame / bounds / Auto Layout 约束 → 需要 CPU 重新布局，性能差
- 动画期间避免触发离屏渲染
- 复杂动画考虑使用 Lottie 或 Core Animation 预合成
- 在低端设备上验证动画帧率（iPhone 8 / iPhone SE）
```
