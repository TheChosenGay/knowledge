---
tags: [swiftui, gesture, drag, environment, 状态管理]
created: 2026-05-13
---

# SwiftUI 拖拽手势与跨视图状态传递

## 拖拽实现：流畅跟手模式

### 核心模式

```swift
@State private var position = CGPoint(x: 0, y: 0)
@State private var dragStart: CGPoint?

SomeView()
    .offset(x: position.x, y: position.y)
    .simultaneousGesture(
        DragGesture(minimumDistance: 5)
            .onChanged { value in
                if dragStart == nil { dragStart = position }
                let start = dragStart!
                position = CGPoint(
                    x: start.x + value.translation.width,
                    y: start.y + value.translation.height
                )
            }
            .onEnded { _ in dragStart = nil }
    )
```

### 关键点

| 要素 | 说明 |
|------|------|
| `dragStart` | 记录拖拽开始时的位置，避免累加偏移 |
| `value.translation` | 始终是「从拖拽起点算起的总偏移」，不是帧间增量 |
| `simultaneousGesture` | 与 Button 等子视图手势共存，不互相吞噬 |
| `minimumDistance: 5` | 防止点击时误触发拖拽 |

### 三种手势挂载方式对比

| 方式 | 行为 |
|------|------|
| `.gesture()` | 与子视图手势竞争，可能被 Button 吞掉 |
| `.simultaneousGesture()` | 与子视图手势同时识别，拖拽+点击共存 |
| `.highPriorityGesture()` | 优先于子视图手势，子视图点击可能失效 |

含 Button 的可拖拽容器必须用 `simultaneousGesture`。

### @GestureState vs @State

| | @GestureState | @State |
|---|---|---|
| 用途 | 手势进行中的临时值 | 持久化位置 |
| 手势结束后 | 自动重置为初始值 | 保持不变 |
| 适合场景 | 缩放/旋转等临时变换 | 拖拽定位（需保持位置） |
| 已知问题 | 与某些 View 结构配合时可能不触发刷新 | 无 |

实践结论：**拖拽定位用 `@State` 更可靠**，`@GestureState` 适合不需要保持最终值的场景。

---

## 跨视图状态传递：自定义 Environment

### 适用场景

深层子视图需要读取顶层状态，但中间层不关心该状态。比如：全局浮动按钮控制 AI 模型选择，分析页面读取。

### 三步定义

```swift
// 1. 定义 EnvironmentKey
private struct ModelSelectionKey: EnvironmentKey {
    static let defaultValue: Binding<AIModel> = .constant(.deepseek)
}

// 2. 扩展 EnvironmentValues
extension EnvironmentValues {
    var selectedAIModel: Binding<AIModel> {
        get { self[ModelSelectionKey.self] }
        set { self[ModelSelectionKey.self] = newValue }
    }
}

// 3. 注入 & 使用
// 父视图注入
.environment(\.selectedAIModel, $selectedModel)

// 子视图读取
@Environment(\.selectedAIModel) private var modelBinding
```

### 传递 Binding vs 传递值

| 传递内容 | 子视图能力 | 适合 |
|----------|-----------|------|
| 值 (`defaultValue: AIModel`) | 只读 | 纯展示 |
| Binding (`defaultValue: Binding<AIModel>`) | 读写 | 子视图需要修改状态 |

### 与其他方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| `Environment` | 无需逐层传递，解耦 | 需定义 Key |
| `@Binding` 逐层传 | 简单直接 | 中间层被迫声明无关参数 |
| `EnvironmentObject` | 自动按类型查找 | 忘记注入会崩溃；iOS 17 后被 @Observable 取代 |

项目中用 `Environment` 传递 `Binding<AIModel>`：MainTabView 持有状态 → DebugModelButton 通过 @Binding 修改 → PoseAnalysisView 通过 Environment 读取并响应变化。

## 相关

- [[swiftui-state|SwiftUI 状态观察机制]]
- [[../项目/fit-overview|PostureAI 项目]]
