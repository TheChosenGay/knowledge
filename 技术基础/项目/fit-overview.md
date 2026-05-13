---
tags: [project, fit, postureai, ios]
created: 2026-05-09
updated: 2026-05-11
---

# PostureAI（体态AI）

## 基本信息

- **项目名**：fit（PostureAI / 体态AI）
- **类型**：iOS App，SwiftUI + MVVM
- **最低版本**：iOS 16.6
- **入口**：`fitApp.swift` → `MainTabView`

## 架构

```
features/<module>/
├── models/          # 数据模型 + 纯计算
├── view_models/     # @MainActor ObservableObject
└── views/           # SwiftUI View

core/                # 跨模块基础
shared/              # 共享组件 + Token
```

## 配置要点

- `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`
- `PBXFileSystemSynchronizedRootGroup`：新文件自动入 Target

## 模块状态

| 模块 | 状态 |
|------|------|
| A. 拍照与照片选择 | ✅ |
| B. 姿态检测与标注 | ✅ |
| C. AI 体态分析 | ❌ |
| D. 矫正动作推荐 | ❌ |
| E. 历史记录与对比 | ❌ |
| F. 订阅付费墙 | ❌ |

## 核心类型

| 类型 | 职责 |
|------|------|
| `PosePoint` | 骨骼点：joint + 坐标 + 置信度 |
| `PoseAngle` | 5 项体态角度 |
| `PoseDetectService` | 检测协议 |
| `VisionPoseDetector` | Vision 框架实现 |
| `AngleCalculator` | 纯计算引擎 |
| `SkeletonRenderer` | 骨骼标注绘制 |
| `CameraSession` | AVFoundation 封装 |
| `CameraViewModel` | 相机逻辑 |
| `PoseAnalysisViewModel` | 分析编排 |

## 架构原则

1. **协议抽象** — 先协议后实现
2. **Vision 隔离** — Vision 依赖仅在 `VisionPoseDetector.swift`
3. **UIKit 桥接隔离** — UIKit 只在 UIViewRepresentable 文件
4. **async/await 统一** — 不散落 completion handler
5. **数据驱动 UI** — `@Published` → View 自动刷新

## 开发规范

- ViewModel：`@MainActor` + `ObservableObject` + `import Combine`
- 纯计算：enum 静态方法
- 存储层：协议 + Factory

## 待办

- [ ] 模块 C：AI 体态分析
- [ ] 模块 D：矫正动作推荐
- [ ] 模块 E：历史记录与对比
- [ ] 模块 F：订阅付费墙
- [ ] `StorageService` 桩实现替换

## 相关

- [[../编程语言/swift-core|Swift 核心概念]]
- [[../框架/swiftui-state|SwiftUI 状态观察]]
