---
tags: [ai, llm, 语音, 实时, deepseek, swift, 项目]
created: 2026-05-14
---

# AI 实时语音教练方案

姿态检测已完成，本方案在此基础上接入 LLM，实现"打电话式"实时动作指导。

## 整体架构

```
摄像头
  ├─► 姿态检测（已完成）→ 关键点数据 → 文字描述
  │
麦克风
  ├─► STT（已完成）→ 用户语音 → 文字
  │
  └─► DeepSeek 流式 Chat API（拼接姿态上下文）
              ↓
         文字流（按标点切割）
              ↓
         TTS（已完成）→ 实时播放
```

## 为什么用 DeepSeek 而不是 Realtime API

| 方案 | 优点 | 缺点 |
|---|---|---|
| OpenAI Realtime API | 原生语音，延迟最低 | 贵，STT/TTS 不可控 |
| Gemini Live API | 支持直传视频帧 | 国内不稳定 |
| **DeepSeek + 自有 STT/TTS** | STT/TTS 已有，成本低，可控 | 多一跳，延迟略高 |

本项目 STT 和 TTS 已自行实现，直接接 DeepSeek Chat 流式 API 即可，无需 Realtime API。

## 核心实现

### AICoach — 流式对话

```swift
class AICoach {
    private let apiKey = "your-deepseek-api-key"
    private let baseURL = "https://api.deepseek.com/chat/completions"

    // 维护对话历史，保持上下文
    private var messages: [[String: String]] = [
        [
            "role": "system",
            "content": """
            你是一个专业健身教练，正在通过语音实时指导用户动作。
            用户会描述或发来他们当前的姿态数据，你需要给出简短、清晰的口头指导。
            每次回复控制在1-2句话，像真人教练说话一样自然。
            """
        ]
    ]

    func chat(userText: String, onChunk: @escaping (String) -> Void) async throws {
        messages.append(["role": "user", "content": userText])

        var request = URLRequest(url: URL(string: baseURL)!)
        request.httpMethod = "POST"
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body: [String: Any] = [
            "model": "deepseek-chat",
            "messages": messages,
            "stream": true,
            "max_tokens": 100  // 回复短一点，TTS 更快
        ]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (stream, _) = try await URLSession.shared.bytes(for: request)
        var fullReply = ""

        for try await line in stream.lines {
            guard line.hasPrefix("data: "),
                  line != "data: [DONE]" else { continue }

            let jsonStr = String(line.dropFirst(6))
            guard let data = jsonStr.data(using: .utf8),
                  let json = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
                  let choices = json["choices"] as? [[String: Any]],
                  let delta = choices.first?["delta"] as? [String: Any],
                  let content = delta["content"] as? String else { continue }

            fullReply += content
            onChunk(content)  // 每个 chunk 立即回调给 TTS
        }

        messages.append(["role": "assistant", "content": fullReply])
    }
}
```

### VoiceCoachSession — 串联 STT / AI / TTS

```swift
class VoiceCoachSession {
    let ai = AICoach()
    var ttsBuffer = ""

    // STT 识别完成后调用
    func onSpeechRecognized(_ text: String) {
        // 把实时姿态数据拼入上下文
        let poseContext = PoseDetector.shared.currentDescription
        let input = poseContext.isEmpty ? text : "\(text)（当前姿态：\(poseContext)）"

        Task {
            try await ai.chat(userText: input) { chunk in
                DispatchQueue.main.async {
                    self.ttsBuffer += chunk
                    // 遇到标点就触发 TTS，不等全部生成完
                    if chunk.contains("。") || chunk.contains("，") || chunk.contains("！") {
                        self.flushTTS()
                    }
                }
            }
        }
    }

    func flushTTS() {
        guard !ttsBuffer.isEmpty else { return }
        let text = ttsBuffer
        ttsBuffer = ""
        YourTTS.speak(text)  // 替换为项目的 TTS 调用
    }
}
```

### 姿态数据转文字描述

```swift
func poseToText(_ pose: PoseResult) -> String {
    """
    当前姿态：
    - 左膝角度：\(pose.leftKneeAngle)°（标准：90°）
    - 躯干前倾：\(pose.trunkAngle)°（标准：<15°）
    - 左右肩高度差：\(pose.shoulderLevelDiff)cm
    状态：\(pose.overallAssessment)
    """
}
```

## 延迟分析

```
STT（本地）：~100ms
DeepSeek 首字：~300-500ms
TTS 首句（按标点切割）：~100ms
─────────────────────
总感知延迟：约 500-700ms   ← 实时指导可接受
```

**关键优化：按标点切割 TTS**，不等 AI 说完整句，拿到"膝盖弯曲，"就立即播放，延迟从秒级降到 500ms 以内。

## 实时语音 API 横向对比（备查）

| 服务 | 协议 | 特点 |
|---|---|---|
| OpenAI Realtime API | WebSocket | 原生多模态，支持打断，延迟最低 |
| Gemini Live API | WebSocket | 支持直传视频帧，可让 AI 直接看画面 |
| 豆包/火山实时语音 | WebSocket | 国内低延迟 |
| DeepSeek + 自有管道 | HTTP 流式 | STT/TTS 自控，成本低，本项目采用 |

> 若后续想升级为原生 Realtime 方案，Gemini Live 支持直传带骨架叠加的视频帧，可省去文字描述姿态这一步。

## 相关

- [[fit-overview|PostureAI 项目]] — 整体架构
- [[fit-pose-detection-implementation|姿态检测实现]] — 关键点数据来源
