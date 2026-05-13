---
tags: [swift, 编程语言, actor, combine]
created: 2026-05-11
---

# Swift 核心概念

## 1. 自动生成 Memberwise Initializer

没有写 `init` 的 struct/class，Swift 自动生成。参数 = 所有没有默认值的存储属性。

```swift
struct PosePoint {
    let joint: String      // 没有 = xxx → init 参数
    let location: CGPoint
    let confidence: Float
}
// 自动生成: init(joint: String, location: CGPoint, confidence: Float)
```

有默认值的属性不参与 init：

```swift
class PoseAnalysisViewModel: ObservableObject {
    let image: UIImage              // 必传
    let poseDetector: PoseDetectService = VisionPoseDetector.detector  // 可省略
    @Published var state: AnalysisState = .loading  // 不参与 init
}
```

> 规则：有 `= 初始值` 的属性不出现在 init 参数里。`@State`、`@StateObject` 给了初始值的也不算。

---

## 2. Actor 隔离：MainActor 与 nonisolated

### 错误信息

`Main actor-isolated static property can not be referenced from a nonisolated context`

### 原因

项目配置 `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`，在非 MainActor 上下文访问其属性时报错。

### 修复

```swift
nonisolated static let detector = VisionPoseDetector()
```

`nonisolated` = "我不属于任何 Actor，随便访问"。

---

## 3. let 常量与线程安全

不是语言要求，是语义自然安全：

- `let` 初始化后不可变 → 多线程读永远不冲突
- 线程安全问题只发生在 `var` 可变状态上
- 一个 class 线程安全的充要条件：**无共享可变状态**

---

## 4. Combine 与 ObservableObject

iOS 16 上 `ObservableObject` + `@Published` 确实在用 Combine：

- `objectWillChange` → `Combine.ObservableObjectPublisher`
- `@Published` → Combine 的 `PassthroughSubject`
- 需显式 `import Combine`

|     | iOS 16                            | iOS 17+         |
| --- | --------------------------------- | --------------- |
| 写法  | `ObservableObject` + `@Published` | `@Observable` 宏 |
| 底层  | **Combine**                       | Observation 框架  |
| 精度  | 对象级别                              | 属性级别            |

## 相关

- [[../框架/swiftui-state|SwiftUI 状态观察]]
- [[../项目/fit-overview|PostureAI 项目]]
