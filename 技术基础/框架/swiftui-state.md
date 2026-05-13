---
tags: [swiftui, combine, 状态管理]
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
