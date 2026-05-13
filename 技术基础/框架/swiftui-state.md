---
tags: [swiftui, combine, 状态管理, 渲染, modifier]
created: 2026-05-11
---

# SwiftUI 状态观察机制

## @Published → UI 刷新 完整链路

```
@Published 赋值
    → objectWillChange.send()     // Combine 信号
        → @StateObject 收到信号   // SwiftUI 自动订阅
            → body 重新计算
                → diff 新旧 View 树
                    → 更新屏幕
                        → onChange 触发链式反应
```

## 各步机制

| 步骤 | 机制 |
|------|------|
| `@Published` | 属性包装器，`willSet` 中自动调 `objectWillChange.send()` |
| `objectWillChange` | `Combine.ObservableObjectPublisher`（`PassthroughSubject<Void, Never>`） |
| `@StateObject` | 创建时自动 `.sink` 订阅 `objectWillChange` |
| `body` 重算 | SwiftUI 重新执行 View 的 body |
| diff | 对比新旧 View 树，只更新变化像素 |
| `onChange` | body 重算后检测属性新旧值变化 |

## 核心理念

> 数据是唯一真相源，UI 是数据的函数。数据变了 UI 自动跟上。

## iOS 17+：@Observable

| | iOS 16 | iOS 17+ |
|---|---|---|
| 写法 | `ObservableObject` + `@Published` | `@Observable` 宏 |
| 底层 | Combine | Observation |
| 精度 | 对象级别 | 属性级别 |
| View 持有 | `@StateObject` | 直接 `var model` |

本项目最低 iOS 16，走 Combine 路线。

## 相关

- [[../编程语言/swift-core|Swift 核心概念]]
- [[../项目/fit-overview|PostureAI 项目]]

---

## body 计算与渲染性能

### body 重计算 ≠ 重建 UI

```
body 重新执行
    │
    └─► 生成新的"视图描述树"（轻量值类型）
            │
            └─► SwiftUI Diff 引擎对比新旧树
                    ├─► 没变化 → 跳过，不碰真实 UI
                    └─► 有变化 → 只更新变化的部分
```

body 返回的是值类型描述，不是真实 UIView，生成极其轻量。

### SwiftUI 三层结构

```
你写的 View（值类型描述）   ← body 在这层计算，廉价
        │
        ▼
   AttributeGraph           ← 内部依赖图，追踪哪些状态影响哪些视图
        │
        ▼
    渲染层（UIKit/Metal）    ← 真正的 UI 对象，尽量少动
```

AttributeGraph 知道哪个 `@State` 影响哪个子视图，状态变化时只重算受影响的子树。

### 需要注意的性能写法

```swift
// ❌ body 里做耗时计算（每次 body 都执行）
var body: some View {
    let sorted = items.sorted { $0.date > $1.date }
    List(sorted) { ... }
}

// ✅ 计算结果放到 ViewModel 维护
// ✅ 拆分子视图，缩小 body 重算范围
```

---

## Modifier

### 本质：包装视图，不是设置属性

```swift
Text("Hello")
    .font(.title)
    .foregroundColor(.red)
    .padding()

// 实际结构（编译期泛型嵌套，运行时零开销）：
// ModifiedContent<ModifiedContent<ModifiedContent<Text, _FontModifier>, _ForegroundColorModifier>, _PaddingLayout>
```

### 顺序影响结果

modifier 从内到外包装，顺序不同结果不同：

```swift
// background 只在文字区域
Text("Hello").background(.yellow).padding()

// background 包含 padding 区域
Text("Hello").padding().background(.yellow)
```

### 自定义 Modifier

```swift
struct CardStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.white)
            .cornerRadius(12)
            .shadow(radius: 4)
    }
}

extension View {
    func cardStyle() -> some View { modifier(CardStyle()) }
}
```

### 环境修饰符 vs 普通修饰符

```swift
// 环境修饰符：自动向下传递，子视图可覆盖
VStack { Text("A"); Text("B") }
.font(.title)  // A、B 都变大

// 普通修饰符：只影响当前视图
VStack { Text("A"); Text("B") }
.onTapGesture { }  // 只有 VStack 整体响应
```

### 常见误区

```swift
// 多个 .padding() 都生效，会叠加
Text("Hello").padding(8).padding(16)  // 总共 24pt

// .frame() 是建议不是约束，不会裁剪内容
// 需要裁剪要加 .clipped()
Text("很长的文字").frame(width: 50).clipped()
```
