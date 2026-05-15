---
tags: [project, fit, ios, design-patterns, architecture]
created: 2026-05-14
---

# Fit 项目 — 良好设计模式

项目中已验证有效的设计模式和编码规范，后续开发应继续遵循。

## 1. 协议抽象 + 单例实现 (Service 层标准模式)

所有 Service 层遵循统一模式：先定义协议（用于依赖注入和可测试性），再提供 final class 单例实现。

```swift
// 协议 — 面向接口编程
protocol PoseDetectService {
    func detectPose(from image: UIImage) async throws -> PosePoints?
}

// 实现 — final class 单例
final class VisionPoseDetector: PoseDetectService {
    nonisolated static let detector = VisionPoseDetector()
    private init() {}
}

// ViewModel 通过 init 注入，支持替换
class PoseAnalysisViewModel {
    init(poseDetector: PoseDetectService? = nil) {
        self.poseDetector = poseDetector ?? VisionPoseDetector.detector
    }
}
```

**关键点：**
- `nonisolated static let` 让单例可从任何隔离域访问
- `private init()` 防止外部实例化
- init 参数有默认值，生产代码无需显式注入

**应用范围：** `PoseDetectService`、`PoseAnalysisService`、`BodyPoseDetectService`、`MultimodalAnalysisService`、`AICoachService`、`DietAnalysisService`、`FitTextToSpeechService`、`HealthKitService`、5 个 DataService

## 2. MVVM 严格隔离

ViewModel 不依赖任何 SwiftUI 视图类型，View 只通过 `@StateObject` / `@Published` 消费状态。

```swift
@MainActor
final class RealTimeCameraViewModel: ObservableObject {
    @Published var detectedJoints: BodyJoints?
    @Published var isDetecting = false

    func startDetection() { /* 纯逻辑 */ }
}

struct RealTimeCameraView: View {
    @StateObject private var viewModel = RealTimeCameraViewModel()
    // 只消费 viewModel 的 @Published 属性
}
```

**关键点：**
- `@MainActor` — 保证 UI 更新在主线程
- `ObservableObject` + `@Published` — 数据驱动 UI 刷新
- ViewModel 文件 `import Combine` 是必需的
- View 不持有业务逻辑，ViewModel 不 import SwiftUI Views

## 3. DataService 协议封装 SwiftData

ViewModel 不直接操作 `ModelContext`，通过 DataService 协议中转：

```swift
protocol UserDataService {
    func fetchProfile(context: ModelContext) throws -> UserProfile?
    func saveProfile(_ profile: UserProfile, context: ModelContext) throws
}
```

**好处：**
- SwiftData API 变更时只需改 DataService 实现
- ViewModel 可注入 mock DataService 进行测试
- 统一的 CRUD 模式（fetch / save / delete）

## 4. 功能层模型隔离底层框架类型

Features 层使用纯 Swift 基础类型，不泄露 Vision / CoreML 框架类型：

```swift
// ✅ 功能层使用 String，不依赖 Vision
struct PosePoint {
    let joint: String       // 不是 VNRecognizedPointKey
    let location: CGPoint   // 归一化 0-1
    let confidence: Float
}

// ✅ 协议也不暴露底层类型
protocol BodyPoseDetectService {
    func detectBodyPose(from sampleBuffer: CMSampleBuffer) async throws -> BodyJoints?
}
```

**好处：** Vision/CoreML 依赖仅限于 `Core/Vision/` 目录，替换检测后端不影响上层。

## 5. 纯计算逻辑使用 enum 静态方法

无状态的纯函数用 enum 命名空间组织：

```swift
enum AngleCalculator {
    static func compute(_ points: [PosePoint], cgImageSize: CGSize) -> Result { ... }
}

enum CoachContextBuilder {
    static func buildDailyContext(profile: UserProfile?, ...) -> CoachContext { ... }
}
```

**好处：** 不可实例化，无副作用，天然线程安全。

## 6. async/await 统一异步模型

全项目使用 Swift Concurrency，不散落 completion handler：

```swift
func detectBodyPose(from sampleBuffer: CMSampleBuffer) async throws -> BodyJoints?
func analyze(angles: PoseAngle) async throws -> AnalysisReport
```

HealthKit 回调通过 `withCheckedThrowingContinuation` 桥接为 async/await。

## 相关

- [[fit-overview|PostureAI 项目概览]]
- [[fit-refactoring-targets|Fit 项目重构目标]]
