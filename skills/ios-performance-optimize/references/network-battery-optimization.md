# 网络与电量优化

## 网络优化

### 请求合并

```swift
// ❌ 短时间内发送大量独立请求
func loadDashboard() {
    fetchUserProfile()      // 请求 1
    fetchNotifications()    // 请求 2
    fetchRecommendations()  // 请求 3
    fetchFeedItems()        // 请求 4
}

// ✅ 合并为批量请求或使用 TaskGroup 并行
func loadDashboard() async {
    async let profile = api.fetchUserProfile()
    async let notifications = api.fetchNotifications()
    async let recommendations = api.fetchRecommendations()

    let (p, n, r) = await (profile, notifications, recommendations)
    updateUI(profile: p, notifications: n, recommendations: r)
}

// ✅ 后端支持时使用批量 API
func loadDashboard() async {
    let response = await api.batchFetch(endpoints: ["/profile", "/notifications", "/recommendations"])
    // 一个请求返回所有数据
}
```

### 预加载

```swift
// ✅ 列表滚动到接近底部时预加载下一页
func scrollViewDidScroll(_ scrollView: UIScrollView) {
    let threshold = scrollView.contentSize.height - scrollView.frame.height * 2
    if scrollView.contentOffset.y > threshold && !isLoading {
        loadNextPage()
    }
}

// ✅ 预加载可能跳转的页面数据
func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
    let item = items[indexPath.row]
    DetailDataPrefetcher.shared.prefetch(for: item.id)
}
```

### 缓存策略

```swift
// ✅ 配置 URLCache
let cache = URLCache(
    memoryCapacity: 20 * 1024 * 1024,   // 20MB 内存缓存
    diskCapacity: 100 * 1024 * 1024,      // 100MB 磁盘缓存
    diskPath: "network_cache"
)
URLCache.shared = cache

// ✅ 使用 HTTP 缓存头
// 服务端配置 Cache-Control / ETag / Last-Modified
// 客户端配置缓存策略
let request = URLRequest(url: url, cachePolicy: .returnCacheDataElseLoad)

// ✅ 业务层缓存策略
class DataRepository {
    private let cache = NSCache<NSString, CacheEntry>()

    func fetchData(forKey key: String) async -> Data {
        // 1. 先检查内存缓存
        if let entry = cache.object(forKey: key as NSString), !entry.isExpired {
            return entry.data
        }

        // 2. 检查磁盘缓存
        if let diskData = DiskCache.read(key: key) {
            cache.setObject(CacheEntry(data: diskData), forKey: key as NSString)
            return diskData
        }

        // 3. 网络请求
        let data = await networkService.fetch(key: key)
        cache.setObject(CacheEntry(data: data), forKey: key as NSString)
        DiskCache.write(data, key: key)
        return data
    }
}
```

### HTTP/2 多路复用

```swift
// ✅ URLSession 默认支持 HTTP/2（服务端也需要支持）
// 多个请求复用同一 TCP 连接，减少连接建立开销

// ✅ 共享 URLSession 实例（复用连接池）
class NetworkService {
    // 全局共享，复用连接
    static let session: URLSession = {
        let config = URLSessionConfiguration.default
        config.httpMaximumConnectionsPerHost = 6  // 默认值
        config.timeoutIntervalForRequest = 30
        return URLSession(configuration: config)
    }()
}

// ❌ 每次请求创建新的 URLSession（无法复用连接）
func fetch(url: URL) async throws -> Data {
    let session = URLSession(configuration: .default)  // 每次新建
    let (data, _) = try await session.data(from: url)
    return data
}
```

### 数据压缩

```swift
// ✅ 请求头声明支持压缩（URLSession 默认支持 gzip）
// 确保服务端开启 gzip/br 压缩

// ✅ 上传大数据时压缩
import Compression

func compressData(_ data: Data) -> Data? {
    let destinationBuffer = UnsafeMutablePointer<UInt8>.allocate(capacity: data.count)
    defer { destinationBuffer.deallocate() }

    let compressedSize = data.withUnsafeBytes { sourceBuffer in
        compression_encode_buffer(destinationBuffer, data.count,
                                  sourceBuffer.bindMemory(to: UInt8.self).baseAddress!,
                                  data.count, nil, COMPRESSION_ZLIB)
    }

    guard compressedSize > 0 else { return nil }
    return Data(bytes: destinationBuffer, count: compressedSize)
}
```

---

## 弱网适配

### 超时策略

```swift
// ✅ 分级超时策略
enum NetworkEnvironment {
    case normal
    case weak

    var timeoutInterval: TimeInterval {
        switch self {
        case .normal: return 15
        case .weak: return 30
        }
    }

    var retryCount: Int {
        switch self {
        case .normal: return 1
        case .weak: return 3
        }
    }
}

// ✅ 使用 NWPathMonitor 监控网络状态
import Network

class NetworkMonitor {
    private let monitor = NWPathMonitor()

    var isConnected: Bool = true
    var isExpensive: Bool = false  // 蜂窝网络
    var isConstrained: Bool = false  // 低数据模式

    func start() {
        monitor.pathUpdateHandler = { [weak self] path in
            self?.isConnected = (path.status == .satisfied)
            self?.isExpensive = path.isExpensive
            self?.isConstrained = path.isConstrained
        }
        monitor.start(queue: .global(qos: .utility))
    }
}
```

### 降级方案

```swift
// ✅ 弱网时降低数据量
func fetchImages(quality: ImageQuality) async -> [UIImage] {
    let networkQuality = NetworkMonitor.shared.isExpensive ? .low : .high

    switch networkQuality {
    case .high:
        return await fetchFullResolutionImages()
    case .low:
        return await fetchThumbnails()  // 弱网只加载缩略图
    }
}

// ✅ 弱网时显示缓存数据 + 提示
func loadContent() async {
    // 先展示缓存
    if let cached = cache.get(key: contentKey) {
        showContent(cached)
    }

    // 再尝试更新
    do {
        let fresh = try await api.fetchContent()
        showContent(fresh)
        cache.set(fresh, key: contentKey)
    } catch {
        if cache.get(key: contentKey) != nil {
            showToast("网络不佳，显示缓存内容")
        } else {
            showRetryView()
        }
    }
}
```

### 离线模式

```swift
// ✅ 关键操作支持离线队列
class OfflineQueue {
    private var pendingOperations: [PendingOperation] = []

    func enqueue(_ operation: PendingOperation) {
        pendingOperations.append(operation)
        saveToDisk()  // 持久化，防止进程被杀
    }

    func processWhenOnline() {
        NotificationCenter.default.addObserver(
            forName: .networkBecameAvailable, object: nil, queue: .main
        ) { [weak self] _ in
            self?.processPendingOperations()
        }
    }
}
```

---

## 电量优化

### 后台任务（BGTaskScheduler）

```swift
import BackgroundTasks

// ✅ 注册后台任务
func registerBackgroundTasks() {
    // 短时任务（最多 30 秒）
    BGTaskScheduler.shared.register(forTaskWithIdentifier: "com.app.refresh",
                                     using: nil) { task in
        self.handleAppRefresh(task: task as! BGAppRefreshTask)
    }

    // 长时任务（可达数分钟，系统决定）
    BGTaskScheduler.shared.register(forTaskWithIdentifier: "com.app.processing",
                                     using: nil) { task in
        self.handleProcessing(task: task as! BGProcessingTask)
    }
}

// ✅ 调度后台刷新
func scheduleAppRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: "com.app.refresh")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)  // 最早 15 分钟后
    try? BGTaskScheduler.shared.submit(request)
}

// ❌ 避免的后台模式
// - 不要用后台定位仅仅为了保活
// - 不要用静音音频播放保活
// - 不要频繁唤醒做无意义的刷新
```

### 定位策略

```swift
import CoreLocation

// ❌ 始终使用最高精度
locationManager.desiredAccuracy = kCLLocationAccuracyBest  // 耗电最多

// ✅ 根据场景选择精度
enum LocationUseCase {
    case navigation   // 导航需要高精度
    case cityLevel    // 天气/内容推荐
    case background   // 后台打卡

    var accuracy: CLLocationAccuracy {
        switch self {
        case .navigation: return kCLLocationAccuracyBest
        case .cityLevel: return kCLLocationAccuracyKilometer
        case .background: return kCLLocationAccuracyHundredMeters
        }
    }

    var distanceFilter: CLLocationDistance {
        switch self {
        case .navigation: return 10     // 每 10 米更新
        case .cityLevel: return 1000    // 每 1km 更新
        case .background: return 500    // 每 500 米更新
        }
    }
}

// ✅ 不需要实时位置时使用 significantLocationChanges
locationManager.startMonitoringSignificantLocationChanges()
// 耗电极低，仅在基站切换时触发（约 500 米）

// ✅ 使用完毕立即停止
func viewDidDisappear(_ animated: Bool) {
    super.viewDidDisappear(animated)
    locationManager.stopUpdatingLocation()
}
```

### 推送唤醒频率

```swift
// ✅ 控制推送唤醒频率，避免频繁唤醒
// 静默推送（content-available: 1）会唤醒 App，消耗电量
// Apple 会对高频静默推送进行限流

// 建议：
// - 静默推送频率不超过每小时 2-3 次
// - 非紧急数据同步使用 BGTaskScheduler 而非静默推送
// - 紧急通知使用 priority: 10，非紧急使用 priority: 5（系统会延迟低优先级推送合并发送）
```

---

## 能耗监控

### MetricKit

```swift
import MetricKit

class EnergyMetricsSubscriber: NSObject, MXMetricManagerSubscriber {
    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            // CPU 使用指标
            if let cpuMetrics = payload.cpuMetrics {
                let cumulativeCPUTime = cpuMetrics.cumulativeCPUTime
                reportMetric("cpu_time", value: cumulativeCPUTime)
            }

            // 定位指标
            if let locationMetrics = payload.locationActivityMetrics {
                let cumulativeBGTime = locationMetrics.cumulativeBestAccuracyForNavigationTime
                reportMetric("bg_location_time", value: cumulativeBGTime)
            }

            // 网络指标
            if let networkMetrics = payload.networkTransferMetrics {
                let cellularUpload = networkMetrics.cumulativeCellularUpload
                let cellularDownload = networkMetrics.cumulativeCellularDownload
                reportMetric("cellular_data", value: cellularUpload + cellularDownload)
            }
        }
    }

    // 诊断数据（崩溃、卡顿、能耗异常）
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        for payload in payloads {
            if let cpuExceptions = payload.cpuExceptionDiagnostics {
                // CPU 异常使用（后台 CPU 占用过高）
                for exception in cpuExceptions {
                    reportDiagnostic("cpu_exception", callStack: exception.callStackTree)
                }
            }
        }
    }
}
```

### Energy Log（Xcode Organizer）

```
1. Xcode → Window → Organizer → Energy
2. 查看各设备的能耗报告
3. 关注 "Energy Exception" — 表示 App 能耗异常

常见能耗异常：
- CPU 后台占用过高（> 80% 持续超过 30s）
- 频繁的网络请求唤醒
- 后台定位持续运行
- 频繁的 I/O 操作
```

---

## 网络与电量优化 checklist

```
- [ ] 是否共享 URLSession 实例而非每次创建？
- [ ] 是否对频繁请求做了合并或批量处理？
- [ ] 是否配置了合理的 HTTP 缓存策略？
- [ ] 弱网环境是否有降级方案？
- [ ] 关键操作是否支持离线队列？
- [ ] 后台任务是否使用 BGTaskScheduler 而非保活 hack？
- [ ] 定位精度是否按场景分级？
- [ ] 不需要定位时是否停止了更新？
- [ ] 静默推送频率是否控制在合理范围？
- [ ] 是否有能耗监控（MetricKit / Organizer Energy Log）？
- [ ] 大数据传输是否启用了压缩？
- [ ] 图片是否按需加载适合的分辨率？
```

## 常见问题速查

```
看到网络请求耗时长
  → 原因：未复用连接 / 串行请求 / 未启用 HTTP/2
  → 优化：共享 URLSession + 并行请求 + 确认服务端支持 HTTP/2

看到弱网下 App 无响应
  → 原因：请求超时过长 / 未显示缓存 / 无重试机制
  → 优化：分级超时 + 先显示缓存 + 弱网降级

看到后台电量消耗高
  → 原因：后台定位持续运行 / 频繁静默推送唤醒
  → 优化：降低定位精度 + 使用 significantLocationChanges + 减少静默推送

看到蜂窝数据消耗高
  → 原因：未压缩 / 加载高清图 / 未缓存
  → 优化：启用压缩 + 低数据模式检测（isConstrained）+ 缓存策略

看到 MetricKit 报告 CPU Exception
  → 原因：后台任务 CPU 占用过高
  → 优化：检查后台任务的计算量，限制后台 CPU 使用时间
```
