# 网络层实现模式

## API Service 标准模板

### Protocol + async throws

```swift
// MARK: - Service 协议
protocol UserServiceProtocol: Sendable {
    func fetchUser(id: String) async throws -> User
    func updateUser(_ user: User) async throws -> User
    func deleteUser(id: String) async throws
}

// MARK: - 具体实现
final class UserService: UserServiceProtocol {
    private let client: APIClientProtocol

    init(client: APIClientProtocol = APIClient.shared) {
        self.client = client
    }

    func fetchUser(id: String) async throws -> User {
        try await client.request(
            endpoint: .get("/users/\(id)"),
            responseType: UserResponse.self
        ).toDomain()
    }

    func updateUser(_ user: User) async throws -> User {
        try await client.request(
            endpoint: .put("/users/\(user.id)", body: UpdateUserRequest(user)),
            responseType: UserResponse.self
        ).toDomain()
    }

    func deleteUser(id: String) async throws {
        try await client.request(
            endpoint: .delete("/users/\(id)")
        )
    }
}
```

---

## Codable 模型设计

### Response 模型 vs 业务模型分离

```swift
// MARK: - API Response 模型（仅用于 JSON 解析）
struct UserResponse: Codable {
    let id: Int
    let userName: String
    let avatarUrl: String?
    let createdAt: String

    enum CodingKeys: String, CodingKey {
        case id
        case userName = "user_name"
        case avatarUrl = "avatar_url"
        case createdAt = "created_at"
    }
}

// MARK: - 业务模型（App 内部使用）
struct User: Identifiable, Hashable {
    let id: String
    let name: String
    let avatarURL: URL?
    let createdDate: Date
}

// MARK: - 转换
extension UserResponse {
    func toDomain() -> User {
        User(
            id: String(id),
            name: userName,
            avatarURL: avatarUrl.flatMap(URL.init),
            createdDate: ISO8601DateFormatter().date(from: createdAt) ?? Date()
        )
    }
}
```

**为什么要分离：**
- Response 模型随后端接口变化，业务模型保持稳定
- 业务模型可以包含计算属性和便捷方法，不受 Codable 限制
- 接口字段重命名只需改 `CodingKeys`，不影响 App 其他代码

### CodingKeys 技巧

```swift
// 统一使用 convertFromSnakeCase 策略（减少手写 CodingKeys）
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
decoder.dateDecodingStrategy = .iso8601

// 仅在需要自定义映射时才手写 CodingKeys
```

---

## 分页加载模式

### Cursor-based 分页（推荐）

```swift
struct PagedResponse<T: Codable>: Codable {
    let items: [T]
    let nextCursor: String?  // nil 表示没有更多数据
    let hasMore: Bool
}

@MainActor
@Observable
final class PaginatedListViewModel<T: Codable & Identifiable> {
    private(set) var items: [T] = []
    private(set) var isLoading = false
    private(set) var hasMore = true
    private var nextCursor: String?

    private let fetchPage: (String?) async throws -> PagedResponse<T>

    init(fetchPage: @escaping (String?) async throws -> PagedResponse<T>) {
        self.fetchPage = fetchPage
    }

    func loadFirstPage() async {
        items = []
        nextCursor = nil
        hasMore = true
        await loadNextPage()
    }

    func loadNextPage() async {
        guard !isLoading, hasMore else { return }
        isLoading = true
        defer { isLoading = false }

        do {
            let response = try await fetchPage(nextCursor)
            items.append(contentsOf: response.items)
            nextCursor = response.nextCursor
            hasMore = response.hasMore
        } catch {
            // 分页加载失败不清空已有数据
            // 由调用方决定错误提示方式
        }
    }
}
```

### Offset-based 分页

```swift
// offset/limit 模式（适用于传统 REST API）
func fetchItems(offset: Int, limit: Int = 20) async throws -> [Item] {
    try await client.request(
        endpoint: .get("/items", query: ["offset": offset, "limit": limit]),
        responseType: [ItemResponse].self
    ).map { $0.toDomain() }
}
```

---

## 错误处理与用户提示

```swift
// MARK: - 统一错误类型
enum APIError: Error, LocalizedError {
    case networkUnavailable
    case unauthorized
    case serverError(statusCode: Int)
    case decodingFailed(underlying: Error)
    case custom(message: String)

    var errorDescription: String? {
        switch self {
        case .networkUnavailable: return "网络连接不可用，请检查网络设置"
        case .unauthorized: return "登录已过期，请重新登录"
        case .serverError(let code): return "服务器错误（\(code)），请稍后重试"
        case .decodingFailed: return "数据解析异常，请更新至最新版本"
        case .custom(let message): return message
        }
    }
}

// MARK: - ViewModel 中的错误处理
func loadData() async {
    do {
        let data = try await service.fetchData()
        state = .loaded(data)
    } catch let error as APIError {
        switch error {
        case .unauthorized:
            // 触发重新登录流程
            await handleUnauthorized()
        case .networkUnavailable:
            state = .error(error.localizedDescription)
            // 可选：加载本地缓存
        default:
            state = .error(error.localizedDescription)
        }
    } catch {
        state = .error("未知错误，请稍后重试")
    }
}
```

---

## Mock Service 编写

```swift
// MARK: - Mock（用于测试和 SwiftUI Preview）
final class MockUserService: UserServiceProtocol {
    var fetchResult: Result<User, Error> = .success(.mock)
    var updateResult: Result<User, Error> = .success(.mock)
    var deleteError: Error?

    func fetchUser(id: String) async throws -> User {
        try fetchResult.get()
    }

    func updateUser(_ user: User) async throws -> User {
        try updateResult.get()
    }

    func deleteUser(id: String) async throws {
        if let error = deleteError { throw error }
    }
}

// MARK: - Mock 数据
extension User {
    static let mock = User(
        id: "1",
        name: "测试用户",
        avatarURL: URL(string: "https://example.com/avatar.png"),
        createdDate: Date()
    )
}
```

**Mock 设计要点：**
- 每个方法提供独立的 Result 属性，测试可按需配置成功或失败
- 提供 `.mock` 静态属性用于 Preview 和快速测试
- Mock 类型实现与正式 Service 相同的 Protocol
