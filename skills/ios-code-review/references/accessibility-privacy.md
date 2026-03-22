# 无障碍与隐私审查

> 深入的无障碍检查和隐私合规审查。基础 UI 规范见 `platform-checklist.md`。

## VoiceOver 检查清单

```
基础标注：
- [ ] 所有可交互元素有 accessibilityLabel
      → 按钮、链接、开关等必须有清晰的标签
      → 纯图标按钮尤其需要标注（如"关闭"、"更多"）
- [ ] accessibilityHint 用于补充说明操作结果
      → "双击打开设置页面"
- [ ] accessibilityTraits 设置正确
      → .button / .header / .link / .adjustable 等
- [ ] 装饰性元素标记为 accessibilityHidden(true)
- [ ] 自定义控件实现 accessibilityValue（如滑块、进度条）

分组与排序：
- [ ] 相关元素是否用 accessibilityElement(children:) 正确分组？
      → 避免 VoiceOver 逐个读取无意义的子元素
- [ ] 自定义的 accessibilitySortPriority 是否合理？
      → 阅读顺序是否符合视觉布局逻辑
- [ ] 模态视图是否正确设置了 accessibilityViewIsModal？
      → 防止 VoiceOver 穿透到背后的视图
```

## SwiftUI 无障碍补充

```
- [ ] 使用 .accessibilityLabel() 而非 Text 作为唯一标签来源
      → Image(systemName: "gear").accessibilityLabel("设置") ✅
      → 纯装饰图片用 .accessibilityHidden(true)

- [ ] .accessibilityAction() 自定义交互
      → 复杂手势控件提供无障碍替代操作
      → .accessibilityAction(.magicTap) 用于主操作快捷触发

- [ ] .accessibilityRepresentation() 简化复杂视图
      → 复杂自定义控件提供简化的无障碍表示
      → 如自定义评分控件 → 用 Slider 表示

- [ ] .accessibilityElement(children: .combine) 合并子元素
      → HStack { Image(...); Text(...) } 合并为一个元素朗读

- [ ] NavigationLink / Button 的标签是否完整？
      → NavigationLink { ... } label: { ... } 中 label 内容即为 VoiceOver 朗读内容
      → 如果 label 只有图标，必须加 .accessibilityLabel

- [ ] List / ForEach 中的行是否有足够的上下文？
      → VoiceOver 逐行朗读时，每行应能独立理解
```

## Dynamic Type 检查清单

```
- [ ] 文本使用 preferredFont(forTextStyle:) 或 .font(.body) 等动态字体
      → 避免硬编码 .font(.system(size: 14))
- [ ] 布局在大字号下不截断、不重叠
      → 使用 Auto Layout 而非固定高度
      → 测试 Accessibility Large Text 设置
- [ ] @ScaledMetric 用于需要随字号缩放的非文本尺寸
      → 图标大小、间距等
      → @ScaledMetric(relativeTo: .body) private var iconSize = 24
```

## Reduce Motion 检查清单

```
- [ ] 动画应检查 UIAccessibility.isReduceMotionEnabled
      → 或使用 @Environment(\.accessibilityReduceMotion)
      → 提供无动画替代（淡入淡出替代滑动）
- [ ] parallax、弹跳等装饰性动画应在 Reduce Motion 下禁用
- [ ] SwiftUI 的 .animation() 是否配合 withAnimation 使用？
      → withAnimation 中可以根据 reduceMotion 选择动画类型
```

## 颜色与对比度

```
- [ ] 不仅靠颜色传达信息（同时使用图标、文字、形状）
      → 例如错误状态不仅标红，还要有错误图标和文字说明
- [ ] 检查 UIAccessibility.isInvertColorsEnabled 下的显示效果
- [ ] 图片考虑设置 accessibilityIgnoresInvertColors
- [ ] 文字与背景的对比度是否满足 WCAG AA 标准？
      → 正文 ≥ 4.5:1，大文字 ≥ 3:1
```

## 权限与隐私

```
- [ ] 新增权限是否有 Info.plist Usage Description？
      → 描述文案是否清晰说明用途？（审核重点）
      → 是否使用了用户能理解的语言？
- [ ] 权限被拒后的降级处理是否完善？
      → 相机/相册/定位等被拒后 UI 是否有引导？
      → 是否提供跳转设置页的选项？
- [ ] 敏感数据（token/密码）是否存储在 Keychain 而非 UserDefaults？
- [ ] 日志中是否泄露了敏感信息？（os_log 的 privacy 参数）
      → os_log("token: %{private}@", token) ✅
      → os_log("token: %@", token) ❌
- [ ] 网络请求是否包含不必要的用户信息？
```

## Privacy Manifest 检查

```
- [ ] PrivacyInfo.xcprivacy 是否需要更新？
      → 新增了 Required Reason API 调用？
      → 新增了第三方 SDK？
      → 数据收集类型是否声明完整？

Required Reason API 类别：
  → File timestamp APIs（NSFileModificationDate 等）
  → System boot time APIs（systemUptime 等）
  → Disk space APIs（volumeAvailableCapacity 等）
  → User defaults APIs（UserDefaults 等）

- [ ] 第三方 SDK 是否提供了自己的 Privacy Manifest？
      → 缺失时需要联系 SDK 提供方或自行补充
- [ ] App Privacy Report 中的数据是否与实际一致？
- [ ] Privacy Manifest 中声明的 tracking domains 是否完整？
      → ATT 框架使用是否正确？
      → SKAdNetwork 配置是否最新？
```

## 常见问题速查

### 看到没有 `accessibilityLabel` 的图标按钮

```
判断影响：
  纯装饰图标（不可交互） → accessibilityHidden(true)
  可交互图标按钮 → 必须添加 accessibilityLabel
  带文字的按钮 → 系统自动使用文字，通常不需要额外标注

建议方案：
  Button { ... } label: {
      Image(systemName: "xmark")
          .accessibilityLabel("关闭")
  }
```

### 看到 `UIAccessibility.post` 通知

```
判断是否合理：
  .announcement → 通知 VoiceOver 用户状态变化（如加载完成）✅
  .screenChanged → 页面切换时聚焦到新内容 ✅
  .layoutChanged → 布局变化后重新聚焦 ✅

常见错误：
  → 频繁发送 announcement 导致 VoiceOver 朗读混乱
  → 忘记在异步操作后发送通知，VoiceOver 用户不知道操作已完成
```
