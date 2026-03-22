# 持久化实现模式

## 技术选型决策树

| 数据特征 | 推荐方案 | 说明 |
|---------|---------|------|
| 少量键值对（设置项、标志位） | UserDefaults | 简单直接，同步读写 |
| 敏感信息（token、密码、密钥） | Keychain | 加密存储，系统级安全 |
| 结构化业务数据（iOS 17+） | SwiftData | 声明式 API，与 SwiftUI 深度集成 |
| 结构化业务数据（需兼容低版本） | Core Data | 成熟稳定，功能完备 |
| 大文件（图片、PDF、视频） | FileManager（沙盒目录） | 存文件路径而非文件本身 |
| 临时缓存数据 | NSCache / URLCache | 系统自动管理内存 |
| 需要跨平台共享的数据 | CloudKit / iCloud KV Store | 依赖 Apple 生态 |

**原则：** 选择团队最熟悉且满足需求的最简方案。不要为了简单的键值存储引入 Core Data。

---

## UserDefaults 最佳实践

```swift
// MARK: - 使用 @propertyWrapper 封装
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    let container: UserDefaults

    init(_ key: String, defaultValue: T, container: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = defaultValue
        self.container = container
    }

    var wrappedValue: T {
        get { container.object(forKey: key) as? T ?? defaultValue }
        set { container.set(newValue, forKey: key) }
    }
}

// MARK: - 集中管理所有 Key
enum AppSettings {
    @UserDefault("hasCompletedOnboarding", defaultValue: false)
    static var hasCompletedOnboarding: Bool

    @UserDefault("lastSyncDate", defaultValue: nil)
    static var lastSyncDate: Date?

    @UserDefault("preferredTheme", defaultValue: "system")
    static var preferredTheme: String
}
```

**注意事项：**
- 不要存储大量数据（> 几百 KB），会影响启动性能
- 不要存储敏感信息（密码、token），使用 Keychain
- Key 统一管理，避免散落在各处导致拼写错误
- App Group 共享：使用 `UserDefaults(suiteName: "group.xxx")` 在主 App 和 Extension 间共享

---

## Keychain 最佳实践

```swift
// MARK: - Keychain 封装
enum KeychainHelper {
    enum KeychainError: Error {
        case saveFailed(OSStatus)
        case readFailed
        case deleteFailed(OSStatus)
    }

    static func save(_ data: Data, for key: String) throws {
        // 先删除旧值
        let deleteQuery: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
        ]
        SecItemDelete(deleteQuery as CFDictionary)

        // 保存新值
        let addQuery: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock,
        ]
        let status = SecItemAdd(addQuery as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    static func read(for key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
        ]
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        guard status == errSecSuccess, let data = result as? Data else {
            throw KeychainError.readFailed
        }
        return data
    }

    static func delete(for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
        ]
        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.deleteFailed(status)
        }
    }
}

// MARK: - 使用示例
// 保存 token
let tokenData = accessToken.data(using: .utf8)!
try KeychainHelper.save(tokenData, for: "accessToken")

// 读取 token
let data = try KeychainHelper.read(for: "accessToken")
let token = String(data: data, encoding: .utf8)
```

**注意事项：**
- `kSecAttrAccessible` 选择合适的保护级别，`afterFirstUnlock` 适用于大多数场景
- 登出时记得清除 Keychain 中的 token
- 使用 `kSecAttrAccessGroup` 在同一开发者的多个 App 间共享

---

## SwiftData 模式（iOS 17+）

```swift
import SwiftData

// MARK: - 模型定义
@Model
final class Task {
    var title: String
    var isCompleted: Bool
    var createdAt: Date
    var priority: Priority

    @Relationship(deleteRule: .cascade)
    var subtasks: [Subtask]

    init(title: String, priority: Priority = .medium) {
        self.title = title
        self.isCompleted = false
        self.createdAt = Date()
        self.priority = priority
        self.subtasks = []
    }

    enum Priority: Int, Codable {
        case low, medium, high
    }
}

// MARK: - 在 SwiftUI 中使用
struct TaskListView: View {
    @Query(sort: \Task.createdAt, order: .reverse)
    private var tasks: [Task]

    @Environment(\.modelContext) private var context

    var body: some View {
        List(tasks) { task in
            TaskRow(task: task)
                .swipeActions {
                    Button(role: .destructive) {
                        context.delete(task)
                    } label: {
                        Label("删除", systemImage: "trash")
                    }
                }
        }
    }

    func addTask(title: String) {
        let task = Task(title: title)
        context.insert(task)
    }
}

// MARK: - App 入口配置
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [Task.self])
    }
}
```

---

## Core Data 模式

```swift
// MARK: - NSManagedObject 子类
@objc(TaskEntity)
public class TaskEntity: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var title: String
    @NSManaged public var isCompleted: Bool
    @NSManaged public var createdAt: Date
}

// MARK: - Core Data Stack
final class CoreDataStack {
    static let shared = CoreDataStack()

    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "AppModel")
        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Core Data 加载失败: \(error)")
            }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
        return container
    }()

    var viewContext: NSManagedObjectContext {
        persistentContainer.viewContext
    }

    // 后台写入
    func performBackgroundTask(_ block: @escaping (NSManagedObjectContext) -> Void) {
        persistentContainer.performBackgroundTask(block)
    }

    func save() {
        let context = viewContext
        guard context.hasChanges else { return }
        do {
            try context.save()
        } catch {
            // 记录日志，不要 fatalError
            print("Core Data 保存失败: \(error)")
        }
    }
}
```

**Core Data 注意事项：**
- `viewContext` 只在主线程操作
- 写入操作使用 `performBackgroundTask` 避免阻塞 UI
- 大批量导入使用 `NSBatchInsertRequest`
- 查询优化：使用 `fetchBatchSize`、`NSFetchedResultsController`

---

## 数据迁移策略

| 场景 | 策略 | 说明 |
|------|------|------|
| 新增字段 | 轻量级迁移（Lightweight Migration） | Core Data / SwiftData 自动处理 |
| 重命名字段 | 映射模型（Mapping Model） | 需要手动创建映射关系 |
| 结构大幅变更 | 自定义迁移（Custom Migration） | 编写迁移代码逐步转换 |
| UserDefaults 结构变更 | 版本号 + 迁移方法 | 在 App 启动时检查并迁移 |

### UserDefaults 迁移示例

```swift
enum SettingsMigration {
    static func migrateIfNeeded() {
        let currentVersion = 2
        let savedVersion = UserDefaults.standard.integer(forKey: "settingsVersion")

        guard savedVersion < currentVersion else { return }

        if savedVersion < 1 {
            // v0 → v1：将 "theme" 字符串迁移为 "preferredTheme"
            if let oldTheme = UserDefaults.standard.string(forKey: "theme") {
                UserDefaults.standard.set(oldTheme, forKey: "preferredTheme")
                UserDefaults.standard.removeObject(forKey: "theme")
            }
        }

        if savedVersion < 2 {
            // v1 → v2：其他迁移逻辑
        }

        UserDefaults.standard.set(currentVersion, forKey: "settingsVersion")
    }
}
```

### Core Data 轻量级迁移

```swift
// 在 NSPersistentContainer 初始化时启用自动迁移
let description = NSPersistentStoreDescription()
description.shouldMigrateStoreAutomatically = true
description.shouldInferMappingModelAutomatically = true
container.persistentStoreDescriptions = [description]
```

**迁移注意事项：**
- 永远不要删除旧版本的 .xcdatamodel，保留完整版本链
- 测试迁移路径：从每个历史版本升级到最新版本
- 大量数据的迁移考虑异步执行，显示迁移进度
