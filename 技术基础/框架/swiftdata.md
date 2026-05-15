---
tags: [swiftdata, swift, ios17, 持久化, coredata]
created: 2026-05-14
---

# SwiftData

Apple 2023 年推出（iOS 17+），Core Data 的 Swift 原生替代。底层仍是 Core Data + SQLite。

## 核心组件关系

```
ModelContainer（数据库文件，全局唯一）
    └─► ModelContext（操作会话，类似数据库连接）
            └─► @Query（响应式查询，自动刷新 UI）
```

- `ModelContainer` 存在 SwiftUI 环境变量 `\.modelContainer` 里
- 主线程的 `ModelContext` 存在 `\.modelContext` 里
- 数据本身存在 SQLite 文件里，不是"存在环境变量里"

---

## 完整示例：Todo App

### 数据模型

```swift
import SwiftData
import Foundation

// @Model 宏：自动生成 PersistentModel 协议实现
@Model
class Todo {
    var title: String
    var isCompleted: Bool
    var createdAt: Date
    // 关系：一个 Todo 属于一个 Tag（多对一）
    var tag: Tag?

    init(title: String) {
        self.title = title
        self.isCompleted = false
        self.createdAt = .now
    }
}

@Model
class Tag {
    var name: String
    var color: String
    // 关系：一个 Tag 包含多个 Todo（一对多）
    // deleteRule: .cascade → 删 Tag 时同时删所有关联 Todo
    @Relationship(deleteRule: .cascade, inverse: \Todo.tag)
    var todos: [Todo] = []

    init(name: String, color: String) {
        self.name = name
        self.color = color
    }
}
```

### App 入口 — 配置 ModelContainer

```swift
import SwiftUI
import SwiftData

@main
struct TodoApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        // 多个 Model 类型用数组传入
        .modelContainer(for: [Todo.self, Tag.self])
    }
}
```

### 主列表视图

```swift
import SwiftUI
import SwiftData

struct ContentView: View {
    // 从环境变量取主线程 ModelContext
    @Environment(\.modelContext) private var context

    // @Query：声明式查询，数据变化自动刷新 UI
    // filter: 只取未完成的
    // sort: 按创建时间降序
    @Query(
        filter: #Predicate<Todo> { !$0.isCompleted },
        sort: \Todo.createdAt,
        order: .reverse
    )
    var pendingTodos: [Todo]

    // 取全部 todo（不加 filter）
    @Query var allTodos: [Todo]

    @State private var showAdd = false

    var body: some View {
        NavigationStack {
            List {
                Section("待完成 (\(pendingTodos.count))") {
                    ForEach(pendingTodos) { todo in
                        TodoRow(todo: todo)
                    }
                    .onDelete(perform: deleteTodos)
                }
            }
            .navigationTitle("Todo")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button("添加", systemImage: "plus") {
                        showAdd = true
                    }
                }
                ToolbarItem(placement: .navigationBarLeading) {
                    EditButton()
                }
            }
            .sheet(isPresented: $showAdd) {
                AddTodoView()
            }
        }
    }

    func deleteTodos(at offsets: IndexSet) {
        for index in offsets {
            context.delete(pendingTodos[index])
        }
        // ModelContext 会在合适时机自动保存
        // 也可手动：try? context.save()
    }
}
```

### Todo 行视图

```swift
struct TodoRow: View {
    // 直接持有 @Model 实例，修改属性会自动触发 UI 刷新
    let todo: Todo

    @Environment(\.modelContext) private var context

    var body: some View {
        HStack {
            Image(systemName: todo.isCompleted ? "checkmark.circle.fill" : "circle")
                .foregroundStyle(todo.isCompleted ? .green : .secondary)
                .onTapGesture {
                    // 直接修改属性，SwiftData 自动追踪变化
                    todo.isCompleted.toggle()
                }

            VStack(alignment: .leading) {
                Text(todo.title)
                    .strikethrough(todo.isCompleted)
                if let tag = todo.tag {
                    Text(tag.name)
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
        }
    }
}
```

### 添加视图

```swift
struct AddTodoView: View {
    @Environment(\.modelContext) private var context
    @Environment(\.dismiss) private var dismiss

    @State private var title = ""
    // 查询所有 Tag 供选择
    @Query var tags: [Tag]
    @State private var selectedTag: Tag?

    var body: some View {
        NavigationStack {
            Form {
                TextField("标题", text: $title)
                Picker("标签", selection: $selectedTag) {
                    Text("无").tag(nil as Tag?)
                    ForEach(tags) { tag in
                        Text(tag.name).tag(tag as Tag?)
                    }
                }
            }
            .navigationTitle("添加 Todo")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("取消") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        guard !title.isEmpty else { return }
                        let todo = Todo(title: title)
                        todo.tag = selectedTag
                        context.insert(todo)   // 插入到 context
                        dismiss()
                    }
                }
            }
        }
    }
}
```

---

## 线程安全：跨 Actor 使用

`ModelContext` **不是线程安全的**，一个 context 只能在创建它的线程/actor 上使用。

### 正确做法：各 actor 用独立 context

```swift
// 后台处理任务，不用主线程的 context
actor DataSyncService {
    let container: ModelContainer

    init(container: ModelContainer) {
        self.container = container
    }

    func syncFromServer(items: [ServerItem]) async throws {
        // 创建属于当前 actor 的独立 context
        let context = ModelContext(container)

        for item in items {
            let todo = Todo(title: item.title)
            context.insert(todo)
        }

        try context.save()  // 后台 context 需手动 save
    }
}

// 主线程调用
struct ContentView: View {
    @Environment(\.modelContainer) private var container  // 取 container

    func syncData() {
        Task {
            let service = DataSyncService(container: container)
            try await service.syncFromServer(items: [...])
        }
    }
}
```

### 跨线程传递对象：用 PersistentIdentifier

```swift
// ❌ 错误：把 context 里的对象传给另一个线程
Task.detached {
    print(todo.title)  // crash！todo 绑定在主线程 context
}

// ✅ 正确：传 ID，在目标线程重新 fetch
let id = todo.persistentModelID   // PersistentIdentifier 是值类型，可跨线程

Task.detached {
    let bgContext = ModelContext(container)
    if let todo = bgContext.model(for: id) as? Todo {
        print(todo.title)  // 安全
    }
}
```

---

## @Query 进阶

```swift
// 动态 filter（需要在初始化时传入）
struct FilteredList: View {
    @Query var todos: [Todo]

    // 通过 init 动态传入 filter
    init(showCompleted: Bool) {
        _todos = Query(
            filter: #Predicate<Todo> { todo in
                showCompleted ? todo.isCompleted : !todo.isCompleted
            },
            sort: \Todo.createdAt
        )
    }
}
```

---

## 数据迁移

```swift
// 模型版本管理
enum TodoSchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [Todo.self] }

    @Model class Todo {
        var title: String
        init(title: String) { self.title = title }
    }
}

enum TodoSchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] { [Todo.self] }

    @Model class Todo {
        var title: String
        var priority: Int = 0   // 新增字段
        init(title: String) { self.title = title }
    }
}

// 迁移计划
enum TodoMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [TodoSchemaV1.self, TodoSchemaV2.self]
    }

    static var stages: [MigrationStage] {
        [MigrationStage.lightweight(fromVersion: TodoSchemaV1.self,
                                    toVersion: TodoSchemaV2.self)]
    }
}

// 使用迁移计划
.modelContainer(for: Todo.self, migrationPlan: TodoMigrationPlan.self)
```

---

## 与 Core Data 对比

| | SwiftData | Core Data |
|---|---|---|
| 最低系统 | iOS 17+ | iOS 3+ |
| 模型定义 | `@Model` 宏 | `.xcdatamodeld` 文件 |
| 查询 | `@Query` + `#Predicate` | `@FetchRequest` + `NSPredicate` |
| 跨线程 | 各自创建 `ModelContext` | `performBackgroundTask` |
| 混用 | 可以，底层共用 SQLite | — |

---

## 结合 @Observable 使用

### 核心区别

- `@Model` = 持久化（存磁盘），底层已内置 `@Observable` 能力
- `@Observable` = 内存状态观察（不持久化）

`@Model` 对象可以直接被 SwiftUI 观察，不需要额外套 ViewModel。

### @Model 直接当 @Observable 用

```swift
// View 直接持有 @Model 实例，let 就够，不需要 @StateObject / @ObservedObject
struct TodoRow: View {
    let todo: Todo   // @Model 已是 Observable

    var body: some View {
        HStack {
            Text(todo.title)
            Toggle("", isOn: $todo.isCompleted)  // 直接双向绑定
        }
    }
}
```

### @Observable ViewModel + SwiftData 配合

ViewModel 管理临时 UI 状态，SwiftData 管持久化：

```swift
@Observable
class TodoViewModel {
    var searchText = ""
    var isShowingAdd = false
    private let context: ModelContext

    init(context: ModelContext) { self.context = context }

    func addTodo(title: String) {
        context.insert(Todo(title: title))
    }
}

struct ContentView: View {
    @Environment(\.modelContext) private var context
    @State private var viewModel: TodoViewModel?  // @Observable 用 @State 持有
    @Query var todos: [Todo]

    var body: some View {
        let vm = viewModel!
        List(todos.filter {
            vm.searchText.isEmpty || $0.title.contains(vm.searchText)
        }) { todo in
            TodoRow(todo: todo)
        }
        .searchable(text: Binding(get: { vm.searchText }, set: { vm.searchText = $0 }))
        .onAppear { viewModel = viewModel ?? TodoViewModel(context: context) }
    }
}
```

### 编辑草稿模式：@Observable 做临时状态

```swift
@Observable
class EditState {
    var draftTitle = ""
    var isDirty = false
}

struct EditTodoView: View {
    let todo: Todo                         // 持久化对象
    @State private var edit = EditState()  // 临时 UI 状态
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        Form {
            TextField("标题", text: $edit.draftTitle)
                .onChange(of: edit.draftTitle) {
                    edit.isDirty = edit.draftTitle != todo.title
                }
        }
        .onAppear { edit.draftTitle = todo.title }
        .toolbar {
            ToolbarItem(placement: .confirmationAction) {
                Button("保存") {
                    todo.title = edit.draftTitle  // 写回 @Model，自动追踪
                    dismiss()
                }
                .disabled(!edit.isDirty)
            }
        }
    }
}
```

### 对比表

| | `@Observable` | `@Model` |
|---|---|---|
| 持久化 | 否（内存） | 是（SQLite） |
| SwiftUI 观察 | 属性级精确更新 | 同上（内置） |
| View 持有 | `@State var vm: MyVM` | 直接 `let todo: Todo` |
| 跨线程 | 看 actor 设计 | 绑定 ModelContext 线程 |

> **原则：临时 UI 状态用 `@Observable`，需要持久化的数据用 `@Model`，两者可以在同一 View 共存。**

---

## 相关

- [[swiftui-state|SwiftUI 状态观察]] — @Observable 与 SwiftData 配合
- [[../编程语言/swift-core|Swift 核心概念]] — Actor 隔离、线程安全
