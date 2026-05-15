---
tags: [project, fit, pose-detection, 3d, videopose3d, coreml]
created: 2026-05-14
status: 设计中
---

# VideoPose3D — 2D 到 3D 姿态提升方案

在 RTMPose（2D CNN）的基础上叠加 VideoPose3D（时序 CNN），实现完整的 3D 姿态管线。

## 动机

RTMPose 133 点是纯 2D 的，缺少深度信息，以下场景被阻塞：

| 需要 3D 的判断 | 2D 视角 | 为什么 2D 不够 |
|---|---|---|
| 手肘是否贴紧身体 | 正面 | 前后方向深度无法感知 |
| 肩膀前探 vs 整体前倾 | 侧面 | 两者在侧面 2D 投影相似 |
| 膝盖是否在脚踝正上方 | 正面 | 前后偏移不可见 |
| 杠铃轨迹是否垂直 | 侧面 | 需要 3D 空间偏移量 |

## VideoPose3D 原理

**两阶段管道：**

```
视频帧 → [第一阶段] RTMPose → 133 个 2D 关键点 (x, y)
           ↓
       [第二阶段] VideoPose3D → 时序空洞卷积 → 133 个 3D 关键点 (x, y, z)
```

第二阶段不做像素级推理，只在 2D 坐标序列上做时序建模，利用多帧的运动信息恢复深度。

### 为什么是 CNN 不是 Transformer

VideoPose3D 使用**时序空洞卷积**（Temporal Dilated Convolution），架构是 Conv1D + BN + ReLU + Residual：

```
2D 姿态序列 (T 帧 × 133 × 2)
  → Conv1D (dilated, 感受野覆盖多帧)
  → BN + ReLU + Residual × N blocks
  → 3D 姿态序列 (T 帧 × 133 × 3)
```

**关键参数：**
- 输入：2D 坐标序列，通常 243 帧（约 8 秒 @ 30fps）
- 感受野：通过空洞卷积指数扩大，覆盖整个序列
- 参数量：约 16M（RTMPose-m 的 1/3 ~ 1/2）
- 模型体积：< 10MB

### CoreML 转换友好性

全部算子都在 CoreML 成熟支持列表里：
- Conv1D → 直接映射
- BatchNorm → 融合到 Conv，零额外开销
- ReLU → 支持
- Residual Add → 支持

**与 RTMPose 转换难度同级——没有 Transformer、没有 Attention、没有动态 shape。**

## 两种部署路线

### 路线 A：全设备端

```
设备上 RTMPose CoreML (2D) → 设备上 VideoPose3D CoreML (2D→3D) → 133 点 3D
```

| 指标 | 估值 |
|---|---|
| RTMPose 推理 | 20-30ms |
| VideoPose3D 推理 | < 5ms（纯坐标运算） |
| 总延迟 | < 35ms，可实时 |
| 额外模型体积 | < 10MB |

**代价：** 需要 N 帧上下文窗口才有好的 3D 结果（通常 9-27 帧），冷启动时有短暂延迟。

### 路线 B：混合部署

```
设备端 RTMPose CoreML (2D)
  → 2D 坐标序列上传到服务器
  → 服务器 VideoPose3D (PyTorch 原生，GPU)
  → 返回 3D 坐标
  → 设备端消费 3D 数据进行评分/渲染
```

| 混合部署 | 说明 |
|---|---|
| 适合场景 | 训练后回放分析、标准模型生成、周报分析 |
| 不适合场景 | 实时教练指导（需要本地 3D） |
| 延迟 | 网络 + 推理，通常 100-500ms |
| 优势 | 零客户端模型体积增加，服务器随时升级模型 |

**推荐策略：** 实时教练用路线 A（本地 VideoPose3D），深度分析/标准模型生成用路线 B（服务器）。

## 2D → 3D Lifting 技术细节

### 输入预处理

RTMPose 输出的 2D 坐标需要先做归一化再送入 VideoPose3D：

```
1. 以髋部中心（root）为原点，所有坐标减去 root
2. 按躯干长度缩放（根颈距 → 1.0），消除摄像机距离/人体比例差异
3. 构造 T 帧窗口（滑动窗口，每次推理取最近 T 帧）
```

### 输出后处理

VideoPose3D 输出的是归一化 3D 坐标，需要反向映射：

```
1. 按躯干长度反归一化
2. 加回 root 偏移
3. 输出 BodyJoints（x, y, z 均以米为单位，root 为原点）
```

### 与 Apple Vision 3D 的关系

Apple Vision 的 `VNDetectHumanBodyPose3DRequest` 也出 3D，但只有 19 点。VideoPose3D 叠加 RTMPose 可出 133 点 3D。两者可以通过共享的 17 个身体点对齐：

```
Apple Vision 3D (19 点) ──对比──→ 用于验证 VideoPose3D 深度精度
RTMPose + VideoPose3D (133 点) ──→ 主 3D 管线
```

## 标准模型生成中的位置

```
标准动作视频
  → MediaPipe (2D, 33 点)
  → VideoPose3D (2D → 3D)
  → 3D 关键帧角度提取
  → 标准模型 JSON (含 x, y, z 三维角度)
```

有了 3D 后，标准模型的质量会明显提升：比如深蹲底部不再只是"膝角 < 90°"，而是能描述膝盖在三维空间中相对脚尖的位置关系。

## 实施步骤

1. Python 环境搭建 VideoPose3D（跑通预训练模型）
2. 用 RTMPose 2D 输出验证输入格式兼容性
3. `torch.onnx.export` → ONNX → `onnx2torch` → `coremltools` → `.mlpackage`
4. 设备端集成：`RTMPoseDetector` 输出 → 滑动窗口缓存 → `VideoPose3DDetector` → `BodyJoints3D`
5. 改造 `Skeleton3DRenderer` 利用真实的 `position3D.z`（目前是占位值 0）

## 相关

- [[fit-pose-detection-implementation|姿态检测实现]] — RTMPose 133 点完整链路
- [[fit-standard-exercise-model|标准动作模型库]]
- [[fit-rtmpose-plan|RTMPose 升级方案]] — 双模型方案
