# 内存与线程安全审查

## 循环引用

```
- [ ] 闭包中是否正确使用了 [weak self]？
      → 网络请求回调 → 必须 weak
      → Timer / CADisplayLink 回调 → 必须 weak
      → NotificationCenter 闭包 → 必须 weak
      → Combine sink → 必须 weak
      → DispatchQueue.main.async 中短暂使用 → 可不加
      → UIView.animate 中 → 无需（系统不持有）
- [ ] delegate 是否声明为 weak？
- [ ] [weak self] 后是否正确处理了 self 为 nil 的情况？
      → guard let self else { return } 而非直接 self?.xxx 链式调用过长
```

## 资源释放

```
- [ ] Timer 是否在 deinit / 离开页面时 invalidate？
- [ ] NotificationCenter observer 是否移除？
      → iOS 9+ 系统会自动移除，但显式移除更安全
- [ ] URLSessionTask 是否在页面销毁时 cancel？
- [ ] CADisplayLink 是否 invalidate？
- [ ] Combine 的 AnyCancellable 是否正确存储和释放？
```

## 大对象管理

```
- [ ] UIImage / Data 等大对象是否有必要长时间持有？
- [ ] 缓存是否有上限和清理机制？
- [ ] 收到 memoryWarning 时是否释放非必要资源？
```

## 主线程安全

```
- [ ] 所有 UI 操作是否在主线程？
      → @MainActor 标注或 DispatchQueue.main
      → 网络回调中更新 UI 是否切回主线程？
- [ ] 耗时操作是否避免了主线程？
      → 大量数据解析、图片处理、文件 I/O
```

## 并发安全

```
- [ ] 共享可变状态是否有保护？
      → actor / 串行队列 / 锁
- [ ] async/await 的 Task 是否正确取消？
      → 页面销毁时 task.cancel()
      → 函数内检查 Task.isCancelled
- [ ] Sendable 约束是否满足？
      → 跨并发域传递的类型
- [ ] DispatchQueue 是否有死锁风险？
      → 同步派发到当前队列 → 死锁
      → 嵌套同步调用 → 死锁
```

## 结构化并发安全

```
- [ ] Task 的继承关系是否正确？
      → Task {} 继承调用者的 actor 上下文
      → Task.detached {} 不继承，需要显式标注 @MainActor
      → 优先使用 Task {} 而非 Task.detached {}

- [ ] TaskGroup 中是否正确处理了错误？
      → withThrowingTaskGroup 中一个子任务抛错，其他子任务自动取消
      → 确保所有子任务都被 await（避免子任务泄漏）

- [ ] async let 绑定是否在作用域结束前被 await？
      → 未 await 的 async let 在作用域结束时会被隐式取消并 await
      → 如果子任务可能抛错，隐式 await 会导致错误被静默丢弃

- [ ] Task 取消传播是否正确？
      → 父 Task 取消时，子 Task 自动收到取消信号
      → 但子 Task 需要主动检查 Task.isCancelled 或调用 Task.checkCancellation()
      → 长循环中应定期检查取消状态

- [ ] actor reentrancy 是否考虑？
      → actor 方法中 await 后，actor 状态可能已被其他调用者修改
      → await 前后的状态假设需要重新验证
```

## Core Data 线程安全

```
- [ ] NSManagedObject 是否在正确的上下文线程访问？
- [ ] context.perform {} 是否正确使用？
- [ ] 主上下文与后台上下文是否正确隔离？
- [ ] 合并通知（NSManagedObjectContextDidSave）是否正确处理？
```

## 常见问题速查

### 看到闭包没有 `[weak self]`

```
判断是否需要：
  UIView.animate { } → 不需要（系统不持有闭包）
  DispatchQueue.main.async { } → 通常不需要（短暂持有）
  网络请求回调 → 必须需要
  Timer / CADisplayLink → 必须需要
  Combine sink → 必须需要
  NotificationCenter 闭包 → 必须需要
  存储在属性中的闭包 → 必须需要
  逃逸闭包（@escaping） → 大概率需要

经验法则：
  如果不确定，加上 [weak self] 总是更安全的选择。
```

### 看到 `DispatchQueue.main.asyncAfter`

```
判断意图：
  ✅ 合理场景：动画序列、用户提示延迟消失
  ❌ 不合理场景：修复时序问题（"加个延迟就好了"）

如果是修复时序问题：
  → 要求作者找到正确的生命周期事件或回调时机
  → 不同设备速度不同，延迟方案不可靠
```
