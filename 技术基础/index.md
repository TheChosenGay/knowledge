---
tags: [moc, 技术]
created: 2026-05-11
---

# 技术基础

## 笔记

- [[编程语言/swift-core|Swift 核心概念]] — 自动 init、Actor 隔离、let 线程安全、Combine
- [[框架/swiftui-state|SwiftUI 状态观察]] — @Published → UI 刷新链路
- [[框架/swiftui-drag-and-env|SwiftUI 拖拽与 Environment]] — 流畅拖拽模式、自定义 Environment 传递状态
- [[框架/swiftdata|SwiftData]] — @Model、@Query、ModelContainer、跨 Actor 线程安全、数据迁移
- [[框架/coreml-integration|iOS CoreML 模型集成]] — 转换链路、加载方式、推理对比、硬件加速
- [[框架/rtmpose-wholebody|RTMPose WholeBody]] — 133 点模型细节、对比 Apple Vision、姿态检测能力边界
- [[项目/fit-overview|PostureAI 项目]] — 架构、模块、类型速查
- [[项目/fit-rtmpose-plan|PostureAI RTMPose 集成方案]] — 双模型架构、Pipeline、文件变更、实施顺序
- [[项目/fit-pose-detection-implementation|姿态检测完整实现方案]] — RTMPose架构、PyTorch→CoreML转换、SimCC解码、关节映射、渲染层、帧处理管线
- [[项目/fit-motion-compare|动作对比方案]] — 实时阶段识别 vs DTW录制分析，DTW原理、帧拉伸机制、问题时段定位
- [[项目/fit-ai-voice-coach|AI 实时语音教练]] — STT+DeepSeek流式+TTS管道，姿态上下文注入，延迟优化
- [[项目/fit-food-calories|拍照识别食物热量]] — 识别方案、份量估算、营养数据库选型（含中国食物成分表）
