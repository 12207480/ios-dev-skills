# ObjC → Swift 迁移

## 迁移策略决策树

```
项目现状评估：
├── ObjC 代码占比 > 80%
│   └── 渐进迁移（新功能用 Swift，旧代码按需迁移）
├── ObjC 代码占比 30-80%
│   └── 模块级迁移（按模块优先级逐步迁移）
├── ObjC 代码占比 < 30%
│   └── 可考虑集中迁移（评估后决定）
└── 共通原则：
    ├── 永远不做"全量一次性迁移"
    ├── 每次迁移一个文件或一个类
    └── 每次迁移都必须编译通过 + 测试通过
```

### 迁移优先级排序

| 优先级 | 迁移对象 | 原因 |
|--------|---------|------|
| 高 | 频繁修改的文件 | 减少日常开发成本 |
| 高 | 纯数据模型（Model） | 依赖少，迁移风险低 |
| 中 | 工具类/扩展 | 独立性强，影响面可控 |
| 中 | Service/Manager 层 | 需要处理桥接，但逻辑独立 |
| 低 | ViewController | 依赖多，迁移成本高 |
| 低 | 核心基础设施 | 影响面大，需充分测试 |

---

## 文件级迁移流程

### 标准流程

```
1. 选择要迁移的 ObjC 文件（.h + .m）
2. 阅读理解现有代码（接口 + 实现 + 调用方）
3. 创建对应的 Swift 文件
4. 逐方法翻译（保持接口不变）
5. 更新桥接头文件
6. 编译 → 修复桥接问题
7. 运行测试 → 修复行为差异
8. 全局搜索确认所有调用方正常
9. 删除原 ObjC 文件
10. 编译 → 测试 → 提交
```

### 迁移单个类的详细步骤

```swift
// === ObjC 原始代码 ===
// UserService.h
@interface UserService : NSObject
@property (nonatomic, strong, readonly) User *currentUser;
- (void)loginWithEmail:(NSString *)email
              password:(NSString *)password
            completion:(void (^)(User * _Nullable user, NSError * _Nullable error))completion;
- (void)logout;
@end

// === Swift 迁移后 ===
class UserService {
    private(set) var currentUser: User?

    func login(email: String, password: String) async throws -> User {
        // 迁移逻辑
    }

    func logout() {
        currentUser = nil
    }
}

// 如果仍有 ObjC 调用方，需要保留 ObjC 兼容性：
@objc class UserService: NSObject {
    @objc private(set) var currentUser: User?

    // 为 ObjC 调用方保留 completion handler 版本
    @objc func login(email: String, password: String,
                     completion: @escaping (User?, Error?) -> Void) {
        Task {
            do {
                let user = try await login(email: email, password: password)
                completion(user, nil)
            } catch {
                completion(nil, error)
            }
        }
    }

    // Swift 调用方使用 async 版本
    func login(email: String, password: String) async throws -> User {
        // ...
    }
}
```

---

## 常见转换模式

### Delegate → Closure / Combine

```swift
// === ObjC Delegate 模式 ===
@protocol ImagePickerDelegate <NSObject>
- (void)imagePicker:(ImagePicker *)picker didSelectImage:(UIImage *)image;
- (void)imagePickerDidCancel:(ImagePicker *)picker;
@end

// === Swift Closure 模式 ===
class ImagePicker {
    var onImageSelected: ((UIImage) -> Void)?
    var onCancel: (() -> Void)?
}

// === Swift Combine 模式 ===
class ImagePicker {
    enum Event {
        case selected(UIImage)
        case cancelled
    }
    let eventPublisher = PassthroughSubject<Event, Never>()
}
```

### NSNotification → NotificationCenter Typed

```swift
// === ObjC ===
// 发送
[[NSNotificationCenter defaultCenter]
    postNotificationName:@"UserDidLogin"
    object:nil
    userInfo:@{@"userId": userId}];

// 监听
[[NSNotificationCenter defaultCenter]
    addObserver:self
    selector:@selector(handleLogin:)
    name:@"UserDidLogin"
    object:nil];

// === Swift ===
extension Notification.Name {
    static let userDidLogin = Notification.Name("UserDidLogin")
}

// 发送
NotificationCenter.default.post(
    name: .userDidLogin,
    object: nil,
    userInfo: ["userId": userId]
)

// 监听（Combine 方式）
NotificationCenter.default.publisher(for: .userDidLogin)
    .compactMap { $0.userInfo?["userId"] as? String }
    .sink { userId in
        // 处理登录
    }
    .store(in: &cancellables)
```

### KVO → Combine / @Observable

```swift
// === ObjC KVO ===
[self.player addObserver:self
              forKeyPath:@"status"
                 options:NSKeyValueObservingOptionNew
                 context:nil];

// === Swift Combine ===
player.publisher(for: \.status)
    .sink { status in
        // 处理状态变化
    }
    .store(in: &cancellables)

// === Swift @Observable (iOS 17+) ===
@Observable
class Player {
    var status: PlayerStatus = .idle
}
```

### Block → Closure

```swift
// === ObjC Block ===
typedef void (^CompletionHandler)(NSData * _Nullable data, NSError * _Nullable error);
- (void)fetchDataWithCompletion:(CompletionHandler)completion;

// === Swift Closure ===
func fetchData(completion: @escaping (Result<Data, Error>) -> Void)

// === Swift async/await（推荐） ===
func fetchData() async throws -> Data
```

### NSArray / NSDictionary → 类型化集合

```swift
// === ObjC ===
@property (nonatomic, strong) NSArray *items;          // 无类型
@property (nonatomic, strong) NSDictionary *userInfo;  // 无类型

// === Swift ===
var items: [Item] = []                      // 强类型数组
var userInfo: [String: Any] = [:]           // 类型化字典
// 或更好的方式：
var userInfo: UserInfo                      // 使用自定义结构体
```

### 错误处理 NSError → Swift Error

```swift
// === ObjC ===
- (BOOL)saveData:(NSData *)data error:(NSError **)error {
    if (!data) {
        *error = [NSError errorWithDomain:@"AppDomain"
                                     code:100
                                 userInfo:@{NSLocalizedDescriptionKey: @"数据为空"}];
        return NO;
    }
    return YES;
}

// === Swift ===
enum DataError: LocalizedError {
    case emptyData
    case saveFailed(underlying: Error)

    var errorDescription: String? {
        switch self {
        case .emptyData: return "数据为空"
        case .saveFailed(let error): return "保存失败: \(error.localizedDescription)"
        }
    }
}

func save(data: Data) throws {
    guard !data.isEmpty else {
        throw DataError.emptyData
    }
    // ...
}
```

---

## 桥接头文件管理

### 基本规则

```
项目名-Bridging-Header.h:
  - 导入 Swift 需要访问的 ObjC 头文件
  - 每迁移一个文件，从这里移除对应的 #import
  - 最终目标：桥接头文件为空并删除

项目名-Swift.h（自动生成）:
  - Xcode 自动生成，ObjC 代码通过它访问 Swift 类
  - 不要手动编辑
  - Swift 类必须标注 @objc 或继承自 NSObject 才会出现在这里
```

### 迁移期间的桥接管理

```
迁移文件 A（ObjC → Swift）：
1. A 被 ObjC 文件 B、C 依赖
   → A.swift 中相关类标注 @objc
   → B.m 和 C.m 导入 "项目名-Swift.h"

2. A 依赖 ObjC 文件 D
   → 在桥接头中保留 D 的 #import
   → 或将 D 也迁移到 Swift

3. A 迁移完成
   → 从桥接头中移除 A 的 #import
   → 删除 A.h 和 A.m
```

---

## 混编期间的类型桥接

### NS_SWIFT_NAME

```objc
// 为 Swift 提供更 Swift 风格的名称
typedef NS_ENUM(NSInteger, NetworkErrorCode) {
    NetworkErrorCodeTimeout NS_SWIFT_NAME(timeout),
    NetworkErrorCodeNoConnection NS_SWIFT_NAME(noConnection),
} NS_SWIFT_NAME(NetworkError.Code);
```

### NS_REFINED_FOR_SWIFT

```objc
// ObjC 头文件中标记，Swift 侧重新封装
- (NSArray *)fetchItemsWithType:(NSInteger)type NS_REFINED_FOR_SWIFT;

// Swift 扩展中提供更好的接口
extension ItemManager {
    func fetchItems(type: ItemType) -> [Item] {
        return __fetchItems(with: type.rawValue) as? [Item] ?? []
    }
}
```

### Nullability 注解

```objc
// 迁移前必须为 ObjC 接口添加 nullability 注解
NS_ASSUME_NONNULL_BEGIN

@interface UserService : NSObject
@property (nonatomic, strong, readonly) User *currentUser;     // nonnull
- (nullable User *)userWithId:(NSString *)userId;              // nullable 返回值
- (void)updateUser:(User *)user
         completion:(nullable void (^)(NSError * _Nullable error))completion;
@end

NS_ASSUME_NONNULL_END
```

---

## 迁移风险评估 Checklist

```
文件级风险评估：
- [ ] 该文件有多少调用方？（> 10 个调用方 = 高风险）
- [ ] 该文件是否涉及 C/C++ 混编？（是 = 需要额外桥接）
- [ ] 该文件是否使用了 runtime 特性？（method swizzling / associated objects）
- [ ] 该文件的测试覆盖率如何？（无测试 = 先补测试再迁移）
- [ ] 该文件是否有 Category？（需要转为 Swift extension）

项目级风险评估：
- [ ] CI/CD 是否支持 Swift + ObjC 混编？
- [ ] 编译时间是否可接受？（Swift 编译通常比 ObjC 慢）
- [ ] 团队是否熟悉 Swift？（迁移后的代码需要团队能维护）
- [ ] 第三方 ObjC 库是否有 Swift 替代？
- [ ] 最低支持系统版本是否允许使用目标 Swift 特性？

迁移完成验证：
- [ ] 所有公共接口行为等价
- [ ] 内存管理无变化（ARC 行为一致）
- [ ] 线程安全性无变化
- [ ] 错误处理路径完整
- [ ] 所有调用方编译通过
- [ ] 完整测试套件通过
- [ ] 桥接头文件已更新
```
