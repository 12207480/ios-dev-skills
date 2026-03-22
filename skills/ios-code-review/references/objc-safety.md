# Objective-C 安全性审查

## 空指针与 nil 消息

```
- [ ] 对 nil 发消息不会崩溃（OC 特性），但结果是否符合预期？
      → [nil stringByAppendingString:@"abc"] 返回 nil，逻辑是否处理了？
      → nil 插入 NSArray / NSDictionary → 崩溃！必须检查
      → [NSNull null] 与 nil 混淆 → JSON 解析常见陷阱
- [ ] 方法返回值为 nil 时，调用方是否做了判断？
      → OC 不像 Swift 有编译期强制，完全靠开发者自觉
- [ ] 使用 nullable / nonnull 标注是否准确？
      → 与 Swift 混编时直接影响可选值类型推断
      → NS_ASSUME_NONNULL_BEGIN / END 包裹后，例外字段是否标注了 nullable？
```

## 集合类型安全

```
- [ ] NSArray / NSDictionary 插入 nil → 崩溃
      → 必须在插入前做 nil 检查
      → 或使用安全封装方法
- [ ] NSMutableArray / NSMutableDictionary 的线程安全？
      → 多线程同时读写 → 崩溃
      → 需要 @synchronized / 串行队列 / 并发队列+barrier
- [ ] 数组越界访问 → 崩溃
      → [array objectAtIndex:index] 前必须检查 index < array.count
      → 或使用 firstObject / lastObject（安全方法）
- [ ] NSDictionary 取值后的类型是否正确？
      → id 类型返回值必须做类型判断再使用
      → if ([obj isKindOfClass:[NSString class]]) { ... }
- [ ] NSMutableArray/Dictionary 是否误用了不可变版本？
      → 从 JSON 解析出来的默认是不可变的
      → 对不可变对象调用 addObject: → 崩溃
```

## 字符串安全

```
- [ ] NSString 的 stringWithFormat 参数类型是否匹配？
      → %@ 对应 id，%d 对应 int，%ld 对应 long，%f 对应 double
      → 类型不匹配 → 崩溃或显示乱码
      → 特别注意 64 位系统上 NSInteger 应使用 %ld 而非 %d
- [ ] NSString 是否可能为 nil？
      → [NSString stringWithFormat:@"%@", nil] → 输出 "(null)"
      → 拼接 URL / 路径时 nil 会导致逻辑错误
- [ ] 使用 isEqualToString: 而非 ==
      → == 比较的是指针地址，不是内容
- [ ] NSMutableString 的线程安全？
```

## 内存管理（MRC 遗留 / ARC 边界）

```
- [ ] Core Foundation 对象是否正确桥接？
      → __bridge：不转移所有权（纯转换）
      → __bridge_retained / CFBridgingRetain：ARC → CF，需要手动 CFRelease
      → __bridge_transfer / CFBridgingRelease：CF → ARC，ARC 接管
      → 桥接方向错误 → 内存泄漏或野指针
- [ ] malloc / calloc 分配的内存是否 free？
- [ ] CGImageRef / CGContextRef / CGColorSpaceRef 等 CG 对象是否 Release？
- [ ] dispatch_semaphore / dispatch_source 在 ARC 下是否正确管理？
- [ ] Block 内部访问 __block 变量是否理解其语义？
      → ARC 下 __block 变量仍可能被 Block 强引用
```

## Block 安全

```
- [ ] Block 作为属性时是否用 copy 修饰？
      → @property (nonatomic, copy) void (^completion)(void);
      → 不用 copy → Block 可能在栈上被销毁，调用时崩溃
- [ ] Block 中是否存在循环引用？
      → __weak typeof(self) weakSelf = self;
      → Block 内部长时间使用需要 __strong typeof(weakSelf) strongSelf = weakSelf;
      → strongSelf 需要判 nil
- [ ] Block 是否可能为 nil？调用 nil Block → 崩溃！
      → 调用前必须判空：if (completion) { completion(); }
      → 或使用 !completion ?: completion();
```

## 属性与内存语义

```
- [ ] 属性修饰符是否正确？
      → strong / weak / copy / assign 选择是否合理？
      → NSString / NSArray / NSDictionary 属性应使用 copy（防止可变子类篡改）
      → delegate 应使用 weak（防止循环引用）
      → 基本类型（int/float/BOOL）使用 assign
- [ ] atomic vs nonatomic 是否有意为之？
      → atomic 只保证属性读写原子性，不保证线程安全
      → 性能敏感场景应使用 nonatomic
- [ ] readonly 属性在 .m 中是否有 readwrite 扩展？
```

## 类型安全

```
- [ ] id 类型使用前是否做了类型检查？
      → isKindOfClass: / respondsToSelector:
      → 不检查直接调用 → 如果类型不对，unrecognized selector 崩溃
- [ ] performSelector: 的参数和返回值是否安全？
      → performSelector 绕过编译器类型检查
      → 优先使用直接调用或 protocol
- [ ] 协议方法调用前是否检查了 respondsToSelector:？
      → @optional 方法不保证实现
      → 不检查直接调用 → unrecognized selector 崩溃
- [ ] 类型强转 (NSString *)obj 是否安全？
      → 无运行时检查，如果类型错误不会报错但行为未定义
      → 应先 isKindOfClass 再转换
```

## KVC / KVO 安全

```
- [ ] KVC setValue:forKey: 的 key 是否拼写正确？
      → key 不存在 → NSUnknownKeyException 崩溃
      → 对非 OC 对象属性（纯 Swift 类型）使用 KVC → 崩溃
- [ ] KVO observer 是否在 dealloc 中移除？
      → 不移除 → 对象释放后通知发送到野指针 → 崩溃
      → 重复移除 → 也会崩溃
      → 配对检查：addObserver 和 removeObserver 必须一一对应
- [ ] KVO 回调中是否正确判断了 keyPath？
      → 多个 KVO 共用一个回调时容易遗漏判断
```

## Swift 混编安全

```
- [ ] OC 头文件的 nullability 标注是否完整？
      → 未标注的属性/参数在 Swift 中被推断为 ImplicitlyUnwrappedOptional (!)
      → Swift 侧使用时可能意外崩溃
- [ ] OC 枚举是否使用 NS_ENUM / NS_OPTIONS？
      → 使用普通 enum → Swift 无法正确桥接
- [ ] OC 类名/方法名是否有 NS_SWIFT_NAME 别名？
      → 提升 Swift 侧的 API 可读性
- [ ] Swift 调用 OC 方法时，NSError ** 参数是否正确处理？
      → 在 Swift 中变为 throws，需要 try-catch
- [ ] OC 泛型（轻量级泛型）是否正确使用？
      → NSArray<NSString *> * 让 Swift 侧得到 [String] 而非 [Any]
```

## 常见问题速查

### 看到 `nil` 插入集合

```
NSArray / NSDictionary 插入 nil → 必崩。

检查所有集合构造和插入操作：
  [NSArray arrayWithObjects:a, b, nil]  → 如果 a 或 b 是 nil，截断而非崩溃
  @[a, b]                               → 如果 a 或 b 是 nil → 崩溃！
  dict[@"key"] = value                   → value 为 nil 等同于 removeObjectForKey（安全）
  [dict setObject:value forKey:key]      → value 或 key 为 nil → 崩溃！

建议方案：
  // 插入前判空
  if (value) {
      [array addObject:value];
  }
  // 或使用 NSNull 占位
  [array addObject:value ?: [NSNull null]];
```

### 看到 `__weak` / `__strong` 配对

```
标准模式：
  __weak typeof(self) weakSelf = self;
  [service fetchWithCompletion:^{
      __strong typeof(weakSelf) strongSelf = weakSelf;
      if (!strongSelf) return;
      [strongSelf doSomething];
  }];

常见错误：
  ❌ 只用 weakSelf 不做 strong → 多行使用中 self 可能中途释放
  ❌ strongSelf 没有判 nil → self 已释放时继续操作
  ❌ Block 外声明了 weakSelf 但 Block 内又直接用了 self → 循环引用仍然存在
```

### 看到 `performSelector:`

```
风险点：
  → 编译器无法检查方法是否存在、参数类型是否正确
  → ARC 下有内存管理警告（#pragma clang diagnostic 压制不是解决方案）

建议方案：
  ✅ 优先直接调用方法
  ✅ 如果是 protocol 方法，先 respondsToSelector 再调用
  ✅ 如果必须动态调用，使用 NSInvocation 或 protocol
  ❌ 避免 performSelector:withObject:afterDelay: 用于时序修复
```

### 看到 `@synchronized` / 锁

```
检查点：
  → 锁的粒度是否合理？（太粗影响性能，太细可能遗漏）
  → 是否有死锁风险？（嵌套 @synchronized 同一对象 → 不会死锁，不同对象注意顺序）
  → @synchronized(nil) → 不起任何作用！锁对象必须非 nil
  → 是否可以用 GCD 串行队列替代？（更清晰且性能更好）
```
