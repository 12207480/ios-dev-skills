# Swift 安全性审查

## 可选值安全

```
- [ ] 无 force unwrap（!）
      → 唯一例外：@IBOutlet（系统约定）
- [ ] 无 force cast（as!）
      → 应使用 as? + guard/if let
- [ ] 无 force try（try!）
      → 应使用 do-catch 或 try?
- [ ] guard let / if let 的 else 分支处理是否合理？
      → 不应静默忽略，至少记录日志
- [ ] 可选链（?.）的末端是否正确处理了 nil 情况？
```

## 错误处理

```
- [ ] catch 块是否有实际处理？（禁止空 catch {}）
- [ ] 错误信息是否足够用于排查？（至少记录 error.localizedDescription）
- [ ] Result 类型的 .failure 分支是否处理？
- [ ] async throws 函数的调用方是否正确 try await？
```

## 类型安全

```
- [ ] 泛型约束是否足够？
- [ ] Any / AnyObject 是否可以替换为具体类型？
- [ ] @objc 和 #selector 是否可以避免？（优先 Swift 原生方式）
- [ ] 枚举是否处理了未来可能新增的 case？（考虑 @unknown default）
```

## Swift 6 并发安全检查

### `@Sendable` 闭包约束

```
- [ ] 跨并发域传递的闭包是否标注 @Sendable？
- [ ] @Sendable 闭包内是否捕获了非 Sendable 类型？
      → 捕获 UIView、UIViewController 等非 Sendable 类型会编译报错
- [ ] 逃逸闭包传递给 actor 方法时，是否满足 Sendable 约束？
```

### Actor Isolation 正确性

```
- [ ] actor 的属性和方法是否只在 actor 上下文中同步访问？
- [ ] 从外部调用 actor 方法是否使用了 await？
- [ ] actor 内部是否有跨 actor 边界传递非 Sendable 数据？
- [ ] actor 的 deinit 中是否访问了隔离属性？（Swift 6 中 deinit 是 nonisolated）
```

### `nonisolated` 使用场景

```
- [ ] nonisolated 方法是否确实不需要访问 actor 隔离状态？
- [ ] nonisolated 方法访问的属性是否为 let 常量或 Sendable 类型？
- [ ] 协议 conformance 中 nonisolated 标注是否正确？
      → 例如 Hashable/Equatable 的方法通常需要 nonisolated
```

### 全局变量的并发安全

```
- [ ] 全局 let 常量 → 安全，无需额外标注
- [ ] 全局 var 变量 → 必须标注 @MainActor 或 nonisolated(unsafe)
      → @MainActor：变量只在主线程访问时使用
      → nonisolated(unsafe)：开发者自行保证线程安全（谨慎使用）
- [ ] static var 属性 → 同全局 var，需要隔离标注
- [ ] 单例模式中的 shared 实例 → 建议声明为 actor 或用 @MainActor 保护
```

### `Sendable` Conformance 检查

```
- [ ] 跨并发域传递的自定义类型是否遵循 Sendable？
- [ ] struct（所有存储属性为 Sendable）→ 可自动推断或显式声明
- [ ] enum（所有关联值为 Sendable）→ 可自动推断或显式声明
- [ ] class → 必须为 final class 且所有存储属性为 let + Sendable
      → 或使用 @unchecked Sendable（需自行保证线程安全）
- [ ] 使用 @unchecked Sendable 时，是否有充分的线程安全保障？
      → 内部是否使用了锁 / actor / 队列保护？
```

## 常见问题速查

### 看到 `!` (force unwrap)

```
判断是否合理：
  @IBOutlet → ✅ 系统约定，可接受
  测试代码中 → ✅ 测试失败即暴露问题，可接受
  其他业务代码 → ❌ 必须要求改为安全处理

建议方案：
  guard let value = optionalValue else {
      // 记录日志 + 合理的降级处理
      return
  }
```

### 看到空 `catch {}`

```
永远不接受空 catch。即使当前不需要处理，也至少要记录日志：

do {
    try someOperation()
} catch {
    logger.error("操作失败: \(error.localizedDescription)")
    // 或者至少：assertionFailure("意外错误: \(error)")
}
```
