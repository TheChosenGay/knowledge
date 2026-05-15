---
tags: [project, fit, rtmpose, coreml, 姿态检测, implementation]
created: 2026-05-14
---

# PostureAI — 姿态检测完整实现方案

## 1. RTMPose 133 点整体架构与数据流

### 模型概览

RTMPose-WholeBody (RTMW-DW-L-M) 是 MMPose 生态的实时全身姿态估计模型，输出 COCO-WholeBody 格式的 133 个 2D 关键点：

| 区域 | 索引 | 点数 | 内容 |
|------|------|------|------|
| Body | 0-16 | 17 | nose, eyes, ears, shoulders, elbows, wrists, hips, knees, ankles |
| Feet | 17-22 | 6 | big_toe, small_toe, heel (左右各3) |
| Face | 23-90 | 68 | 面部轮廓 + 五官特征点 |
| Left Hand | 91-111 | 21 | wrist + 5指×4关节 |
| Right Hand | 112-132 | 21 | 同上 |

### 推理流水线

```
摄像头帧 (CMSampleBuffer)
  │
  ├─ VNDetectHumanRectanglesRequest（Vision 人体框检测）
  │    └─ 取 confidence 最高的人体 boundingBox (归一化 0-1, 左下角原点)
  │
  ├─ CIContext 裁剪 + 缩放到 256×192 (模型输入尺寸)
  │    └─ cropRect → CIImage.cropped → transformed(scaleX, scaleY)
  │
  ├─ CoreML 推理 (MLModel.prediction)
  │    └─ 输入: CVPixelBuffer (192×256, 32BGRA)
  │    └─ 输出: simcc_x [1, 133, 384], simcc_y [1, 133, 512]
  │
  ├─ SimCC 解码 → 133 个 (x, y, confidence)
  │    └─ argmax ×2 轴上做，再除以 simccScale(2.0) 得到模型输入空间坐标
  │
  ├─ 坐标映射回全帧
  │    └─ xFull = bbox.origin.x + xNormCrop × bbox.width
  │    └─ yFull = bbox.origin.y + (1 - yNormCrop) × bbox.height
  │
  └─ WholeBodyJointMap 过滤/重命名
       ├─ Body 17 点 → legacy 19 点命名 (含合成的 neck_1_joint, root)
       ├─ Feet 6 点 + Hands 42 点 → 保持 canonical 命名
       └─ Face 68 点 → 丢弃 (renderable 阶段不需要面部)
```

### 为什么用 VNDetectHumanRectanglesRequest 而不是直接全图推理

RTMPose 是 **Top-down** 模型：它假设输入图像已经裁剪到单个人体区域。全图直接推理会导致精度严重下降。所以必须先用 Vision 检测人体框，再 crop+resize 送入模型。

### 关键设计决策

- **使用 `MLModel.prediction(from:)` 直接推理**，而非 `VNCoreMLRequest`。后者有额外的 Vision 协调开销，在实时视频场景下不必要。
- **`computeUnits = .all`**：让系统在 CPU/GPU/Neural Engine 间自动选择，A14+ 芯片上 Neural Engine 对 CNN 有显著加速。
- **`SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`** 是项目级设置，意味着所有代码默认 MainActor。`RTMPoseDetector` 用 `nonisolated` 修饰以脱离 MainActor。

---

## 2. PyTorch → ONNX → CoreML 完整转换链路

这是本方案最核心的基础设施。转换脚本位于 `ModelConvert/ConvertRTMPose.py`，Python 3.12 环境。

### 2.1 整体流程

```
PyTorch (.pth)  ──(官方导出)──→  ONNX (.onnx)  ──(onnx2torch)──→  PyTorch (中间)  ──(coremltools)──→  CoreML (.mlpackage)
```

注意：这里绕了一圈 ONNX → back to PyTorch → CoreML。原因是 **coremltools 直接吃 ONNX 的兼容性很差**（尤其 SimCC 这类复杂输出头），而 `torch.jit.trace` + `coremltools.convert` 的路径成熟稳定。

### 2.2 步骤详解

#### Step 1: 获取官方 ONNX

MMPose 官方提供了预导出的 ONNX 文件，打包在 zip 中：

```python
POSE_ONNX_URL = "https://download.openmmlab.com/mmpose/v1/projects/rtmw/onnx_sdk/rtmw-dw-l-m_simcc-cocktail14_270e-256x192_20231122.zip"
```

模型规格：
- **RTMW-DW-L-M**：Depth-Wise Large-M 变体，在速度和精度间取平衡
- **输入**：256×192 RGB
- **输出**：SimCC 格式 — `simcc_x [1, 133, 384]`, `simcc_y [1, 133, 512]`
- **关键点数**：133 (COCO-WholeBody)

#### Step 2: 修复 ONNX Clip 算子

这是最常见的一个坑。官方 ONNX 中 Clip 节点的 min/max 输入可能是空字符串，`onnx2torch` 转换时会崩溃。

```python
import onnx
from onnxsim import simplify

model = onnx.load(str(onnx_path))
model, _ = simplify(model)  # 先简化图结构

for node in model.graph.node:
    if node.op_type == "Clip":
        # 找到空的 min/max 输入，填充默认值
        for i, inp in enumerate(node.input):
            if inp == "":
                val = 0.0 if i == 1 else float("inf")
                tensor = helper.make_tensor(f"clip_const_{i}", TensorProto.FLOAT, [], [val])
                model.graph.initializer.append(tensor)
                node.input[i] = tensor.name
```

**为什么会出现空输入？** ONNX 规范允许 Clip 算子的 min/max 为可选输入（未提供时表示无限制），但 `onnx2torch` 和 `coremltools` 都要求它们是具体值。

#### Step 3: ONNX → PyTorch (中间表示)

```python
from onnx2torch import convert as onnx2torch_convert

model = onnx2torch_convert(str(fixed_onnx_path))
model.eval()

# 验证：跑一次 dummy forward，确认输出 shape
dummy = torch.randn(1, 3, 256, 192)
with torch.no_grad():
    outputs = model(dummy)
# 预期: simcc_x [1, 133, 384], simcc_y [1, 133, 512]
```

**`onnx2torch` 库**：将 ONNX 图的每个算子 1:1 映射为 PyTorch 的 `nn.Module`。转换后的模型可以直接 `torch.jit.trace`，这是后续 CoreML 转换的前置条件。

#### Step 4: PyTorch → CoreML (关键步骤)

```python
import torch
import coremltools as ct

dummy = torch.randn(1, 3, 256, 192)
traced = torch.jit.trace(torch_model, dummy)  # TorchScript 化

mlmodel = ct.convert(
    traced,
    inputs=[
        ct.ImageType(
            name="input",
            shape=(1, 3, 256, 192),      # BCHW
            scale=1.0 / 255.0,            # 归一化：像素 0-255 → 0-1
            color_layout=ct.colorlayout.RGB,
        )
    ],
    outputs=[
        ct.TensorType(name="simcc_x"),
        ct.TensorType(name="simcc_y"),
    ],
    minimum_deployment_target=ct.target.iOS17,
    compute_precision=ct.precision.FLOAT16,
)
```

**各个参数的意义和注意事项：**

| 参数 | 说明 | 注意事项 |
|------|------|---------|
| `ct.ImageType` | 声明输入为图像，CoreML 会自动处理 `CVPixelBuffer` → tensor 的转换 | `scale=1.0/255.0` 等价于 PyTorch 的 `/255` 预处理。**color_layout 必须和训练时一致**（RTMPose 训练用 RGB） |
| `minimum_deployment_target=iOS17` | 最低 iOS 版本 | iOS17 支持更多 CoreML 算子。设太低会导致某些 op 不被支持而回退到 CPU |
| `compute_precision=FLOAT16` | 权重精度减半 | **float32 → float16 几乎不掉精度但模型体积减半**。如果检测结果异常，优先怀疑这里，改成 FLOAT32 对比 |
| `ct.TensorType` (输出) | 多输出模型必须显式命名每个输出，否则只取第一个 | **名称必须和 ONNX 输出名一致**（`simcc_x`, `simcc_y`），后续 iOS 端通过名称取 MLMultiArray |

#### Step 5: 验证

```python
loaded = ct.models.MLModel(str(coreml_path))
spec = loaded.get_spec()

# 检查输入输出描述
for inp in spec.description.input:
    print(inp.name)
for out in spec.description.output:
    print(out.name)

# 跑一次预测验证 shape
img = Image.new("RGB", (192, 256), (128, 128, 128))
result = loaded.predict({"input": img})
# 预期: simcc_x shape=(1, 133, 384), simcc_y shape=(1, 133, 512)
```

### 2.3 常见坑

1. **CoreML 输出 shape 多了 batch 维**：`simcc_x` 在 CoreML 中是 `[1, 133, 384]`，比 PyTorch 的 `[133, 384]` 多一个 batch=1 的维度。iOS 端取 `shape[1]` 是 keypoint 数，`shape[2]` 是 SimCC 分辨率。

2. **Float16 精度损失**：大部分情况无事。但如果某个关节点的置信度普遍偏低，可以怀疑 FLOAT16 量化导致的精度损失。改成 `ct.precision.FLOAT32` 对比验证。

3. **输入颜色空间**：RTMPose 训练用的是 RGB，但 `CVPixelBuffer` 从摄像头来通常是 BGRA。CoreML 的 `ImageType` 声明了 RGB 会自动转换，但如果是手动构造 `MLFeatureValue(pixelBuffer:)` 而不经过 ImageType 预处理，就需要自己保证颜色空间正确。当前代码用的是 `MLDictionaryFeatureProvider` 直接传 `CVPixelBuffer`，CoreML 会自动根据 `ImageType` 声明做转换。

4. **iOS 版本兼容性**：设为 `iOS17` 是因为 SimCC 解码后的某些算子需要 iOS17 的 CoreML 运行时。如果降级到 iOS16，部分算子会 fallback 到 CPU，推理变慢。

5. **`.mlpackage` vs `.mlmodelc`**：`.mlpackage` 是源码格式（目录），Xcode 编译时自动转为 `.mlmodelc`（编译后格式）。运行时优先加载 `.mlmodelc`（更快），fallback 到 `.mlpackage`。

### 2.4 依赖环境

```
Python 3.12
coremltools==8.2
onnx==1.17.0
onnxsim==0.4.36
onnx2torch==1.5.14
torch==2.5.1
```

---

## 3. SimCC 解码原理与坐标映射

### 3.1 SimCC 是什么

SimCC (Simple Coordinate Classification) 是 MMPose 提出的一种轻量解码头。不同于传统的直接回归 (x, y)，SimCC 将 x 和 y 坐标分别视为**分类问题**：

- 水平方向分成 `simccW` 个 bin（如 384），对每个关键点输出一个 384 维的 logits 向量，argmax 得到水平位置
- 垂直方向分成 `simccH` 个 bin（如 512），对每个关键点输出一个 512 维的 logits 向量，argmax 得到垂直位置

### 3.2 为什么分辨率是 2×

`simccScaleX = simccScaleY = 2.0`。SimCC 的 bin 数是模型输入尺寸的 2 倍（384 = 192×2, 512 = 256×2），这提供了**亚像素精度**：argmax 的结果除以 2 后可以得到 0.5 像素级别的精度。

### 3.3 解码步骤

```swift
// 1. argmax 在 x 轴上 (每个关键点的 384 个 bin 中找最大)
let xInCrop = Float(bestXIdx) / simccScaleX  // 除以 2 → 模型空间坐标

// 2. argmax 在 y 轴上 (每个关键点的 512 个 bin 中找最大)
let yInCrop = Float(bestYIdx) / simccScaleY  // 除以 2 → 模型空间坐标

// 3. 归一化到裁剪区域 (除以模型输入尺寸)
let xNormCrop = xInCrop / Float(modelInputWidth)   // 0~1
let yNormCrop = yInCrop / Float(modelInputHeight)  // 0~1

// 4. 映射到全帧 (Vision bounding box 坐标系)
let xFull = bbox.origin.x + xNormCrop * bbox.width
let yFull = bbox.origin.y + (1 - yNormCrop) * bbox.height
```

### 3.4 坐标系注意事项

- **Vision boundingBox**：原点在**左下角**，归一化 0-1
- **Core Image**：原点在**左上角**（CGRect 标准），所以 cropRect 直接使用 `bbox.origin.x * w, bbox.origin.y * h`
- **映射回全帧时**：x 直接映射，y 需要 `1 - yNormCrop` 翻转（从模型坐标系的左上角原点转回 Vision 的左下角原点）

### 3.5 Float16 数据读取

CoreML 输出可能是 `Float16` 类型。读取时需要判断 `dataType`：

```swift
if mlArray.dataType == .float16 {
    let src = mlArray.dataPointer.assumingMemoryBound(to: Float16.self)
    for i in 0..<count { result[i] = Float(src[i]) }
} else {
    let src = mlArray.dataPointer.assumingMemoryBound(to: Float.self)
    for i in 0..<count { result[i] = src[i] }
}
```

**不可用 `Float16` 直接做运算**（Swift 的 `Float16` 不支持比较运算符），必须转为 `Float`。

---

## 4. WholeBodyJointMap 133 点命名映射体系

### 4.1 设计思路

采用**三层映射**，互不干扰：

```
Canonical 名称 (133 个, 模型原生输出)
  │
  ├─→ legacyMapping (17 个 body 点 → 19 个 legacy 名称)
  │     └─ 用于兼容现有的 AngleCalculator / EdgeDetector / Skeleton3DRenderer
  │
  ├─→ renderable (body legacy + feet + hands, 跳过 face 68 点)
  │     └─ 用于骨架渲染 (Skeleton3DRenderer / SkeletonRenderer)
  │
  └─→ 原始输出 (全部 133 点, 不做映射)
        └─ 用于未来扩展 (如面部表情分析)
```

### 4.2 Canonical 命名规则

- Body 17 点：COCO 标准名称 (`nose`, `left_shoulder`, `right_elbow`, ...)
- Feet 6 点：`left_big_toe`, `left_small_toe`, `left_heel`, 右侧同理
- Face 68 点：`face_0` ~ `face_67` (不渲染，仅做映射)
- Hands 42 点：`left_hand_wrist`, `left_thumb_1`~`4`, `left_index_1`~`4`, ..., 右侧同理

### 4.3 Legacy 19 点映射

```swift
static let legacyMapping: [String: String] = [
    "left_shoulder": "left_shoulder_1_joint",
    "left_elbow":    "left_forearm_joint",
    "left_wrist":    "left_hand_joint",
    "left_hip":      "left_upLeg_joint",
    "left_knee":     "left_leg_joint",
    "left_ankle":    "left_foot_joint",
    // ... 右侧同理
]

// 合成点
neck_1_joint = midpoint(left_shoulder, right_shoulder)
root         = midpoint(left_hip, right_hip)
```

**为什么需要合成 `neck_1_joint` 和 `root`？** 因为 Apple Vision 的 19 点模型包含后颈和髋部中心点，而 COCO 格式只有左右肩/髋的端点。为了兼容下游代码（它们依赖这两个点做脊柱角度计算），需要从左右端点合成。

### 4.4 面部点的处理

面部 68 点（索引 23-90）在 `mapToRenderable` 中被跳过。原因是：
- 健身场景不需要面部信息
- 68 个面部点渲染出来会严重遮挡身体骨骼
- 减少骨架渲染开销

---

## 5. 双 Service 协议设计与单例模式

### 5.1 两个协议

```swift
// 实时视频流 (CMSampleBuffer 输入)
protocol BodyPoseDetectService {
    func detectBodyPose(from sampleBuffer: CMSampleBuffer) async throws -> BodyJoints?
}

// 静态图片 (UIImage 输入)
protocol PoseDetectService {
    func detectPose(from image: UIImage) async throws -> PosePoints?
}
```

**为什么分两个协议？**
- 实时视频用 `CMSampleBuffer` 可以直接拿 `CVPixelBuffer`，避免 buffer→UIImage 的转换开销
- 静态图片用 `UIImage`，适配相册照片、视频抽帧等场景
- 返回类型不同：`BodyJoints` 包含 `position3D`，为 Phase 2 苹果 3D 做准备；`PosePoints` 仅 2D

### 5.2 单例模式

```swift
nonisolated static let detector = RTMPoseDetector()

private init() {}  // 外部无法创建新实例
```

**`nonisolated` 关键**：项目级 `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`，但 `RTMPoseDetector` 的推理方法需要在后台线程执行。`nonisolated` 让静态属性和方法脱离 MainActor 隔离。

**为什么是单例？** CoreML 模型加载有显著开销（IO + 编译），复用同一个 `MLModel` 实例可以避免重复加载。`MLModel` 的 `.prediction` 方法是线程安全的。

### 5.3 后端切换

```swift
enum PoseDetectorBackend: String, CaseIterable {
    case appleVision  // 系统内置 19 点 3D
    case rtmPose      // RTMPose 133 点 2D
}
```

`PoseFrameProcessor` 构造时选择后端，内部持有 `BodyPoseDetectService` 协议引用，对上游透明。

---

## 6. 骨架渲染层

### 6.1 Skeleton3DRenderer — 3D 实时渲染 (Canvas)

文件：`Features/ExerciseForm/Feedback/Skeleton3DRenderer.swift`

**职责**：在 `RealTimeCameraView` 的 SwiftUI `Canvas` 上叠加骨架。

**连接数**：59 根（body 13 + feet 6 + left hand 20 + right hand 20）

**特性**：
- **时序平滑** (`smoothingFactor = 0.35`)：使用 EMA 平滑关节位置，消除帧间抖动
- **深度衰减** (`depthOpacity`)：根据 `position3D.z` 调整骨骼不透明度（近浓远淡），为 Phase 2 做准备
- **前置摄像头镜像**：`isFrontCamera` 时水平翻转坐标
- **中文关节标签**：预留 `jointLabels` 字典，Canvas 直接渲染 `Text`
- **CGContext 渲染** (`renderOntoContext`)：为录制提供非 Canvas 的渲染路径

**坐标转换**：归一化 (0-1, 左下角原点) → 画布像素坐标
```swift
let rawPoint = CGPoint(
    x: vx * canvasSize.width,
    y: (1.0 - j.location2D.y) * canvasSize.height
)
```

### 6.2 SkeletonRenderer — 2D 静态渲染 (UIImage)

文件：`Features/PoseAnalysis/Models/SkeletonRenderer.swift`

**职责**：在静态图片上绘制骨架，用于照片分析和视频离线处理。

**连接数**：61 根（body 15 + body 横向 2 + feet 6 + hands 40）

**与 3D 版的区别**：
- `nonisolated` — 在 `Task.detached` 后台线程上运行（iOS 现代 UIGraphicsImageRenderer 线程安全）
- 使用 `UIGraphicsImageRenderer` 直接生成标注后的 `UIImage`
- 无时序平滑（每次独立渲染）
- 关节根据置信度着色：`>0.6` 绿色，否则黄色

---

## 7. PoseFrameProcessor 帧处理管线

文件：`Core/Vision/PoseFrameProcessor.swift`

### 7.1 管线设计

```
CMSampleBuffer
  │
  ├─ 跳帧控制: frameCount % frameSkipInterval == 0 (默认每4帧处理一次)
  │
  ├─ processQueue (serial, userInitiated)
  │    ├─ isProcessing gate: 确保同一时间只有一帧在处理 (丢帧策略)
  │    └─ Task { detectBodyPose → onPoseDetected/onPoseLost callback }
  │
  └─ PoseVideoRecorder (可选)
       └─ appendFrame(pixelBuffer, joints): 合成骨架到帧 → AVAssetWriter 写入
```

### 7.2 关键设计

- **`@unchecked Sendable`**：类的 `processFrame` 从 AVCaptureSession 的回调队列调用，需要 Sendable 才能跨 actor 传递。但内部可变状态 (`frameCount`, `isProcessing`) 有串行队列保护，实际线程安全
- **跳帧**：`frameSkipInterval = 4`，和模型推理速度匹配。CoreML 推理 ~20-30ms/帧，不跳帧会导致处理队列堆积
- **isProcessing gate**：如果上一帧还在推理中，直接丢弃当前帧。避免队列堆积导致的延迟
- **延迟启动录制**：录制器在第一帧到达时才根据实际 `CVPixelBuffer` 尺寸初始化，避免硬编码分辨率

---

## 8. 视频姿态检测 — AVAssetReader/Writer 离线处理

文件：`Features/ExerciseForm/ViewModels/VideoAnalysisViewModel.swift`

### 8.1 处理流程

```
选中视频 (AVAsset)
  │
  ├─ AVAssetReader + AVAssetReaderTrackOutput (32BGRA)
  │    └─ 逐帧读取 CVPixelBuffer
  │
  ├─ 帧采样: frameIndex % frameInterval == 0
  │    └─ 用户可选 1/2/4/8 帧间隔
  │
  ├─ CIImage(cvPixelBuffer) → CIContext.createCGImage → UIImage
  │    └─ (CoreML 推理需要 UIImage 输入)
  │
  ├─ RTMPoseDetector.detectPose(from: uiImage)
  │    └─ 检测成功 → SkeletonRenderer.render(image:points:)
  │    └─ 检测失败 → 写入原始帧
  │
  ├─ CGContext 绘制标注到 CVPixelBuffer
  │
  └─ AVAssetWriter + AVAssetWriterInputPixelBufferAdaptor
       └─ 写入 .mov 文件 (h264 编码)
```

### 8.2 为什么离线处理不走 `BodyPoseDetectService`？

离线有 `UIImage`（从视频帧解码），用 `PoseDetectService.detectPose(from:)` 更直接。避免 `UIImage → CVPixelBuffer → CMSampleBuffer` 的无意义转换。

### 8.3 线程模型

```swift
processingTask = Task.detached { [weak self] in
    // 这里脱离 MainActor，可以同步阻塞等待 AVAssetReader
    while reader.status == .reading {
        // 逐帧处理...
        await MainActor.run { self?.progress = p }
    }
}
```

**`Task.detached`** 脱离当前 `@MainActor` 上下文，在后台协程上运行。这是必须的，因为 `AVAssetReader.copyNextSampleBuffer()` 是同步阻塞调用，如果在 MainActor 上会卡 UI。

### 8.4 帧采样间隔

提供 4 档可选的采样间隔：

| 选项 | 间隔 | 适用场景 |
|------|------|---------|
| 每帧 | 1 | 短片段、关键动作分析 |
| 每2帧 | 2 | 正常速度动作 |
| 每4帧 | 4 | 长视频、慢动作分析 (默认) |
| 每8帧 | 8 | 超长视频、快速浏览 |

30fps 视频 × 4 帧间隔 = 实际处理 7.5fps，大部分人体动作在这个帧率下足够。

### 8.5 像素缓冲池

使用 `CVPixelBufferPool` 管理渲染缓冲区的内存分配和复用：

```swift
let poolAttrs: [String: Any] = [
    kCVPixelBufferPixelFormatTypeKey: kCVPixelFormatType_32BGRA,
    kCVPixelBufferWidthKey: Int(outputSize.width),
    kCVPixelBufferHeightKey: Int(outputSize.height),
    kCVPixelBufferIOSurfacePropertiesKey: [:],
]
CVPixelBufferPoolCreate(nil, nil, poolAttrs as CFDictionary, &pool)
```

**`IOSurfaceProperties`** 是关键：让 PixelBuffer 在 CPU 和 GPU 间共享内存，CGContext 绘制 (CPU) 和 AVAssetWriter 编码 (GPU 加速) 都不需要拷贝。

---

## 9. 文件索引

| 文件 | 职责 |
|------|------|
| `ModelConvert/ConvertRTMPose.py` | PyTorch→ONNX→CoreML 转换脚本 |
| `Core/Vision/RTMPoseDetector.swift` | CoreML 推理 + SimCC 解码 + 双协议实现 |
| `Core/Vision/WholeBodyJointMap.swift` | 133 点名映射 (canonical/legacy/renderable) |
| `Core/Vision/PoseDetectorConfiguration.swift` | 后端枚举 (appleVision / rtmPose) |
| `Core/Vision/PoseFrameProcessor.swift` | 实时帧处理管线 (跳帧/录制/isProcessing gate) |
| `Features/ExerciseForm/Feedback/Skeleton3DRenderer.swift` | 3D 实时骨架渲染 (Canvas + CGContext) |
| `Features/PoseAnalysis/Models/SkeletonRenderer.swift` | 2D 静态图像骨架渲染 |
| `Features/ExerciseForm/Views/RealTimeCameraView.swift` | 实时摄像头 UI + 入口 (RealTimeCameraEntryView) |
| `Features/ExerciseForm/ViewModels/VideoAnalysisViewModel.swift` | 视频离线处理 (AVAssetReader/Writer) |
| `Features/ExerciseForm/Views/VideoAnalysisView.swift` | 视频分析 UI (选择/处理/预览) |
| `Features/Camera/Views/VideoPickerView.swift` | PHPicker 视频选择器封装 |

## 相关

- [[fit-overview|PostureAI 项目总览]]
- [[fit-rtmpose-plan|RTMPose 集成方案]]
- [[../框架/rtmpose-wholebody|RTMPose WholeBody 模型]]
- [[../框架/coreml-integration|iOS CoreML 模型集成]]
- [[../编程语言/swift-core|Swift 核心概念]]
