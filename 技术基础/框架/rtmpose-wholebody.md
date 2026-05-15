---
tags: [pose-estimation, rtmpose, coreml, 姿态检测]
created: 2026-05-14
---

# RTMPose WholeBody

## 模型概览

RTMPose 是 MMPose 框架下的实时姿态估计模型，WholeBody 变体支持 133 个关键点（COCO-WholeBody 格式）。

## 133 点组成

| 区域 | 点数 | 内容 |
|------|------|------|
| 身体 | 17 | nose, eyes, ears, shoulders, elbows, wrists, hips, knees, ankles |
| 脚部 | 6 | big_toe, small_toe, heel（左右各 3） |
| 面部 | 68 | 面部轮廓和五官特征点 |
| 左手 | 21 | wrist + 每根手指 4 个关节 |
| 右手 | 21 | 同左手 |

## 关键特性

- **Top-down 模型**：需要先检测人体框（bounding box），再对裁剪区域做 pose estimation
- **输入尺寸**：通常 256×192 或 384×288
- **输出格式**：可能是 `[133, 3]` regression 坐标，或 SimCC 格式（两个 1D 数组分别表示 x/y 的概率分布）。取决于导出配置，需用 `coremltools` 确认
- **模型体积**：RTMPose-m 约 30-70MB
- **推理速度**：A14+ 芯片上 CoreML 约 20-30ms/帧

## 对比 Apple Vision

| 维度 | Apple Vision | RTMPose-WholeBody |
|------|-------------|-------------------|
| 关键点数 | 19（躯干仅 neck + root） | 133（含脊柱中间点、手部、脚部） |
| 3D 支持 | 有（iOS 17+） | 无（仅 2D） |
| 依赖 | 零（系统内置） | 需打包 .mlpackage（30-70MB） |
| 精度 | 中等 | 高（尤其遮挡和侧面场景） |
| 自定义 | 不可能 | 可换模型 |

## 姿态检测能力边界

### 任何模型都无法检测

- **肩胛骨收紧**：皮下骨骼运动，无表面标记
- **核心/肌肉发力状态**：属于 EMG 信号范畴
- **足弓细节**：关键点精度不够

### 需要 3D 深度才能判断

- 手肘是否贴紧身体（前后方向）
- 肩膀前探 vs 整体前倾的区分

### 2D + 固定机位可覆盖

- 侧面：深蹲深度、背部挺直、前倾角度、膝盖超伸
- 正面：膝盖内扣、高低肩、身体对称性

## 相关

- [[coreml-integration|iOS CoreML 模型集成]]
- [[../项目/fit-rtmpose-plan|PostureAI RTMPose 集成方案]]
