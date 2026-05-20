---
tags: [swiftui, uikit, interop, representable]
created: 2026-05-20
---

# SwiftUI 中包装 UIKit

## 核心协议

```text
UIView                  -> UIViewRepresentable
UIViewController        -> UIViewControllerRepresentable
UIKit delegate/callback -> Coordinator
```

`SomeView` / `SomeViewController` 只是示例占位符，实际要换成 `UILabel`、`UIScrollView`、`UIImagePickerController`、`AVPlayerViewController` 等具体 UIKit 类型。

## 包装 UIView

```swift
struct UIKitViewWrapper: UIViewRepresentable {
    func makeUIView(context: Context) -> UILabel {
        UILabel()
    }

    func updateUIView(_ uiView: UILabel, context: Context) {
        uiView.text = "Hello"
    }
}
```

- `makeUIView`：创建 UIKit view，通常只做初始化。
- `updateUIView`：SwiftUI 状态变化时调用，用来同步状态到 UIKit。
- 不要在 `updateUIView` 中重复创建重对象，应做差异更新。

## 包装 UIViewController

```swift
struct UIKitControllerWrapper: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIImagePickerController {
        UIImagePickerController()
    }

    func updateUIViewController(_ controller: UIImagePickerController, context: Context) {}
}
```

适合包装 `UIImagePickerController`、`PHPickerViewController`、`AVPlayerViewController`、第三方 SDK controller 等。

## Context

`context` 是 SwiftUI 传入的运行时上下文，不是自己创建的。

常用内容：

```swift
context.coordinator   // SwiftUI 创建的 coordinator
context.environment   // SwiftUI 环境值，如 colorScheme
context.transaction   // 本次更新事务，如动画信息
```

## Coordinator

当 UIKit 组件需要 delegate、target-action、callback 时，用 `Coordinator` 桥接 UIKit -> SwiftUI。

创建顺序可理解为：

```text
makeCoordinator()
    -> SwiftUI 放入 context.coordinator
    -> makeUIView(context:) / makeUIViewController(context:)
    -> updateUIView(...) / updateUIViewController(...)
    -> dismantleUIView(...) / dismantleUIViewController(...)
```

示例：

```swift
struct TextFieldWrapper: UIViewRepresentable {
    @Binding var text: String

    func makeCoordinator() -> Coordinator {
        Coordinator(text: $text)
    }

    func makeUIView(context: Context) -> UITextField {
        let textField = UITextField()
        textField.delegate = context.coordinator
        return textField
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        if uiView.text != text {
            uiView.text = text
        }
    }

    final class Coordinator: NSObject, UITextFieldDelegate {
        @Binding var text: String

        init(text: Binding<String>) {
            self._text = text
        }

        func textFieldDidChangeSelection(_ textField: UITextField) {
            text = textField.text ?? ""
        }
    }
}
```

数据流：

```text
SwiftUI @State
    -> $binding
    -> updateUIView 同步到 UIKit

UIKit delegate/callback
    -> Coordinator
    -> @Binding / closure / ViewModel
    -> SwiftUI 刷新
```

## 数据传递选择

| 场景 | 推荐 |
|---|---|
| SwiftUI 持有状态，UIKit 可修改 | `@Binding` |
| UIKit 只通知单个事件 | closure |
| 状态复杂、多处共享 | `ObservableObject` / `@Observable` ViewModel |
| UIKit 只展示，不回传 | 普通属性 + `updateUIView` |
| delegate 回调多 | `Coordinator` |

## 资源清理

用 dismantle 方法清理资源：

```swift
static func dismantleUIView(_ uiView: SomeUIKitView, coordinator: Coordinator) {
    uiView.delegate = nil
}

static func dismantleUIViewController(_ controller: SomeController, coordinator: Coordinator) {
    controller.delegate = nil
}
```

适合清理：delegate、observer、timer、camera session、player、task、网络请求等。

## 注意点

- SwiftUI View 是 struct，会频繁重建；长期状态应放在 `Coordinator`、UIKit view/controller 或 ViewModel 中。
- UIKit 回调更新 SwiftUI 状态时，确保在主线程 / `MainActor`。
- `context.coordinator` 由 SwiftUI 自动创建并注入，不需要手动赋值。
- 没有 delegate/callback 时，可以不实现 `makeCoordinator()`。
