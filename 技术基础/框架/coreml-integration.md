---
tags: [coreml, ios, ml, 模型部署]
created: 2026-05-14
---

# iOS CoreML 模型集成

## 转换链路

```
PyTorch (.pth) → ONNX (.onnx) → CoreML (.mlpackage)
```

- **PyTorch → ONNX**：`torch.onnx.export(model, dummy_input, "model.onnx")`
- **ONNX → CoreML**：`coremltools.convert(onnx_model)` → `.mlpackage`

## 模型加载

两种方式：

1. **Xcode 自动生成 Swift 类**：将 `.mlpackage` 拖入 Xcode，自动生成同名 Swift 类
   ```swift
   let model = try MyModel(configuration: MLModelConfiguration())
   ```
2. **手动加载**：
   ```swift
   let model = try MLModel(contentsOf: modelURL)
   ```

## 推理方式对比

| 方式 | 适用场景 | 开销 |
|------|---------|------|
| `MLModel.prediction(from:)` | 直接推理，需手动预处理 | 低 |
| `VNCoreMLRequest` | Vision 框架封装，自动处理 resize | 有额外 Vision 协调开销 |

实时视频场景推荐 `MLModel.prediction(from:)` 直接推理，避免 `VNCoreMLRequest` 的额外开销。

## 预处理

输入图像需要裁剪和缩放到模型输入尺寸：
- **vImage**：Apple 的高性能图像处理框架，适合 `CVPixelBuffer` 操作
- **CGContext**：通用方案，灵活但稍慢

## 硬件加速

```swift
let config = MLModelConfiguration()
config.computeUnits = .all  // 让系统自动选择：CPU / GPU / Neural Engine
```

- A14+ 芯片的 Neural Engine 对 CNN 类模型有显著加速
- `.all` 是推荐默认值，系统会根据模型结构和当前负载动态选择

## 相关

- [[rtmpose-wholebody|RTMPose WholeBody]]
- [[../项目/fit-rtmpose-plan|PostureAI RTMPose 集成方案]]
