---
tags: [project, fit, ios, refactoring, technical-debt]
created: 2026-05-14
status: 待重构
---

# Fit 项目重构目标

当前项目存在的架构问题和改进方向，按优先级排列。

## 1. 网络层应引入 HTTPClient 协议抽象

**现状：** `NetworkService` 是 `final class` + 静态单例，所有 AI 服务直接硬依赖它：

```swift
// ❌ 当前：无法 mock，无法测试
let response = try await NetworkService.shared.request(url: ..., body: ...)
```

**目标：** 抽象 HTTP 层为协议，支持注入和 mock：

```swift
// ✅ 目标
protocol HTTPClient {
    func request<T: Decodable>(url: URL, method: String,
        headers: [String: String], body: Data?) async throws -> T
}

final class URLSessionHTTPClient: HTTPClient { /* URLSession 实现 */ }
final class MockHTTPClient: HTTPClient { /* 测试桩 */ }

// AI 服务通过 init 注入
class DeepSeekAICoachService {
    private let httpClient: HTTPClient
    init(httpClient: HTTPClient = URLSessionHTTPClient()) { ... }
}
```

**影响范围：** `NetworkService.swift`、所有 AI Service 文件

## 2. OpenAI Chat Completion 请求/响应类型应提取为共享类型

**现状：** `DeepSeekRequest`/`DeepSeekResponse` 在以下文件中各定义了一份私有副本：
- `AIAnalysisService.swift`
- `AICoachService.swift`
- `MultimodalAnalysisService.swift`
- `DietAnalysisService.swift`
- `TrainingPlanGenerationView.swift`

完全相同的数据结构复制了 5 遍。

**目标：** 提取为 `Core/Network/OpenAITypes.swift`，所有 AI 服务共享：

```swift
// Core/Network/OpenAITypes.swift
struct ChatCompletionRequest: Encodable {
    let model: String
    let maxTokens: Int
    let messages: [Message]
    // ...
}
struct ChatCompletionResponse: Decodable { /* ... */ }
```

**影响范围：** 5 个 AI Service 文件改为 `import` 共享类型

## 3. Markdown 围栏清理应提取为 String 扩展

**现状：** `stripMarkdownCodeBlock` 逻辑在 4 个文件中各自实现，完全相同的代码：

```swift
// ❌ 在 4 个地方重复
private func stripMarkdownCodeBlock(_ text: String) -> String {
    var t = text.trimmingCharacters(in: .whitespacesAndNewlines)
    if t.hasPrefix("```") { ... }
    return t
}
```

**目标：** 扩展为 `String` 方法或在 `Core/Extensions/` 中定义全局函数：

```swift
// Core/Extensions/String+Markdown.swift
extension String {
    func strippingMarkdownCodeBlock() -> String { ... }
}
```

**影响范围：** `AIAnalysisService`、`AICoachService`、`DietAnalysisService`、`TrainingPlanGenerationView`

## 4. API 密钥应从源代码中移除

**现状：** `Secrets.swift` 中 DeepSeek API 密钥以明文硬编码，已提交到 Git 历史：

```swift
// ❌ 密钥泄露
enum Secrets {
    static let deepseekAPIKey = "sk-xxxxxxxxxxxxxxxx"
}
```

**目标：** 使用 `.xcconfig` 文件 + `Info.plist` 管理密钥：

```
// Secrets.xcconfig (gitignored)
DEEPSEEK_API_KEY = sk-xxxxxxxx

// Secrets.swift
enum Secrets {
    static var deepseekAPIKey: String {
        Bundle.main.infoDictionary?["DEEPSEEK_API_KEY"] as? String ?? ""
    }
}
```

**重要：** 旧密钥已在 Git 历史中泄露，需要在 DeepSeek 后台吊销并生成新密钥。

**影响范围：** `Secrets.swift`、`project.pbxproj`（添加 xcconfig 引用）、`.gitignore`

## 5. CoachContextBuilder 应改为协议 + 依赖注入

**现状：** `CoachContextBuilder` 是 enum 静态方法命名空间，硬依赖具体类型，不可注入不可 mock：

```swift
// ❌ 无法替换实现
let ctx = CoachContextBuilder.buildDailyContext(profile: ..., healthData: ..., ...)
```

**目标：** 改为协议，通过 init 注入到 ViewModel：

```swift
// ✅ 可注入可测试
protocol CoachContextBuilding {
    func buildDailyContext(profile: UserProfile?, ...) -> CoachContext
}

struct DefaultCoachContextBuilder: CoachContextBuilding { ... }

class DashboardViewModel {
    private let contextBuilder: CoachContextBuilding
    init(contextBuilder: CoachContextBuilding = DefaultCoachContextBuilder()) { ... }
}
```

**影响范围：** `CoachContextBuilder.swift`、`DashboardView` 的 ViewModel、`WorkoutSessionViewModel`

## 6. WorkoutSessionViewModel 应拆分为更小职责单元

**现状：** 当前约 200 行，同时管理 CameraSession、PoseFrameProcessor、ExerciseFormEvaluator、AI 教练调用、TTS 播报、数据持久化。

**目标：** 拆分为 3 个独立组件：

```
CameraManager          → 摄像头生命周期（已有 CameraSession，只需薄封装）
CoachingEngine         → 动作评估 + AI 反馈调度（节流、去重）
WorkoutSessionViewModel → 协调 CameraManager + CoachingEngine，管理 UI 状态
```

**影响范围：** `WorkoutSessionViewModel.swift`（拆分）、`WorkoutSessionView.swift`（不变）

## 7. ExerciseFormEvaluator 评分阈值需要科学校准

**现状：** 评分阈值是猜测的魔法数字，缺少运动科学依据：

```swift
// ❌ 无依据的阈值
if angle >= 80 && angle <= 170 { return 100 }
```

**目标：** 从运动科学文献或专业教练数据中获取标准角度范围，使评分有据可依。可考虑：
- 参考 NSCA（美国国家体能协会）动作标准
- 与健身教练合作标定数据
- 将阈值提取为可配置的 `ExerciseCalibration` 类型，便于后续调整

**影响范围：** `ExerciseFormEvaluator.swift`

## 8. 项目应建立统一的目录索引

**现状：** `项目/` 下有 6 个 `.md` 文件但没有 `index.md` 导航。

**目标：** 创建 `项目/index.md` 作为文件清单和入口。

## 相关

- [[fit-design-patterns|Fit 项目良好设计模式]]
- [[fit-overview|PostureAI 项目概览]]
- [[../架构设计/ios-service-pattern|iOS Service 层模式]]
