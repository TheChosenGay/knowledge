# SwiftData 中 ModelContext 的并发使用

## 主题定位

这不是单纯的 SwiftData API 用法问题，而是一个交叉主题，综合了：

- Swift 并发：actor 隔离、`Task.detached`、跨 actor 访问
- 持久化系统：context、事务、save、底层 store 同步
- SwiftData 本身：`ModelContainer`、`ModelContext`、`@Model`、`@Query`
- SwiftUI 刷新机制：环境注入、查询结果变化、View 重建

核心理解：`ModelContainer` 更像持久化容器，可以共享；`ModelContext` 更像一次执行域/事务上下文，应该和 actor/任务边界保持一致。

## 后台批量导入/大量写入是什么场景

常见场景包括：

- 首次启动从 JSON 导入大量数据
- 从服务器同步大量记录
- 离线缓存接口返回的数据
- 解析本地大文件后写入数据库
- 图片、视频元数据批量入库
- 数据迁移、清洗、去重

如果这些操作直接放在 SwiftUI 环境中的 `modelContext` 上执行，可能导致：

- UI 卡顿
- SwiftUI 刷新频繁
- `@Query` 结果不断变化
- 内存短时间上涨
- 主线程被大量对象追踪占用

因此大量写入通常应该放到后台 context 中执行。

## 不要在后台使用 MainActor 的同一个 ModelContext

SwiftUI 中常见的 context 来自环境：

```swift
@Environment(\.modelContext) private var modelContext
```

这个 context 通常用于 UI 层，也就是 MainActor 场景。

不要这样做：

```swift
Task.detached {
    modelContext.insert(item)
}
```

原因：

- `ModelContext` 不适合跨 actor / 跨线程共享
- 它内部会追踪对象状态
- UI 读和后台写共用同一个 context，容易造成数据竞争
- `@Model` 对象本身也不应该随意跨 actor 传递

## 推荐做法：后台创建新的 ModelContext

`ModelContainer` 可以共享，`ModelContext` 应该按执行域隔离。

```swift
func importData(container: ModelContainer, items: [ItemDTO]) {
    Task.detached {
        let context = ModelContext(container)

        for dto in items {
            let model = Item(name: dto.name)
            context.insert(model)
        }

        try? context.save()
    }
}
```

可以理解为：

```text
MainActor/UI 使用环境里的 modelContext
Background 任务使用新的 ModelContext(container)
```

## UI 正在 @Query，后台正在导入，会不会冲突

如果使用不同 context：

```text
Main/UI context      -> @Query 读取
Background context  -> 批量 insert/save
```

这是正常场景。

一般流程是：

1. 后台 context 插入数据
2. 后台 context `save()`
3. 底层持久化存储更新
4. UI context / `@Query` 感知变化
5. SwiftUI 刷新列表

UI 通常不会直接看到后台 context 中尚未 save 的临时对象。只有保存之后，变化才会同步到 store，再被 UI 观察到。

## 大批量导入对 UI 的影响

不会因为“UI query + 后台 insert”天然冲突，但数据量大时可能导致：

- UI 列表突然刷新大量数据
- `@Query` 重新计算
- View body 大量重建
- 内存短时间上涨
- 滚动卡顿

因此大批量写入建议分批保存：

```swift
let batchSize = 500

for index in items.indices {
    let model = Item(name: items[index].name)
    context.insert(model)

    if index % batchSize == 0 {
        try context.save()
    }
}

try context.save()
```

同时避免 UI 查询全量数据：

```swift
@Query var allItems: [Item]
```

如果数据很多，更适合使用过滤、排序、分页或限制展示范围。

## 不要跨 actor 传递 @Model 对象

不要这样：

```swift
let user: User = ...

Task.detached {
    print(user.name)
}
```

更推荐传：

```swift
let id = user.persistentModelID
```

或者传 DTO：

```swift
struct UserDTO: Sendable {
    let name: String
}
```

后台任务中再用自己的 context fetch / insert。

## 总结

```text
ModelContainer 可以全局共享。
ModelContext 不应该随意跨 actor 共享。
UI 使用环境里的 modelContext。
后台批量导入使用新的 ModelContext(container)。
后台 save 后，UI 的 @Query 再感知变化并刷新。
```
