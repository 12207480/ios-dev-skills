# 内存优化

## 内存指标解读

iOS 中不同内存指标的含义：

| 指标 | 含义 | 关注度 |
|------|------|--------|
| **Footprint（占用量）** | 系统计费的内存量，Jetsam 依据此值决定是否杀进程 | **最关键** |
| **Dirty Memory** | 被 App 修改过的内存页，不可被系统回收 | 高 |
| **Compressed Memory** | 系统压缩的不活跃 Dirty Memory | 中 |
| **Clean Memory** | 可被系统回收重建的内存（如 mmap 的文件） | 低 |
| **Virtual Memory** | 虚拟地址空间，不代表实际物理占用 | 参考 |

> **核心公式：** Footprint ≈ Dirty + Compressed。优化目标是降低 Footprint。

---

## OOM 分析

### 前台 OOM vs 后台 OOM

| 类型 | 触发条件 | 特征 | 排查难度 |
|------|---------|------|---------|
| **前台 OOM（FOOM）** | 前台运行时内存超限 | 用户正在操作时闪退，无崩溃日志 | 高 |
| **后台 OOM（BOOM）** | 后台挂起时内存超限 | 下次打开 App 重新启动 | 中 |

### Jetsam 机制

系统通过 Jetsam 管理内存压力：
- 每个设备有不同的内存上限（非固定值，取决于设备状态）
- iPhone 通常前台限制在 **设备物理内存的 50%-70%**
- 后台限制更低，通常 **50MB-80MB**
- 收到 `didReceiveMemoryWarning` 是最后的释放机会

### OOM 排查步骤

```
1. 确认是否为 OOM（无崩溃堆栈、Jetsam 日志）
2. 使用 Instruments → Allocations 复现内存增长过程
3. 使用 Memory Graph Debugger 在内存峰值时捕获
4. 分析大对象和持续增长的对象
5. 检查是否有内存泄漏（Leaks Instrument）
```

---

## 内存泄漏排查

### Memory Graph Debugger

```
排查步骤：
1. 操作 App 到怀疑泄漏的页面
2. 退出该页面
3. Debug Navigator → Memory → 点击相机图标捕获
4. 左侧面板搜索应该已释放的类名
5. 如果对象仍存在 → 查看引用关系图
6. 追踪引用链，找到形成环的路径
7. 紫色感叹号 = Xcode 检测到的泄漏
```

### Leaks Instrument

```
排查步骤：
1. Instruments → Leaks 模板
2. 操作 App，反复进出页面
3. 自动检测到的泄漏会标记红色
4. 展开泄漏详情，查看引用链
5. 定位到代码中的循环引用
```

### 常见内存泄漏模式

```swift
// ❌ 闭包循环引用
class ViewController: UIViewController {
    var onComplete: (() -> Void)?

    func setup() {
        onComplete = {
            self.doSomething()  // self 强引用 onComplete，onComplete 强引用 self
        }
    }
}

// ✅ 使用 [weak self]
func setup() {
    onComplete = { [weak self] in
        self?.doSomething()
    }
}

// ❌ Timer 循环引用
class ViewController: UIViewController {
    var timer: Timer?

    func startTimer() {
        timer = Timer.scheduledTimer(timeInterval: 1.0, target: self,
                                     selector: #selector(tick), userInfo: nil, repeats: true)
        // Timer 强引用 target(self)，self 强引用 timer → 泄漏
    }
}

// ✅ 使用 block-based Timer + [weak self]
func startTimer() {
    timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
        self?.tick()
    }
}

// ❌ Combine 忘记存储 cancellable
func subscribe() {
    publisher.sink { value in  // AnyCancellable 被立即释放，订阅立即取消
        self.handle(value)
    }
}

// ✅ 存储 cancellable 并使用 [weak self]
var cancellables = Set<AnyCancellable>()

func subscribe() {
    publisher.sink { [weak self] value in
        self?.handle(value)
    }.store(in: &cancellables)
}
```

---

## 大对象治理

### 图片降采样（ImageIO）

```swift
// ❌ 直接加载原图（一张 4000x3000 的图片解码后占用 ~48MB）
let image = UIImage(contentsOfFile: path)

// ✅ 使用 ImageIO 降采样，按目标尺寸加载
func downsample(imageAt url: URL, to pointSize: CGSize, scale: CGFloat) -> UIImage? {
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let imageSource = CGImageSourceCreateWithURL(url as CFURL, imageSourceOptions) else {
        return nil
    }

    let maxDimensionInPixels = max(pointSize.width, pointSize.height) * scale
    let downsampleOptions = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,       // 在降采样时立即解码
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxDimensionInPixels
    ] as CFDictionary

    guard let downsampledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions) else {
        return nil
    }
    return UIImage(cgImage: downsampledImage)
}
```

### 缓存策略（NSCache）

```swift
// ✅ 使用 NSCache 而非 Dictionary 做图片缓存
// NSCache 在内存压力时自动淘汰，Dictionary 不会
class ImageCache {
    static let shared = ImageCache()

    private let cache = NSCache<NSString, UIImage>()

    init() {
        cache.countLimit = 100           // 最多缓存 100 张
        cache.totalCostLimit = 50 * 1024 * 1024  // 最大 50MB
    }

    func image(for key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }

    func store(_ image: UIImage, for key: String) {
        let cost = Int(image.size.width * image.size.height * image.scale * image.scale * 4)
        cache.setObject(image, forKey: key as NSString, cost: cost)
    }
}
```

---

## autorelease pool 优化

```swift
// ❌ 循环中创建大量临时对象，autorelease pool 不及时释放
func processLargeDataset(_ items: [Data]) {
    for item in items {
        let image = UIImage(data: item)  // 临时对象堆积
        process(image)
    }
}

// ✅ 在循环内使用 autoreleasepool 及时释放
func processLargeDataset(_ items: [Data]) {
    for item in items {
        autoreleasepool {
            let image = UIImage(data: item)
            process(image)
        }
        // 每次循环结束后临时对象被释放
    }
}
```

---

## Swift 值类型 vs 引用类型的内存影响

| 特性 | 值类型（struct/enum） | 引用类型（class） |
|------|---------------------|------------------|
| 存储位置 | 栈（小）/ 堆（大） | 堆 |
| 拷贝行为 | 深拷贝（COW 优化） | 引用计数 |
| 内存泄漏风险 | 无 | 有（循环引用） |
| 额外开销 | 无引用计数开销 | 引用计数 +16 字节对象头 |

**注意事项：**

```swift
// ⚠️ struct 中包含引用类型时，拷贝会增加引用计数
struct ViewModel {
    let title: String        // String 有 COW，影响小
    let image: UIImage       // 引用类型，拷贝增加 retain
    let items: [Item]        // Array 有 COW，影响小
}

// ⚠️ 大 struct 频繁拷贝可能比 class 更慢
// 超过 3-4 个引用类型属性的 struct → 考虑用 class
```

---

## 内存优化 checklist

```
- [ ] 是否使用 Instruments Allocations 检查内存峰值？
- [ ] 是否使用 Memory Graph 检查内存泄漏？
- [ ] 大图片是否使用 ImageIO 降采样而非直接加载？
- [ ] 图片缓存是否使用 NSCache 并设置上限？
- [ ] 所有闭包是否正确使用 [weak self]？
- [ ] Timer / NotificationCenter / Combine 是否正确释放？
- [ ] delegate 是否声明为 weak？
- [ ] 是否响应 didReceiveMemoryWarning 释放资源？
- [ ] 大量循环中是否使用 autoreleasepool？
- [ ] 是否有线上内存监控（MetricKit / 自定义）？
- [ ] 后台挂起时是否释放非必要资源？
```

## 常见问题速查

```
看到内存持续增长不回落
  → 原因：内存泄漏（循环引用 / 未释放资源）
  → 优化：Memory Graph 定位泄漏链，修复循环引用

看到内存峰值过高导致 OOM
  → 原因：大图片未降采样 / 缓存无上限 / 一次性加载大量数据
  → 优化：ImageIO 降采样 + NSCache 设上限 + 分页加载

看到后台被频繁杀死
  → 原因：后台内存超限（Jetsam）
  → 优化：进入后台时释放图片缓存、Web 内容等非必要资源

看到 ViewController 退出后不释放
  → 原因：闭包/Timer/delegate 循环引用
  → 优化：[weak self] + Timer invalidate + weak delegate

看到 Dirty Memory 占比过高
  → 原因：大量被修改的内存页，通常是解码后的图片
  → 优化：按需加载 + 及时释放 + 使用 mmap 减少 dirty 页
```
