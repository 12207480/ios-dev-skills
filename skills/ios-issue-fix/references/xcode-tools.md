# Xcode 调试工具速查

## 工具总览

| 问题类型 | 推荐工具 | 入口 |
|---------|---------|------|
| UI 布局异常 | View Debugger | Debug → View Debugging → Capture |
| UI 渲染性能 | Instruments Core Animation | Instruments → Core Animation |
| 主线程违规 | Main Thread Checker | Scheme → Diagnostics（默认开启） |
| 约束冲突 | Console 紫色警告 | Xcode Console 过滤 "constraint" |
| 内存泄漏 | Memory Graph Debugger | Debug Navigator → Memory |
| 野指针访问 | Zombie Objects | Scheme → Diagnostics → Zombie Objects |
| 内存越界 | Address Sanitizer | Scheme → Diagnostics → ASan |
| 数据竞争 | Thread Sanitizer | Scheme → Diagnostics → TSan |
| 网络请求 | Charles / Proxyman | 代理抓包 |
| JSON 解析 | Console + Breakpoint | LLDB `po` 命令 |
| SwiftUI 重建 | `Self._printChanges()` | 代码中添加 |
| 性能热点 | Time Profiler | Instruments → Time Profiler |
| 断点调试 | Xcode Breakpoints | Cmd + 8 |
| 异常捕获 | Exception Breakpoint | Breakpoint Navigator → + |

---

## LLDB 常用命令

### 基础检查

```
po expression           # 打印对象描述（调用 debugDescription）
p expression            # 打印值（不调用 description）
v variableName          # 打印局部变量（最快，无表达式求值）
frame variable          # 显示当前帧所有局部变量
```

### 内存与地址

```
po unsafeBitCast(0x地址, to: MyClass.self)    # 通过地址查看对象
memory read 0x地址                             # 读取原始内存
expression -l objc -O -- [0x地址 description]  # OC 对象查看
```

### 断点增强

```
breakpoint set -n methodName                       # 按方法名设断点
breakpoint set -S selectorName                     # 按 OC selector 设断点
breakpoint set -r "ViewModel.*load"                # 正则匹配方法名
breakpoint modify --condition 'index == 5'         # 条件断点
breakpoint command add                             # 断点触发时自动执行命令
```

### 运行时修改

```
expression someVariable = newValue                       # 修改变量值
expression someView.backgroundColor = UIColor.red        # 改颜色定位视图
thread return                                            # 强制当前函数返回
```

### Swift 并发调试

```
swift task backtrace all       # 查看所有 Task 的调用栈
thread list                    # 列出所有线程
thread backtrace all           # 所有线程调用栈
```

---

## View Debugger 使用技巧

### 启动
Debug → View Debugging → Capture View Hierarchy（或 Debug Bar 的 3D 按钮）

### 排查步骤
1. **旋转 3D 视图** — 检查视图层级是否有重叠/遮挡
2. **选中视图** — 右侧 Inspector 查看 frame/bounds/约束
3. **过滤系统视图** — 底部取消 "Show System Views" 减少噪音
4. **约束问题** — 紫色 = ambiguous layout，黄色 = 约束冲突
5. **检查 hidden/alpha** — 视图存在但不可见时检查

### SwiftUI 视图调试
- SwiftUI 视图在层级中显示为 `_UIHostingView` 的子视图
- 注意 SwiftUI 可能生成额外的包装视图（如 `ModifiedContent`）

---

## Memory Graph Debugger 使用技巧

### 启动
Debug Navigator（Cmd+7） → Memory → 点击相机图标捕获

### 排查循环引用
1. **捕获 Memory Graph** — 在怀疑泄漏的页面退出后捕获
2. **左侧面板过滤** — 输入类名查找应该已释放的对象
3. **如果对象仍存在** — 点击对象，右侧显示引用关系图
4. **追踪引用链** — 找到形成环的引用路径
5. **紫色感叹号** — 表示 Xcode 检测到的潜在泄漏

### 常见泄漏模式
- ViewController dismiss 后仍存在 → 检查闭包/Timer/NotificationCenter
- Closure 持有 self → 需要 `[weak self]`
- Delegate 未声明 weak → `weak var delegate: SomeDelegate?`

---

## Instruments 使用技巧

### Time Profiler（性能热点）
1. Product → Profile（Cmd+I） → 选择 Time Profiler
2. 操作 App 复现卡顿 → 停止录制 → 选中卡顿时间段
3. **Call Tree 设置**（底部）：
   - 勾选 "Separate by Thread" — 区分主线程和后台线程
   - 勾选 "Invert Call Tree" — 显示最耗时的叶子函数
   - 勾选 "Hide System Libraries" — 聚焦自己的代码
4. 展开调用栈 → 找到耗时最高的自定义方法

### Allocations（内存分配）
1. 选择 Allocations 模板 → 操作 App → 观察内存曲线
2. 使用 "Mark Generation" 标记操作前后
3. 比较两次标记之间的新增分配 → 找到未释放对象
4. 反复进出同一页面 → 内存持续增长说明有泄漏

### Core Animation（渲染性能）
1. 关注 FPS — 低于 60 说明掉帧
2. "Color Blended Layers" — 红色区域 = 透明度混合（影响性能）
3. "Color Offscreen-Rendered" — 黄色区域 = 离屏渲染
4. 优化：减少透明视图、避免圆角+阴影同时使用、预渲染图片

---

## Diagnostics 诊断选项

Scheme → Edit Scheme → Run → Diagnostics：

| 选项 | 用途 | 性能影响 |
|------|------|---------|
| Main Thread Checker | 检测非主线程 UI 操作 | 低（默认开启） |
| Thread Sanitizer (TSan) | 检测数据竞争 | 高（5-10x 减速） |
| Address Sanitizer (ASan) | 检测内存越界/use-after-free | 中（2-3x 减速） |
| Undefined Behavior Sanitizer | 检测未定义行为 | 低 |
| Zombie Objects | 检测向已释放对象发消息 | 中（内存增长） |
| Malloc Scribble | 释放内存时填充 0x55 | 低 |
| Guard Malloc | 精确定位内存越界 | 高 |

> **注意：** TSan 和 ASan 不能同时启用。日常开发开启 Main Thread Checker，排查问题时按需开启其他选项。
