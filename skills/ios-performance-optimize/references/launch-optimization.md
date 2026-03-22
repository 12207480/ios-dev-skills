# 启动优化

## 启动阶段划分

iOS App 启动分为两大阶段：

### pre-main（系统阶段）

从用户点击图标到 `main()` 函数执行前，由 dyld 和系统负责：

| 阶段 | 耗时因素 | 优化方向 |
|------|---------|---------|
| 加载动态库（dylib loading） | 动态库数量、依赖深度 | 减少动态库，合并或转静态库 |
| Rebase/Bind | Mach-O 中指针修正数量 | 减少 ObjC 类/分类/selector 数量 |
| ObjC Runtime Setup | 类注册、分类注册 | 减少不必要的类和分类 |
| Initializers | `+load` 方法、C++ 全局构造函数、`__attribute__((constructor))` | 移除 +load，用 +initialize 替代 |

### post-main（应用阶段）

从 `main()` 到首屏渲染完成（首帧展示），由应用代码负责：

| 阶段 | 耗时因素 | 优化方向 |
|------|---------|---------|
| `application(_:didFinishLaunchingWithOptions:)` | SDK 初始化、服务注册 | 延迟非必要初始化 |
| 首屏 ViewController 创建 | 视图层级复杂度、数据加载 | 异步加载、简化首屏 |
| 首屏布局和渲染 | Auto Layout 计算、图片解码 | 减少首屏视图层级 |
| 首帧展示（`viewDidAppear`） | 首屏数据请求 | 骨架屏 + 异步请求 |

---

## 冷启动 vs 温启动

| 类型 | 定义 | 关注重点 |
|------|------|---------|
| **冷启动** | App 进程不存在，从零开始 | 完整优化（pre-main + post-main） |
| **温启动** | App 进程被系统杀死但缓存仍在 | 主要关注 post-main |
| **热启动（恢复）** | App 在后台未被杀死 | 通常无需优化 |

> 性能基线应以**冷启动**为准，因为这是用户体验最差的场景。测试前重启设备或确认进程已完全退出。

---

## pre-main 优化

### 减少动态库

```
# 查看 App 使用的动态库数量
otool -L YourApp.app/YourApp | wc -l

# 查看各动态库加载耗时（环境变量）
DYLD_PRINT_STATISTICS=1
DYLD_PRINT_STATISTICS_DETAILS=1
```

**优化措施：**
- 将不频繁更新的动态库转为静态库
- 合并功能相近的小型动态库
- 目标：系统动态库外的自定义动态库控制在 6 个以内

### +load 治理

> **注意：** `+load` 和 `+initialize` 均为 ObjC 运行时方法。Swift 中 `override class func load()` 和 `override class func initialize()` 均已不可用（编译器直接报错）。Swift 应使用 `static let` 惰性初始化或 `@UIApplicationDelegateAdaptor` 进行启动时配置。

```objc
// ❌ ObjC 中避免使用 +load
@implementation MyManager
+ (void)load {
    // 启动时同步执行，阻塞主线程
    [self setupSomething];
}
@end

// ✅ ObjC 中改用 +initialize（懒加载，首次使用时才调用）
@implementation MyManager
+ (void)initialize {
    if (self == [MyManager class]) {
        // 首次使用该类时才执行（需判断 self 防止子类重复调用）
        [self setupSomething];
    }
}
@end
```

```swift
// ✅ Swift 使用 static let 惰性初始化（线程安全，首次访问时执行）
class MyManager {
    static let shared: MyManager = {
        let manager = MyManager()
        manager.setup()
        return manager
    }()
}

// ✅ Swift 使用 @UIApplicationDelegateAdaptor 进行启动配置（SwiftUI App）
@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    var body: some Scene { WindowGroup { ContentView() } }
}
```

### 二进制重排

通过调整代码在二进制中的排列顺序，减少启动时的 Page Fault：

1. 使用 Instruments → System Trace 记录启动阶段的 Page Fault
2. 收集启动阶段调用的函数符号（Clang SanitizerCoverage 或 fishhook）
3. 生成 order file，在 Xcode Build Settings → Order File 配置
4. 验证 Page Fault 次数是否减少

---

## post-main 优化

### 延迟初始化

```swift
// ❌ 全部在 didFinishLaunching 中初始化
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    setupAnalytics()      // 非首屏必须
    setupPushNotification() // 非首屏必须
    setupDatabase()       // 首屏需要
    setupTheme()          // 首屏需要
    setupAdSDK()          // 非首屏必须
    return true
}

// ✅ 分级初始化
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    // 第一优先级：首屏展示必须
    setupDatabase()
    setupTheme()

    // 第二优先级：首屏展示后异步初始化
    DispatchQueue.main.async {
        self.setupAnalytics()
        self.setupPushNotification()
    }

    // 第三优先级：用户首次触发时懒初始化
    // setupAdSDK() → 在广告页面首次打开时初始化
    return true
}
```

### 首屏渲染优化

```swift
// ✅ 使用骨架屏，避免首屏空白等待
class HomeViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        showSkeletonView()  // 立即展示骨架屏

        Task {
            let data = await fetchHomeData()
            hideSkeletonView()
            updateUI(with: data)
        }
    }
}

// ✅ 首屏图片使用缩略图，完整图异步加载
func configureCell(with item: HomeItem) {
    // 先展示低分辨率缩略图
    imageView.image = item.thumbnailImage

    // 异步加载高清图
    Task {
        let fullImage = await ImageLoader.load(item.fullImageURL)
        imageView.image = fullImage
    }
}
```

---

## 启动耗时测量方法

### Instruments App Launch

最准确的官方测量方式：
1. Xcode → Product → Profile → App Launch
2. 录制完整冷启动过程
3. 查看各阶段耗时（Process Lifecycle → App Lifecycle）

### os_signpost 自定义标记

```swift
import os.signpost

let launchLog = OSLog(subsystem: "com.yourapp", category: "Launch")

// 在 main.swift 或 AppDelegate
os_signpost(.begin, log: launchLog, name: "AppLaunch")

// 在首屏 viewDidAppear 中
os_signpost(.end, log: launchLog, name: "AppLaunch")
```

### MetricKit 线上监控

```swift
import MetricKit

class MetricsManager: NSObject, MXMetricManagerSubscriber {
    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            if let launchMetrics = payload.applicationLaunchMetrics {
                let histogrammedTime = launchMetrics.histogrammedTimeToFirstDraw
                // 上报启动耗时分布
                reportLaunchTime(histogrammedTime)
            }
        }
    }
}
```

### CFAbsoluteTimeGetCurrent 简易打点

```swift
// 适用于快速验证，不适合精确测量
let startTime = CFAbsoluteTimeGetCurrent()
// ... 执行操作
let elapsed = CFAbsoluteTimeGetCurrent() - startTime
print("耗时: \(elapsed * 1000)ms")
```

---

## 启动优化 checklist

```
- [ ] 是否在真机上测量冷启动时间？（不是模拟器）
- [ ] 自定义动态库数量是否 ≤ 6 个？
- [ ] 是否消除了所有 +load 方法？
- [ ] didFinishLaunching 中是否只保留首屏必须的初始化？
- [ ] 非首屏初始化是否延迟到首屏展示后？
- [ ] 首屏是否使用骨架屏或占位图避免白屏？
- [ ] 首屏数据是否异步加载？
- [ ] 首屏视图层级是否简洁（避免过深嵌套）？
- [ ] 是否有启动耗时的线上监控（MetricKit / 自定义埋点）？
- [ ] 启动路径上是否避免了同步网络请求和文件 I/O？
```

## 常见问题速查

```
看到 pre-main 耗时 > 1s
  → 原因：动态库过多或 +load 方法过多
  → 优化：减少动态库、消除 +load

看到 didFinishLaunching 耗时 > 500ms
  → 原因：首屏不需要的 SDK 在此同步初始化
  → 优化：分级初始化，延迟非必要 SDK

看到首屏白屏时间长
  → 原因：首屏数据依赖网络请求，阻塞渲染
  → 优化：骨架屏 + 异步加载 + 缓存上次数据

看到启动时大量 Page Fault
  → 原因：启动路径代码分散在不同内存页
  → 优化：二进制重排，集中启动路径代码
```
