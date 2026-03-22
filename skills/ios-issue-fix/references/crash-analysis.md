# 崩溃日志分析与符号化

## 崩溃日志获取渠道

| 渠道 | 适用场景 | 获取方式 |
|------|---------|---------|
| Xcode Organizer | 已上架/TestFlight | Window → Organizer → Crashes |
| TestFlight 反馈 | 内测用户崩溃 | App Store Connect → TestFlight → Crashes |
| Firebase Crashlytics | 线上监控 | Firebase Console → Crashlytics |
| Sentry | 线上监控 | Sentry Dashboard → Issues |
| Bugly | 国内项目常用 | Bugly 控制台 → 崩溃分析 |
| 设备日志 | 本地调试 | Xcode → Devices → View Device Logs |
| 用户导出 | 用户配合 | 设置 → 隐私 → 分析 → 分析数据 |

---

## 崩溃日志符号化

### 全自动符号化（Xcode Organizer）

Xcode Organizer 接收到的崩溃日志会自动匹配本地 dSYM 进行符号化，前提：
- 上传构建时勾选了 "Include app symbols for your application"
- 本地 Archive 未被删除（dSYM 在 Archive 内）

### 半自动符号化（atos 命令）

当自动符号化失败时，手动使用 `atos`：

```bash
# 基本用法
atos -arch arm64 -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp -l 0x加载地址 0x崩溃地址

# 批量符号化
# 1. 找到崩溃日志中的 Binary Images 段，获取加载地址
# 2. 对每个未符号化的地址逐一执行 atos
```

### dSYM 管理

| 操作 | 命令/路径 |
|------|----------|
| 查找本地 dSYM | `mdfind "com_apple_xcode_dsym_uuids == <UUID>"` |
| 获取 dSYM UUID | `dwarfdump --uuid MyApp.app.dSYM` |
| 获取崩溃日志 UUID | 崩溃日志 Binary Images 段中的 UUID |
| 从 App Store Connect 下载 | App Store Connect → App → Activity → Build → Download dSYM |
| Archive 中的 dSYM | `~/Library/Developer/Xcode/Archives/` → 右键 Show in Finder |

### 符号化常见失败原因

| 现象 | 原因 | 解决 |
|------|------|------|
| 地址显示为 `0x...` 无函数名 | dSYM 未找到 | 确认 UUID 匹配，重新下载 dSYM |
| 仅系统框架已符号化 | 缺少应用 dSYM | 从 Archive 或 App Store Connect 获取 |
| dSYM UUID 不匹配 | 构建版本不对应 | 确认 Build Number 一致 |
| Bitcode 重编译后 dSYM 变化 | Apple 重编译了二进制 | 从 App Store Connect 下载正确的 dSYM |
| 第三方库未符号化 | 缺少第三方 dSYM | 联系 SDK 提供方获取，或检查 Pods/SPM 的 dSYM 输出设置 |

---

## 崩溃日志阅读

### 关键字段解读

```
# 1. Exception Type — 崩溃信号类型
Exception Type: EXC_BAD_ACCESS (SIGSEGV)

# 2. Exception Codes — 错误码，辅助判断原因
Exception Codes: KERN_INVALID_ADDRESS at 0x0000000000000000  # 空指针

# 3. Termination Reason — 系统终止原因
Termination Reason: Namespace SIGNAL, Code 0xb  # SIGSEGV

# 4. Triggered by Thread — 崩溃发生的线程
Triggered by Thread: 0  # 主线程

# 5. Thread 0 Crashed — 崩溃线程调用栈（从下往上读）
Thread 0 Crashed:
0   MyApp         0x... MyViewController.viewDidLoad() + 120  # 崩溃点
1   UIKitCore     0x... -[UIViewController _sendViewDidLoadWithAppearanceProxyObjectTaggingEnabled] + 100
```

### 按 Exception Type 分类处理

| Exception Type | 含义 | 排查方向 |
|---------------|------|---------|
| EXC_BAD_ACCESS (SIGSEGV) | 访问无效内存 | 空指针、野指针、已释放对象 → 开启 Zombie Objects |
| EXC_BAD_ACCESS (SIGBUS) | 内存对齐错误 | Unmanaged/UnsafePointer 使用不当 |
| EXC_BAD_INSTRUCTION (SIGILL) | 非法指令 | Swift 强制解包 nil、数组越界、enum 未匹配 |
| EXC_CRASH (SIGABRT) | 主动中止 | assert/precondition/fatalError 触发、ObjC 异常未捕获 |
| EXC_CRASH (SIGKILL) | 系统强杀 | Watchdog 超时、内存超限、后台任务超时 |
| EXC_RESOURCE | 资源超限 | CPU 使用过高、内存增长过快 |
| EXC_GUARD | 受保护资源违规 | 文件描述符/端口操作不当 |

---

## 线上高频崩溃模式

### Top 5 Swift 崩溃

| # | 模式 | 典型日志特征 | 修复方向 |
|---|------|------------|---------|
| 1 | 强制解包 nil | `EXC_BAD_INSTRUCTION`，调用栈包含 `Swift runtime failure: unexpectedly found nil` | 替换为 `guard let` / `if let` / `??` |
| 2 | 数组越界 | `EXC_BAD_INSTRUCTION`，`Swift runtime failure: Index out of range` | 添加 bounds 检查，使用 `safe` 下标扩展 |
| 3 | 主线程外更新 UI | `Main Thread Checker` 警告或随机 UI 崩溃 | 添加 `@MainActor` 标注或 `DispatchQueue.main.async` |
| 4 | Task 泄漏/页面销毁后回调 | `EXC_BAD_ACCESS`，涉及已释放的 ViewController | 存储 Task 引用，deinit 中 cancel |
| 5 | 枚举未匹配（后端新增字段） | `EXC_BAD_INSTRUCTION`，switch 未匹配 | 添加 `default` case 或使用 `String` RawValue + 兜底 |

### Top 5 ObjC 崩溃

| # | 模式 | 典型日志特征 | 修复方向 |
|---|------|------------|---------|
| 1 | unrecognized selector | `SIGABRT`，`-[NSNull objectForKey:]` 或类似 | 检查服务端返回 null 的处理 |
| 2 | KVO 未移除 observer | `SIGABRT`，`was deallocated while key value observers were still registered` | 在 dealloc 中移除 observer，或改用 Combine |
| 3 | 集合类型修改时遍历 | `SIGABRT`，`mutated while being enumerated` | 使用 copy 遍历或 DispatchQueue 串行保护 |
| 4 | NSInternalInconsistencyException | `SIGABRT`，`Invalid update: invalid number of rows` | UITableView/CollectionView 数据源与 UI 不同步 |
| 5 | 野指针 (EXC_BAD_ACCESS) | `EXC_BAD_ACCESS`，地址非 0x0 | 开启 Zombie Objects 定位已释放对象 |

---

## 何时升级为架构问题

以下信号说明崩溃不是孤立的 Bug，而是架构层面的缺陷：

| 信号 | 含义 | 建议 |
|------|------|------|
| 同一崩溃点在多个版本反复出现 | 根因未真正修复 | 重新分析根因，可能需要重构数据流 |
| 同类崩溃分散在 10+ 个不同位置 | 缺少统一的安全访问层 | 引入安全的下标扩展/统一的可选值处理 |
| 崩溃仅在高并发场景出现 | 缺少线程安全设计 | 引入 actor 或统一的并发模型 |
| 内存崩溃持续增长 | 存在系统性内存泄漏 | 全面排查引用关系，考虑引入内存监控 |
| 第三方 SDK 内部崩溃无法修复 | 依赖不可控 | 评估替换 SDK 或添加防护层 |

此时应记录问题并单独排期架构优化，不在 Bug 修复 PR 中混入重构。
