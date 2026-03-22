# Instruments 工具指南

## 工具选择决策树

```
性能问题是什么？
│
├── 启动慢 → App Launch 模板
├── 界面卡顿 / 掉帧 → Core Animation 模板 或 Time Profiler
├── 内存占用高 / OOM → Allocations 模板
├── 内存泄漏 → Leaks 模板 或 Memory Graph Debugger
├── CPU 占用高 → Time Profiler
├── 网络慢 → Network 模板
├── 线程阻塞 / 锁竞争 → System Trace
├── 电量消耗高 → Energy Log（Instruments 或 Organizer）
├── 磁盘 I/O → File Activity 模板
└── 不确定 → 先用 Time Profiler 定位热点
```

---

## Time Profiler 深度使用

### 基本流程

```
1. Product → Profile（Cmd+I） → Time Profiler
2. 点击红色录制按钮
3. 在 App 上操作复现性能问题
4. 停止录制
5. 选中问题时间段（拖拽选择）
6. 分析 Call Tree
```

### 采样原理

Time Profiler 通过**定时采样**（默认每 1ms）记录线程调用栈：
- 记录的是"哪个函数在执行"，而非"哪个函数执行了多久"
- 采样次数越多 ≈ 该函数占用 CPU 时间越长
- **不是精确的执行时间**，而是统计概率

### Call Tree 分析技巧

**底部面板设置（关键）：**

| 选项 | 说明 | 建议 |
|------|------|------|
| Separate by Thread | 按线程分组 | **必须勾选** — 区分主线程和后台线程 |
| Invert Call Tree | 反转调用栈 | **必须勾选** — 显示叶子函数（真正耗时的函数） |
| Hide System Libraries | 隐藏系统库 | **建议勾选** — 聚焦自己的代码 |
| Flatten Recursion | 合并递归调用 | 可选 — 递归函数分析时勾选 |
| Top Functions | 合并同函数 | 可选 — 显示函数的总耗时 |

### Focus 和 Prune

```
Focus（聚焦）：
  右键点击某个函数 → "Focus on Subtree"
  → 只显示该函数及其子调用的耗时
  → 用于深入分析某个具体的调用路径

Prune（剪枝）：
  右键点击某个函数 → "Prune Subtree"
  → 隐藏该函数及其子调用
  → 用于排除已知的、不关注的耗时路径

Charge（归因）：
  右键 → "Charge to Caller"
  → 将该函数的耗时归到其调用者
  → 用于跳过中间层函数
```

### 分析主线程卡顿

```
1. 勾选 Separate by Thread
2. 找到 "Main Thread" 展开
3. 找到采样数最高的函数
4. 双击跳转到源代码
5. 确认该操作是否应该在主线程执行
   → 如果是 I/O / 网络 / 大量计算 → 移到后台线程
   → 如果是布局计算 → 简化视图层级或缓存
```

---

## Allocations 深度使用

### 基本流程

```
1. Instruments → Allocations 模板
2. 录制并操作 App
3. 观察内存曲线
4. 使用 Mark Generation 分析增长
```

### Generation Analysis（世代分析）

```
排查内存持续增长：
1. 进入某页面 → 点击 "Mark Generation"（Generation A）
2. 操作页面
3. 退出页面 → 点击 "Mark Generation"（Generation B）
4. 再次进入并退出 → 点击 "Mark Generation"（Generation C）

分析：
  Generation B 中的对象 = 操作期间分配的
  如果 Generation B 中有对象在 C 时仍存在 → 可能泄漏
  反复进出相同页面，Growth 列持续增长 → 确认泄漏
```

### 过滤和分析

```
Statistics 视图：
  - "All Allocations" → 看到所有分配的对象
  - 按 "Persistent" 排序 → 找到未释放的大对象
  - 搜索类名过滤 → 专注于怀疑泄漏的类型

Call Tree 视图：
  - 切换到 "Call Tree" → 看到分配代码路径
  - Invert Call Tree → 找到分配最多内存的叶子函数
```

### 关注指标

| 指标 | 含义 | 关注 |
|------|------|------|
| All Heap Allocations | 堆上所有分配 | 整体趋势 |
| Persistent | 当前仍存活的对象 | **内存泄漏排查关键** |
| Transient | 已释放的对象 | 分配频率 |
| Total Bytes | 累计分配量 | 是否频繁大量分配 |
| #Persistent | 存活对象数量 | 持续增长 = 泄漏 |

---

## Leaks 使用

### 基本流程

```
1. Instruments → Leaks 模板
2. 录制并操作 App（反复进出页面）
3. 泄漏自动检测，红色标记
4. 选中泄漏 → 展开查看引用链
5. 定位到代码中的循环引用
```

### 注意事项

```
- Leaks 只能检测"真正的泄漏"（无任何引用指向的内存）
- 循环引用（retain cycle）不算 Leaks 工具的 "leak"
  → 两个对象互相引用，内存可达但不会释放
  → 这种情况用 Memory Graph Debugger 更容易发现
- 结合 Allocations 的 Generation Analysis 排查循环引用
```

---

## Core Animation 深度使用

### 调试选项详解

```
Color Blended Layers（透明混合）：
  绿色 = 不透明，GPU 无需混合
  红色 = 透明，GPU 需要计算多层混合
  优化：
    - 设置 view.backgroundColor = .white（非透明色）
    - 设置 view.isOpaque = true
    - UILabel 设置 backgroundColor（避免默认透明）

Color Offscreen-Rendered（离屏渲染）：
  黄色 = 触发了离屏渲染
  优化：
    - cornerRadius + masksToBounds → 预处理圆角
    - shadow 无 shadowPath → 设置 shadowPath
    - 复杂遮罩 → 简化或预渲染

Color Misaligned Images（图片未对齐）：
  黄色 = 图片尺寸与 ImageView 尺寸不匹配
  洋红色 = 图片像素未对齐到屏幕像素
  优化：
    - 下载/缓存时按显示尺寸裁剪
    - 确保 image.size * scale == imageView.frame.size * screenScale

Color Copied Images（图片拷贝）：
  蓝色 = 图片在渲染前被拷贝了
  优化：
    - 使用正确的色彩空间（避免 Core Animation 转换）
    - 避免不必要的图片格式转换
```

### FPS 监控

```
Instruments Core Animation 模板自动显示 FPS：
  60 fps = 每帧 16.67ms，流畅
  低于 45 fps = 用户可感知卡顿
  低于 30 fps = 明显卡顿

ProMotion 设备（iPhone 13 Pro+）：
  目标 120 fps = 每帧 8.33ms
  但系统会动态调整帧率，不必追求 120 常驻
```

---

## Network 使用

### 基本流程

```
1. Instruments → Network 模板
2. 录制并操作 App
3. 查看请求列表：URL、耗时、数据量
4. 分析：
   - 是否有重复请求？
   - 是否有串行请求可以并行？
   - 响应数据量是否合理？
   - 是否有不必要的请求？
```

---

## System Trace 使用

### 适用场景

```
System Trace 适用于：
  - 线程调度分析（线程被阻塞/唤醒的原因）
  - 锁竞争分析
  - 系统调用耗时
  - 主线程被后台线程阻塞
```

### 分析线程调度

```
1. Instruments → System Trace
2. 录制问题场景
3. 找到 Main Thread
4. 查看 Thread State：
   Running（绿色）= 正在执行
   Blocked（红色）= 被阻塞（等待锁/I/O/信号量）
   Preempted（橙色）= 被系统抢占
   Waiting（灰色）= 等待调度

5. 选中 Blocked 区间 → 查看阻塞原因
   → 通常是锁竞争或同步 I/O
```

### 锁竞争分析

```
看到主线程频繁 Blocked：
1. 查看 Blocked 的原因（通常是 mutex / dispatch_sync）
2. 追踪持有锁的线程
3. 优化方案：
   - dispatch_sync → dispatch_async
   - 缩小锁的粒度
   - 使用无锁数据结构
   - 使用 actor（Swift Concurrency）
```

---

## os_signpost 自定义标记

### 基础用法

```swift
import os.signpost

let log = OSLog(subsystem: "com.yourapp", category: "Performance")

// 标记时间区间
func loadData() {
    os_signpost(.begin, log: log, name: "DataLoad")

    // ... 数据加载操作

    os_signpost(.end, log: log, name: "DataLoad")
}

// 带附加信息
os_signpost(.begin, log: log, name: "ImageDecode", "%{public}s", imageURL.lastPathComponent)
// ... 解码
os_signpost(.end, log: log, name: "ImageDecode")
```

### 在 Instruments 中查看

```
1. Instruments → 添加 "os_signpost" instrument
2. 或使用 "Blank" 模板手动添加
3. 录制后可以看到自定义标记的时间区间
4. 可以看到每次调用的耗时、统计数据

适用场景：
  - 启动各阶段耗时
  - 网络请求耗时
  - 图片加载和解码耗时
  - 业务流程耗时
```

---

## MetricKit 线上监控

### 配置

```swift
import MetricKit

class MetricsManager: NSObject, MXMetricManagerSubscriber {
    static let shared = MetricsManager()

    func start() {
        MXMetricManager.shared.add(self)
    }

    // 每日汇总指标（约每 24 小时回调一次）
    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            processPayload(payload)
        }
    }

    // 诊断数据（崩溃、卡顿、CPU 异常等）
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        for payload in payloads {
            processDiagnostic(payload)
        }
    }
}
```

### 可获取的指标

| 指标类 | 包含信息 | iOS 版本 |
|--------|---------|---------|
| `applicationLaunchMetrics` | 启动耗时分布 | 13+ |
| `applicationExitMetrics` | 退出原因（OOM/Watchdog/崩溃） | 14+ |
| `memoryMetrics` | 内存峰值 | 13+ |
| `cpuMetrics` | CPU 累计使用时间 | 13+ |
| `diskIOMetrics` | 磁盘读写量 | 13+ |
| `networkTransferMetrics` | 网络传输量 | 13+ |
| `animationMetrics` | 滚动卡顿率（Hitch Rate） | 14+ |
| `locationActivityMetrics` | 定位使用时间 | 13+ |
| `applicationTimeMetrics` | 前台/后台时间 | 13+ |

### 诊断数据

```swift
func processDiagnostic(_ payload: MXDiagnosticPayload) {
    // 崩溃诊断
    if let crashDiagnostics = payload.crashDiagnostics {
        for crash in crashDiagnostics {
            let callStack = crash.callStackTree
            reportCrash(callStack)
        }
    }

    // 卡顿诊断（主线程挂起 > 250ms）
    if let hangDiagnostics = payload.hangDiagnostics {
        for hang in hangDiagnostics {
            let duration = hang.hangDuration
            reportHang(duration: duration, callStack: hang.callStackTree)
        }
    }

    // CPU 异常
    if let cpuExceptions = payload.cpuExceptionDiagnostics {
        for exception in cpuExceptions {
            reportCPUException(callStack: exception.callStackTree)
        }
    }

    // 磁盘写入异常
    if let diskExceptions = payload.diskWriteExceptionDiagnostics {
        for exception in diskExceptions {
            reportDiskException(callStack: exception.callStackTree)
        }
    }
}
```

---

## App Hangs 排查

### 什么是 App Hang

主线程被阻塞超过 **250ms** 即被系统视为一次 Hang。用户体感为界面"卡住"、点击无响应。

- **严重等级：** > 250ms 为 Hang，> 1s 为严重 Hang
- **系统影响：** iOS 会在设置中标记频繁 Hang 的 App，影响用户信任度
- **Xcode Organizer：** Xcode → Window → Organizer → Hangs 可查看线上 Hang 报告

### 排查方法

**方法一：Xcode 主线程检测器**

```
Product → Scheme → Edit Scheme → Diagnostics
  → 勾选 "Main Thread Checker"（默认开启）
  → 勾选 "Thread Performance Checker"（检测主线程耗时操作）
  → 运行时触发主线程违规会在控制台打印紫色警告
```

**方法二：Instruments → Time Profiler**

```
1. 使用 Time Profiler 录制问题场景
2. 勾选 Separate by Thread → 找到 Main Thread
3. 查找超过 250ms 的连续 CPU 占用区间
4. Invert Call Tree → 定位阻塞主线程的具体函数
```

**方法三：Instruments → System Trace**

```
1. 使用 System Trace 录制
2. 找到 Main Thread → 查看 Thread State
3. 长时间 Blocked（红色）= 主线程被阻塞
4. 查看阻塞原因（锁、同步 I/O、dispatch_sync）
5. 追踪持有锁的线程或阻塞的系统调用
```

**方法四：MetricKit 线上监控**

```swift
func didReceive(_ payloads: [MXDiagnosticPayload]) {
    for payload in payloads {
        if let hangDiagnostics = payload.hangDiagnostics {
            for hang in hangDiagnostics {
                // hangDuration：挂起时长
                let duration = hang.hangDuration
                let callStack = hang.callStackTree
                reportHang(duration: duration, callStack: callStack)
            }
        }
    }
}
```

### 常见 Hang 原因与修复

```
主线程同步 I/O（文件读写、数据库查询）
  → 修复：移到后台线程执行

主线程同步网络请求
  → 修复：使用 async/await 或异步回调

主线程等待锁（dispatch_sync / NSLock）
  → 修复：改为 dispatch_async 或减小锁粒度

Auto Layout 计算过重
  → 修复：简化视图层级 / 减少约束数量

大量 JSON 解析在主线程
  → 修复：在后台线程解析，主线程仅更新 UI

CoreData fetch 在主线程
  → 修复：使用 performBackgroundTask 或后台 context
```

---

## Instruments 使用 checklist

```
- [ ] 是否在真机上进行 profiling？（不是模拟器）
- [ ] 是否使用 Release 配置进行 profiling？（Debug 有额外开销）
- [ ] Time Profiler 是否勾选了 Separate by Thread + Invert Call Tree？
- [ ] 是否选中了问题时间段再分析（而非全时段）？
- [ ] 是否多次录制取平均值？
- [ ] Allocations 是否使用了 Mark Generation 做对比？
- [ ] 是否用 os_signpost 标记了业务关键路径？
- [ ] 线上是否集成了 MetricKit 进行持续监控？
```

## 常见问题速查

```
看到 Time Profiler 中主线程采样占比 > 80%
  → 原因：主线程执行了耗时操作
  → 排查：Invert Call Tree 找到叶子函数，确认是否应移到后台

看到 Allocations 内存曲线只升不降
  → 原因：内存泄漏或缓存无上限
  → 排查：Mark Generation 对比 + Memory Graph 定位泄漏链

看到 Core Animation FPS 频繁低于 45
  → 原因：离屏渲染 / 主线程阻塞 / 复杂布局
  → 排查：开启调试选项定位问题区域

看到 System Trace 主线程大量 Blocked
  → 原因：锁竞争 / 同步 I/O / dispatch_sync
  → 排查：查看阻塞原因，改为异步操作

看到 os_signpost 标记的区间耗时波动大
  → 原因：受其他进程/线程影响 / 系统资源竞争
  → 排查：多次测量取中位数，排除异常值
```
