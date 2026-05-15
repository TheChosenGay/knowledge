---
tags: [swift, 编程语言, actor, combine]
created: 2026-05-11
---

# Swift 核心概念

## 1. 自动生成 Memberwise Initializer

没有写 `init` 的 struct/class，Swift 自动生成。参数 = 所有没有默认值的存储属性。

```swift
struct PosePoint {
    let joint: String      // 没有 = xxx → init 参数
    let location: CGPoint
    let confidence: Float
}
// 自动生成: init(joint: String, location: CGPoint, confidence: Float)
```

有默认值的属性不参与 init：

```swift
class PoseAnalysisViewModel: ObservableObject {
    let image: UIImage              // 必传
    let poseDetector: PoseDetectService = VisionPoseDetector.detector  // 可省略
    @Published var state: AnalysisState = .loading  // 不参与 init
}
```

> 规则：有 `= 初始值` 的属性不出现在 init 参数里。`@State`、`@StateObject` 给了初始值的也不算。

---

## 2. Actor 并发模型

### Actor 模型的来源

Actor 模型是 1973 年 Carl Hewitt 提出的**通用并发计算理论**，不是 Swift 专属概念。  
Erlang、Akka（Scala/Java）、Swift、Kotlin 都实现了这个模型。

"Hollywood Principle"（好莱坞原则）是 IoC/依赖注入的说法（"Don't call us, we'll call you"），  
**与 Actor 模型是两个不同的概念**，不要混淆。

### Actor 的核心思想

并发世界里最根本的问题：**多个任务同时读写同一块数据 → 数据竞争**。

传统解法是加锁，Actor 模型换了一个角度：  
> **每个 Actor 独占自己的状态，外界只能通过消息通信，不能直接访问。**

```
传统并发：多线程 + 锁
  Thread1 ──┐
  Thread2 ──┼──► 共享内存  (需要手动加锁保护)
  Thread3 ──┘

Actor 模型：消息传递
  Task1 ──► [消息队列] ──► Actor (独占状态，串行处理)
  Task2 ──► [消息队列] ──┘
```

### Swift Actor 的内部实现

每个 actor 内置一个**串行执行器（Serial Executor）**：

```
actor Counter {
  ┌──────────────────────────┐
  │  var count = 0  ← 受保护  │
  │  Serial Executor (队列)   │
  │  [task1] [task2] [task3] │  ← 任务排队，一次只跑一个
  └──────────────────────────┘
}
```

从外部调用时：
1. 调用被包装成任务，**投递到 actor 的串行队列**
2. 调用方任务**挂起**（不阻塞线程，线程可去做别的事）
3. actor 处理完后，**恢复**调用方，返回结果

### 为什么必须 await

`await` = "把我的请求排进 actor 队列，我挂起等结果"。

```swift
// 挂起 ≠ 阻塞：线程被释放去执行其他任务
let c = Counter()
await c.increment()   // 必须 await，因为要排队等执行权
```

**actor 内部**调用自己的方法不需要 await，因为已经持有执行权：

```swift
actor Counter {
    var count = 0
    func incrementTwice() {
        count += 1   // ✅ 不需要 await，已在队列中执行
        count += 1
    }
}
```

### 挂起 vs 阻塞

```
❌ 传统锁（阻塞）：线程卡死等待，浪费 CPU
   Thread ████[等锁]████████████ 继续

✅ Actor await（挂起）：线程释放去干别的
   Task A  ████[挂起]────────────[恢复]
   Thread  ████[Task B 使用这个线程]████
```

### Actor 间共享数据

| 场景 | 方案 |
|---|---|
| 不可变数据 | 直接传 `Sendable` 值类型，零成本 |
| 读写对方数据 | `await` 调用对方方法，消息传递 |
| 多个 actor 共享可变状态 | 把共享状态单独包成一个 actor |
| 绝对不能做的 | 把同一个普通 `class` 传给多个 actor 直接修改 |

**核心原则：状态归属于唯一的 actor，其他人通过消息请求，不直接触碰。**

### Sendable：跨 actor 传递的通行证

`Sendable` 是编译期标记协议，无任何方法，只告诉编译器："这个类型跨并发边界传递是安全的"。

```swift
// ✅ 值类型（struct/enum）— 传递时复制，自动推断为 Sendable
struct Config { let host: String; let port: Int }

// ✅ actor 本身是 Sendable — 访问受串行执行器保护
actor MyActor { ... }

// ❌ 普通 class — 引用类型，多个 actor 持有同一对象可同时修改
class UserSession { var token: String = "" }  // 不是 Sendable
```

**class 需要跨 actor 传递时的选择：**

```swift
// 方案一：加锁后标记 @unchecked Sendable（手动保证，编译器不再检查）
class SafeCache: @unchecked Sendable {
    private let lock = NSLock()
    private var data: [String: Any] = [:]
}

// 方案二：改成 struct（值语义，复制传递）
struct UserSession: Sendable { let token: String }

// 方案三：改成 actor（访问通过 await 保护）
actor UserSession { var token: String = "" }
```

### Actor 隔离：MainActor 与 nonisolated

**MainActor** — 特殊的全局 Actor，代表主线程，所有 UI 操作必须在此执行：

```swift
@MainActor
class MyViewModel: ObservableObject { ... }
```

项目配置 `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`，默认所有类型隔离到主线程。

**错误场景**：在非 MainActor 上下文访问 MainActor 隔离的属性：

`Main actor-isolated static property can not be referenced from a nonisolated context`

**修复**：

```swift
nonisolated static let detector = VisionPoseDetector()
```

`nonisolated` = "不属于任何 Actor，可同步访问"。适用条件：`let` 不可变，或纯计算无副作用。

### Task 与 Actor 的关系

**Task 是执行载体，Actor 是隔离边界。** Task 在某个 actor 的执行器上运行，actor 决定"谁有权执行这段代码"。

| 创建方式 | 继承的 actor 上下文 | 实际执行位置 |
|---|---|---|
| `Task { }` | 继承当前 actor | 当前是 @MainActor → 主线程；当前是普通 actor → 该 actor |
| `Task.detached { }` | 不继承任何 actor | 协作线程池（非主线程） |
| `.task { }` (SwiftUI) | 继承 View 的 @MainActor | 主线程启动，await 后可切换 |

```swift
@MainActor
func setup() {
    Task {
        // 继承 @MainActor → 仍在主线程
        updateUI()  // ✅ 安全
    }

    Task.detached {
        // 不继承 → 协作线程池，不在主线程
        updateUI()  // ❌ 危险！需要 await MainActor.run { }
    }
}
```

**Task.detached 的正确使用姿势：**

```swift
Task.detached {
    // 在后台线程池执行耗时操作
    let result = await heavyComputation()

    // 需要更新 UI 时，显式跳回主线程
    await MainActor.run {
        self.result = result
    }
}
```

---

## 3. Task 并发任务

### Task — 手动创建异步任务

```swift
// 在同步上下文启动异步工作
Task {
    let result = await fetchData()
    print(result)
}

// 带优先级
Task(priority: .background) {
    await heavyComputation()
}
```

**持有 Task 以便取消：**

```swift
class MyViewModel: ObservableObject {
    private var loadTask: Task<Void, Never>?

    func load() {
        loadTask?.cancel()      // 取消上一次
        loadTask = Task {
            await fetchAndUpdate()
        }
    }

    deinit { loadTask?.cancel() }
}
```

### async let — 并行等待多个值

```swift
// ❌ 串行：总时间 = 2s + 3s = 5s
let user = await fetchUser()
let posts = await fetchPosts()

// ✅ 并行：总时间 = max(2s, 3s) = 3s
async let user = fetchUser()
async let posts = fetchPosts()
let (u, p) = await (user, posts)
```

### TaskGroup — 动态并行任务

```swift
let images = await withTaskGroup(of: UIImage?.self) { group in
    for url in urls {
        group.addTask { await downloadImage(url) }
    }
    var result: [UIImage] = []
    for await image in group {
        if let img = image { result.append(img) }
    }
    return result
}
```

### 取消机制（协作式）

取消不是强制中断，需要任务主动检查：

```swift
Task {
    for item in largeList {
        try Task.checkCancellation()  // 已取消则抛出 CancellationError
        await process(item)
    }
}
```

### .task — SwiftUI 生命周期绑定

```swift
struct ContentView: View {
    @State private var data = ""

    var body: some View {
        Text(data)
            .task {
                data = await fetchData()
                // 视图消失时自动取消 ← 与手动 Task{} 最大的区别
            }
    }
}
```

**带依赖值，值变化时自动重跑：**

```swift
.task(id: userId) {
    // userId 变化 → 旧任务自动取消，新任务自动启动
    await loadUser(id: userId)
}
```

### Task vs .task

| | `Task { }` | `.task { }` |
|---|---|---|
| 使用场景 | 任意地方（ViewModel、service）| SwiftUI View |
| 生命周期 | 手动管理，需手动取消 | 绑定视图，自动取消 |
| 响应值变化 | 手动处理 | `.task(id:)` 自动重跑 |

---

## 4. async/await 错误处理

### 基础：async throws + try await

```swift
func fetchUser(id: String) async throws -> User {
    let data = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data.0)
}

do {
    let user = try await fetchUser(id: "123")
} catch {
    print("失败：\(error)")
}
```

### 细化错误类型

```swift
enum APIError: Error {
    case notFound
    case unauthorized
    case networkFailure(underlying: Error)
}

do {
    let user = try await fetchUser(id: id)
} catch APIError.unauthorized {
    showLogin()
} catch APIError.notFound {
    showEmpty()
} catch {
    showGenericError(error)  // 兜底
}
```

### Task 内部错误不会自动传播，必须自己捕获

```swift
// ❌ 错误被静默吞掉
Task { let user = try await fetchUser(id: id) }

// ✅ 内部处理
Task {
    do {
        user = try await fetchUser(id: id)
    } catch is CancellationError {
        return  // 取消是正常流程，不显示错误
    } catch {
        errorMessage = error.localizedDescription
    }
}
```

### ViewModel 标准模式

```swift
@MainActor
class UserViewModel: ObservableObject {
    @Published var user: User?
    @Published var errorMessage: String?
    @Published var isLoading = false

    func load(id: String) async {
        isLoading = true
        errorMessage = nil
        defer { isLoading = false }

        do {
            user = try await fetchUser(id: id)
        } catch APIError.unauthorized {
            errorMessage = "请重新登录"
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

### Result 类型：把错误变成值

不想用 throws 时，用 `Result` 更灵活：

```swift
func fetchUser(id: String) async -> Result<User, APIError> { ... }

let result = await fetchUser(id: id)
switch result {
case .success(let user): self.user = user
case .failure(let error): self.errorMessage = error.localizedDescription
}
```

### 选择哪种模式

| 场景 | 推荐方式 |
|---|---|
| 普通异步函数 | `async throws` + `try await` |
| ViewModel 加载数据 | 内部 do-catch，暴露 `@Published var error` |
| 需要把错误当值传递 | `Result<T, E>` |
| Task 内部 | 必须自己捕获，不会自动传播 |
| 取消流程 | 单独 catch `CancellationError` |

---

## 5. let 常量与线程安全

不是语言要求，是语义自然安全：

- `let` 初始化后不可变 → 多线程读永远不冲突
- 线程安全问题只发生在 `var` 可变状态上
- 一个 class 线程安全的充要条件：**无共享可变状态**

---

## 6. Combine 与 ObservableObject

iOS 16 上 `ObservableObject` + `@Published` 确实在用 Combine：

- `objectWillChange` → `Combine.ObservableObjectPublisher`
- `@Published` → Combine 的 `PassthroughSubject`
- 需显式 `import Combine`

|     | iOS 16                            | iOS 17+         |
| --- | --------------------------------- | --------------- |
| 写法  | `ObservableObject` + `@Published` | `@Observable` 宏 |
| 底层  | **Combine**                       | Observation 框架  |
| 精度  | 对象级别                              | 属性级别            |

---

## 7. #Predicate — 类型安全的条件表达式

Swift 5.9 引入，替代 `NSPredicate` 字符串写法，编译期检查类型安全。

### 对比

```swift
// 老写法：字符串，运行时才知道错没错
NSPredicate(format: "age > %d AND name == %@", 18, "Alice")

// 新写法：编译期检查
#Predicate<Person> { $0.age > 18 && $0.name == "Alice" }
```

### 基本用法

```swift
// 定义
let isAdult = #Predicate<Person> { $0.age >= 18 }

// 手动过滤
let adults = try people.filter(isAdult)

// SwiftData @Query
@Query(filter: #Predicate<Todo> { $0.isCompleted == false })
var pendingTodos: [Todo]
```

### 支持的操作

```swift
#Predicate<Item> { $0.price > 100 }                    // 比较
#Predicate<Item> { $0.price > 100 && $0.inStock }      // 逻辑组合
#Predicate<Item> { $0.name.contains("Pro") }            // 字符串
#Predicate<Todo> { $0.tag?.name == "work" }            // 可选值
#Predicate<Order> { $0.items.isEmpty }                  // 集合
```

### 动态构建（运行时捕获外部变量）

```swift
func makePredicate(keyword: String) -> Predicate<Todo> {
    #Predicate<Todo> { $0.title.contains(keyword) }  // 捕获 keyword
}

// SwiftData @Query 动态 filter
struct FilteredList: View {
    @Query var todos: [Todo]
    init(keyword: String) {
        _todos = Query(filter: #Predicate<Todo> { $0.title.contains(keyword) })
    }
}
```

### 与 SwiftData 的关系

`#Predicate` 被转换成 SQL WHERE 子句在数据库层过滤，效率高于内存过滤。

> **注意**：predicate 内不能用自定义函数或复杂计算，否则运行时报错。复杂逻辑在内存中二次 `.filter` 处理。

---

## 相关

- [[../框架/swiftui-state|SwiftUI 状态观察]]
- [[../框架/swiftdata|SwiftData]] — #Predicate 在 @Query 中的应用
- [[../项目/fit-overview|PostureAI 项目]]
