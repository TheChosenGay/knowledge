---
tags: [project, fit, rtmpose, coreml, 姿态检测]
created: 2026-05-14
---

# PostureAI — RTMPose 集成方案

## 背景

Apple Vision 内置模型只有 19 个关键点（躯干仅 neck + root），无法满足健身私教对脊柱分段弯曲、手腕角度等精细姿态的需求。

## 方案：分阶段双模型

- **阶段一**（优先）：RTMPose-WholeBody CoreML 替代 Apple Vision 2D 检测，输出 133 个 2D 关键点
- **阶段二**：双模型并行 — RTMPose 133 点 2D + Apple Vision 19 点 3D，通过共有关键点对齐

## 推理 Pipeline

```
摄像头帧 → VNDetectHumanRectanglesRequest（人体框检测）
         → 裁剪 + 缩放到 256×192
         → CoreML RTMPose 推理
         → 133 个 (x, y, confidence)
         → 坐标映射回全帧（Vision 风格，归一化 0-1，左下角原点）
         → BodyJoints
```

## 向后兼容

RTMPoseDetector 默认输出 **legacy 19 点格式**：
- RTMPose body 17 点 → 映射到现有 canonical 命名（如 `left_shoulder` → `left_shoulder_1_joint`）
- `neck_1_joint` = 左右 shoulder 中点（合成）
- `root` = 左右 hip 中点（合成）
- 下游 AngleCalculator / Skeleton3DRenderer / EdgeDetector **零改动**

## 文件变更

### 新建 4 个文件

| 文件 | 位置 | 职责 |
|------|------|------|
| `WholeBodyJointMap.swift` | core/vision/ | 133 点索引 → canonical 名称映射 + legacy 兼容 |
| `RTMPoseDetector.swift` | core/vision/ | CoreML 推理封装，实现 BodyPoseDetectService + PoseDetectService |
| `PoseDetectorConfiguration.swift` | core/vision/ | 检测后端枚举（appleVision / rtmPose） |
| `WholeBodySkeletonConnections.swift` | features/pose_analysis/models/ | 133 点骨架连接定义（Phase 1.5） |

### 修改 3 个文件

| 文件 | 改动 |
|------|------|
| `PoseFrameProcessor.swift` | detector 从硬编码改为协议注入，新增 init(backend:) |
| `RealTimeCameraViewModel.swift` | 新增 poseBackend 属性，传入 PoseFrameProcessor |
| `PoseAnalysisViewModel.swift` | 默认 detector 改为 RTMPoseDetector |

### 不需要改动

AngleCalculator / Skeleton3DRenderer / SkeletonRenderer / EdgeDetector / PosePoint3D / PoseVideoRecorder — 全部通过 String joint name 字典查找，legacy 命名不变。

## 实施顺序

1. Python 端转换模型，确认输出格式
2. 创建 WholeBodyJointMap（纯数据，无依赖）
3. 创建 PoseDetectorConfiguration
4. 创建 RTMPoseDetector（核心推理 pipeline）
5. 修改 PoseFrameProcessor（协议注入）
6. 修改 ViewModel 层
7. 集成测试

## 相关

- [[fit-overview|PostureAI 项目总览]]
- [[../框架/rtmpose-wholebody|RTMPose WholeBody]]
- [[../框架/coreml-integration|iOS CoreML 模型集成]]
