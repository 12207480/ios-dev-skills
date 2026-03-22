# 包体积优化

## 包体积分析

### Xcode Build Report

```
1. Xcode → Product → Build（Cmd+B）
2. Report Navigator（Cmd+9） → 选择最新的 Build
3. 查看编译和链接的详细信息
```

### 生成 LinkMap 文件

```
Build Settings → Write Link Map File = YES
Build Settings → Path to Link Map File = $(TEMP_DIR)/LinkMap.txt

# LinkMap 文件中可以看到：
# - 各 .o 文件的大小
# - 各 section 的大小
# - 符号表（每个函数/变量的大小）
```

### App Thinning Size Report

```
1. Xcode → Product → Archive
2. Distribute App → Development
3. 勾选 "App Thinning" → 选择 "All compatible device variants"
4. 导出后查看 App Thinning Size Report.txt
   → 显示各设备 variant 的下载大小和安装大小
```

### 第三方分析工具

```bash
# 使用 bloaty 分析二进制段大小
bloaty YourApp.app/YourApp

# 使用 nm 查看符号大小
nm -size -sort YourApp.app/YourApp | tail -20
```

---

## 资源瘦身

### 未使用资源检测

```bash
# 使用 FengNiao（开源工具）检测未使用的资源
# https://github.com/onevcat/FengNiao
fengniao --project /path/to/project

# 手动检查：搜索代码中是否引用了资源名
# 注意动态拼接的资源名无法被静态检测到
```

**检查清单：**
- 删除未使用的图片、音频、视频文件
- 删除未使用的 xib / storyboard
- 删除开发阶段的测试资源
- 检查 Pods/SPM 中是否引入了不需要的资源 bundle

### 图片压缩

| 格式 | 适用场景 | 压缩建议 |
|------|---------|---------|
| **PNG** | 需要透明度、UI 图标 | 使用 pngquant 有损压缩（节省 60-80%） |
| **JPEG** | 照片、背景图 | 质量 75-80% 即可 |
| **WebP** | 通用替代（iOS 14+） | 比 PNG/JPEG 小 25-35% |
| **HEIC** | 照片（iOS 11+） | 比 JPEG 小 50% |
| **SVG / PDF** | 简单图标、单色图 | Asset Catalog 支持 PDF 矢量图 |

```bash
# pngquant 压缩 PNG
pngquant --quality=65-80 --strip *.png

# cwebp 转换为 WebP
cwebp -q 80 input.png -o output.webp
```

### Asset Catalog 优化

```
- 使用 Asset Catalog 管理图片（自动 App Thinning）
- 设置正确的 Scale（1x/2x/3x），避免包含不需要的倍率
- 使用 "Single Scale" PDF 矢量图替代多倍率 PNG（适合简单图标）
- 启用 "Preserve Vector Data" 在大尺寸渲染时保持清晰
```

---

## 代码瘦身

### Strip Symbols

```
Build Settings:
  Strip Linked Product = YES          # Release 默认开启
  Strip Style = All Symbols           # 最大程度 strip
  Deployment Postprocessing = YES     # Archive 时生效
  Strip Swift Symbols = YES           # strip Swift 标准库符号
```

### Dead Code Elimination

```
Build Settings:
  Dead Code Stripping = YES                        # 移除未使用的代码
  Link-Time Optimization = Incremental             # LTO 跨文件优化

# Clang 编译选项
Other C Flags: -fdata-sections -ffunction-sections
# 将每个函数/数据放在独立 section，便于链接器移除未使用的部分
```

### 编译优化级别

| 选项 | 效果 | 适用场景 |
|------|------|---------|
| `-O0` | 无优化 | Debug |
| `-O` / `-O1` | 基本优化 | — |
| `-O2` | 更多优化（默认 Release） | 大多数场景 |
| `-Os` | 优化大小（与 -O2 接近但优先减小体积） | Release |
| `-Osize` | Swift 编译器优化大小 | Release 需要缩减体积时 |
| `-O3` | 最大化速度（可能增大体积） | 计算密集场景 |

```
# Swift 编译优化
Build Settings:
  Swift Optimization Level:
    Debug: No Optimization [-Onone]
    Release: Optimize for Size [-Osize]   # 比 -O 更小

  Compilation Mode:
    Whole Module Optimization = YES        # Release 默认，允许跨文件优化
```

### 减少 ObjC 元数据

```swift
// ❌ 不必要的 @objc 暴露增加元数据大小
@objc class MyHelper: NSObject {
    @objc func utilityMethod() { }
}

// ✅ 纯 Swift 类不需要继承 NSObject
class MyHelper {
    func utilityMethod() { }
}

// Build Settings:
// Swift 3 @objc Inference = Default（不自动推断 @objc）
```

---

## 动态库 vs 静态库对包体积的影响

| 维度 | 动态库（.dylib / .framework） | 静态库（.a / static .framework） |
|------|-------------------------------|--------------------------------|
| **包内体积** | 单独存在于 App Bundle 中 | 合并到主二进制中 |
| **Dead Code Stripping** | 整个动态库都打包 | 只链接实际使用的符号 |
| **启动开销** | dyld 加载和链接 | 无额外启动开销 |
| **体积建议** | 如果只用了库的小部分功能，改为静态库 | 优先选择，便于链接器优化 |

```ruby
# CocoaPods 全部使用静态库
# Podfile 中不使用 use_frameworks!（默认静态库）

# 或指定静态链接
use_frameworks! :linkage => :static
```

---

## App Thinning

### Bitcode（Xcode 16 已废弃）

Xcode 16 起 Bitcode 已被移除，不再需要配置。

### Slicing

系统自动根据设备生成对应的变体：
- 只包含目标设备的 CPU 架构（arm64）
- 只包含目标设备的 GPU 资源（Metal Shader）
- 只包含目标倍率的图片资源（2x 或 3x）

> Asset Catalog 中的资源自动参与 Slicing，散落文件不参与。

### On-Demand Resources (ODR)

```swift
// 适用于游戏关卡、教程内容等非首次启动必须的资源
let request = NSBundleResourceRequest(tags: ["level-5"])
request.beginAccessingResources { error in
    if let error = error {
        print("资源下载失败: \(error)")
        return
    }
    // 使用资源
    let image = UIImage(named: "level5_background")
}

// 使用完毕后释放
request.endAccessingResources()
```

---

## 包体积监控与预算

### 建议预算

| App 类型 | 下载大小建议 | 说明 |
|---------|------------|------|
| 轻量工具 | < 30 MB | 用户对下载速度敏感 |
| 社交/内容 | < 80 MB | 平衡功能和体积 |
| 游戏/复杂应用 | < 200 MB | 超过 200MB 需 Wi-Fi 下载 |

> **Apple 限制：** iOS 13-14 蜂窝网络下载限制为 200MB。iOS 15+ 该限制已放宽，用户可在"设置 → App Store → App 下载"中自行选择"始终允许"来移除蜂窝下载大小限制。但建议仍将下载大小控制在合理范围内，因为大体积下载会增加用户流失率。

### CI 集成监控

```bash
# 在 CI 中记录每次构建的包大小
# Archive → 导出 IPA → 记录大小

xcrun altool --validate-app -f YourApp.ipa -t ios
# 或使用 App Store Connect API 获取 Thinning 后大小

# 设置阈值告警
MAX_SIZE_MB=80
CURRENT_SIZE_MB=$(stat -f%z YourApp.ipa | awk '{print int($1/1024/1024)}')
if [ $CURRENT_SIZE_MB -gt $MAX_SIZE_MB ]; then
    echo "⚠️ 包体积超标: ${CURRENT_SIZE_MB}MB > ${MAX_SIZE_MB}MB"
    exit 1
fi
```

---

## 包体积优化 checklist

```
- [ ] 是否分析了包体积构成（资源 vs 代码 vs 框架）？
- [ ] 是否检测并删除了未使用的资源文件？
- [ ] 图片是否使用了合适的格式和压缩？
- [ ] 是否使用 Asset Catalog 管理图片资源？
- [ ] Release 构建是否开启了 Strip Symbols？
- [ ] 是否开启了 Dead Code Stripping 和 LTO？
- [ ] Swift 优化级别是否设置为 -Osize？
- [ ] 是否减少了不必要的 @objc 暴露？
- [ ] 依赖库是否优先使用静态链接？
- [ ] 是否有 CI 集成的包体积监控和预算？
- [ ] 大型资源是否考虑使用 On-Demand Resources？
```

## 常见问题速查

```
看到 IPA 包突然增大
  → 原因：新增依赖库 / 引入大资源文件 / Debug 符号未 strip
  → 优化：检查 git diff，定位新增内容，确认 Release 配置正确

看到资源文件占包体 > 60%
  → 原因：图片未压缩 / 未使用的资源 / 音视频未优化
  → 优化：压缩图片 + 删除无用资源 + 音视频转高效编码

看到二进制文件过大
  → 原因：未开启 LTO / 过多 ObjC 元数据 / 未 strip
  → 优化：开启 LTO + -Osize + Strip All Symbols

看到依赖库占比过高
  → 原因：引入了大型 SDK 但只用了小部分功能
  → 优化：评估是否可替换为轻量方案或自行实现
```
