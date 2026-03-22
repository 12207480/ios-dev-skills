# 网络请求排查指南

## HTTP 状态码排查决策树

### 4xx 客户端错误

| 状态码 | 含义 | 排查方向 |
|-------|------|---------|
| 400 Bad Request | 请求参数错误 | 抓包检查请求 body/query 格式，对比 API 文档 |
| 401 Unauthorized | 未认证 | 检查 Token 是否过期/缺失，检查 Authorization Header 格式 |
| 403 Forbidden | 无权限 | 检查用户角色/权限配置，检查是否需要额外的请求头 |
| 404 Not Found | 资源不存在 | 检查 URL 拼接（path/query 编码）、API 版本号、环境地址 |
| 408 Request Timeout | 请求超时 | 检查 URLSession 的 `timeoutIntervalForRequest` 配置 |
| 422 Unprocessable | 业务校验失败 | 检查请求参数的业务规则（格式/范围/依赖） |
| 429 Too Many Requests | 频率限制 | 检查请求频率，实现退避重试（Exponential Backoff） |

### 5xx 服务端错误

| 状态码 | 含义 | 客户端应对 |
|-------|------|----------|
| 500 Internal Server Error | 服务端异常 | 记录请求参数，联系后端排查，客户端展示友好错误提示 |
| 502 Bad Gateway | 网关错误 | 通常为后端部署/网关问题，客户端可重试 |
| 503 Service Unavailable | 服务不可用 | 检查是否在维护期，客户端实现重试逻辑 |
| 504 Gateway Timeout | 网关超时 | 后端处理过慢，客户端可适当延长超时或重试 |

### 网络层错误（非 HTTP）

| NSURLErrorDomain Code | 含义 | 排查方向 |
|----------------------|------|---------|
| -1001 (Timeout) | 请求超时 | 检查网络环境，调整 `timeoutInterval` |
| -1003 (CannotFindHost) | DNS 解析失败 | 检查域名是否正确，是否有 DNS 劫持 |
| -1004 (CannotConnectToHost) | 无法连接 | 检查服务器是否在线，端口是否开放 |
| -1009 (NotConnectedToInternet) | 无网络 | 检查设备网络状态，展示离线提示 |
| -1200 (SecureConnectionFailed) | SSL 握手失败 | 检查证书有效性、ATS 配置、SSL Pinning |
| -999 (Cancelled) | 请求被取消 | 检查是否重复取消，页面销毁时的 Task 管理 |

---

## Codable 解析排查

### 常见解析失败原因

| 错误类型 | 典型报错 | 修复方向 |
|---------|---------|---------|
| 字段名不匹配 | `keyNotFound` | 添加 `CodingKeys` 映射，或使用 `keyDecodingStrategy = .convertFromSnakeCase` |
| 类型不匹配 | `typeMismatch` | 后端返回 String 但声明为 Int → 自定义解码或使用兼容类型 |
| 值为 null 但声明非可选 | `valueNotFound` | 将属性改为 `Optional`，或提供 `decodeIfPresent` + 默认值 |
| 日期格式不匹配 | `typeMismatch` (Date) | 配置 `dateDecodingStrategy`（ISO8601 / 自定义 formatter / 时间戳） |
| 嵌套结构变化 | `keyNotFound` | 后端调整了 JSON 结构，更新 Model 层级 |
| 枚举值不匹配 | `dataCorrupted` | 后端新增枚举值，添加 `default` case 或自定义 `init(from:)` |

### 调试技巧

```swift
// 1. 捕获详细错误信息
do {
    let model = try JSONDecoder().decode(MyModel.self, from: data)
} catch let DecodingError.keyNotFound(key, context) {
    print("缺少字段: \(key.stringValue), 路径: \(context.codingPath)")
} catch let DecodingError.typeMismatch(type, context) {
    print("类型不匹配: 期望 \(type), 路径: \(context.codingPath)")
} catch {
    print("解码失败: \(error)")
}

// 2. 先解析为 [String: Any] 查看原始结构
if let json = try? JSONSerialization.jsonObject(with: data) {
    print(json)
}
```

### 防御性 Codable 设计

```swift
// 枚举兜底：防止后端新增值导致解析失败
enum Status: String, Codable {
    case active, inactive, pending
    case unknown  // 兜底值

    init(from decoder: Decoder) throws {
        let value = try decoder.singleValueContainer().decode(String.self)
        self = Status(rawValue: value) ?? .unknown
    }
}

// 可选值 + 默认值：字段可能缺失时
struct User: Codable {
    let name: String
    let avatar: String?          // 允许缺失
    let score: Int               // 必须存在

    // 使用 decodeIfPresent 提供默认值
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        name = try container.decode(String.self, forKey: .name)
        avatar = try container.decodeIfPresent(String.self, forKey: .avatar)
        score = try container.decodeIfPresent(Int.self, forKey: .score) ?? 0
    }
}
```

---

## SSL Pinning 问题

### 常见场景

| 现象 | 原因 | 解决 |
|------|------|------|
| 开发环境请求全部失败 | SSL Pinning 阻止了自签名证书 | 开发环境关闭 Pinning 或添加调试证书 |
| 证书更新后线上请求失败 | 客户端 Pin 了旧证书 | Pin 公钥（SPKI）而非证书本身，或提前内置新证书 |
| 抓包工具无法拦截请求 | SSL Pinning 拒绝代理证书 | 调试时临时关闭 Pinning（仅 Debug 构建） |
| ATS 报错 | Info.plist 缺少 ATS 配置 | 配置 `NSAppTransportSecurity`，生产环境避免全局关闭 |

### Pin 公钥 vs Pin 证书

- **Pin 证书**：证书更新后必须发版 → 不推荐
- **Pin 公钥（SPKI）**：证书更新但公钥不变即可 → 推荐
- **Pin 根证书**：最灵活，但安全性略低

---

## 抓包调试技巧

### Charles / Proxyman 配置

1. **安装证书**：Charles → Help → SSL Proxying → Install Charles Root Certificate on iOS Device
2. **信任证书**：设置 → 通用 → 关于本机 → 证书信任设置 → 启用对应证书
3. **iOS 16+ 额外步骤**：设置 → Wi-Fi → 当前网络 → 配置代理 → 手动 → 填写电脑 IP 和端口
4. **启用 SSL Proxy**：Charles → Proxy → SSL Proxying Settings → 添加目标域名

### 抓包排查清单

| 检查项 | 确认内容 |
|-------|---------|
| 请求 URL | 域名/路径/query 参数是否正确 |
| 请求 Method | GET/POST/PUT/DELETE 是否与 API 文档一致 |
| 请求 Headers | Content-Type / Authorization / 自定义 Header |
| 请求 Body | JSON 字段名/类型/格式是否与文档一致 |
| 响应 Status | 状态码是否符合预期 |
| 响应 Body | JSON 结构是否与 Model 定义一致 |
| 响应时间 | 是否超时或异常缓慢 |

### 模拟网络异常

| 工具 | 用途 |
|------|------|
| Network Link Conditioner | 模拟弱网/延迟/丢包（设置 → 开发者） |
| Charles → Throttle | 限速特定域名请求 |
| Charles → Map Local | 用本地 JSON 替代接口返回（模拟异常数据） |
| Charles → Breakpoints | 拦截并修改请求/响应内容 |

---

## 重试与错误恢复模式

### 指数退避重试

```swift
func fetchWithRetry<T: Decodable>(
    request: URLRequest,
    maxRetries: Int = 3,
    type: T.Type
) async throws -> T {
    var lastError: Error?
    for attempt in 0..<maxRetries {
        do {
            let (data, response) = try await URLSession.shared.data(for: request)
            guard let httpResponse = response as? HTTPURLResponse,
                  200..<300 ~= httpResponse.statusCode else {
                throw NetworkError.invalidResponse
            }
            return try JSONDecoder().decode(T.self, from: data)
        } catch {
            lastError = error
            // 非幂等请求（POST）不重试
            guard request.httpMethod == "GET" else { throw error }
            // 4xx 客户端错误不重试（除 408/429）
            if let urlError = error as? URLError,
               urlError.code == .cancelled { throw error }
            // 指数退避：1s → 2s → 4s
            let delay = pow(2.0, Double(attempt))
            try await Task.sleep(for: .seconds(delay))
        }
    }
    throw lastError!
}
```

### 可重试 vs 不可重试判断

| 可重试 | 不可重试 |
|-------|---------|
| 网络超时 (-1001) | 请求被取消 (-999) |
| 服务端 5xx | 客户端 4xx（除 408/429） |
| DNS 解析失败 (-1003) | SSL 证书错误 |
| 连接中断 | 认证失败 (401) |
