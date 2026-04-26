# 第5章：Entity 系统

## 5.1 `Entity<T>` 是什么

### 5.1.1 GPUI 的智能指针模式

GPUI 用 `Entity<T>` 管理所有应用状态。它类似 `Rc<RefCell<T>>`，但在运行时借用检查之上增加了额外的能力：

- 类型安全的 `TypeId` 标记
- 自动的观察者通知机制
- 弱引用 `WeakEntity<T>`
- 与 `App` 的 EntityMap 集成

```rust
use gpui::{Entity, Context};

struct Counter {
    value: i32,
}

// 在上下文中创建 Entity
fn create_counter(cx: &mut impl AppContext) -> Entity<Counter> {
    cx.new(|_cx| Counter { value: 0 })
}
```

### 5.1.2 `Entity<T>` vs `Rc<RefCell<T>>`

| 特性 | `Entity<T>` | `Rc<RefCell<T>>` |
|------|-------------|------------------|
| 借用检查 | 运行时（lease 模式） | 运行时 |
| 观察者通知 | 内置 `cx.notify()` | 无 |
| 类型擦除 | `AnyEntity` 支持动态类型 | 无 |
| 弱引用 | `WeakEntity<T>` | `Weak<RefCell<T>>` |
| 生命周期管理 | EntityMap 自动回收 | Rc 引用计数 |
| 嵌套可变访问 | Lease 模式，同一 Entity 不可嵌套 | 嵌套 borrow 会 panic |

关键区别在于 lease 模式：当调用 `Entity::update()` 时，GPUI 将数据从 EntityMap 中移出到栈上（`Lease`），修改完成后再移回去。这避免了 `RefCell` 的运行时 panic，改为编译期保证——你不可能在更新一个 Entity 的同时再次更新它。

```rust
// EntityMap::lease() — 将数据从映射中取出
pub fn lease<T>(&mut self, pointer: &Entity<T>) -> Lease<T> {
    let entity = self.entities.remove(pointer.entity_id).unwrap();
    Lease { entity, id: pointer.entity_id }
}

// EntityMap::end_lease() — 将数据放回映射
pub fn end_lease<T>(&mut self, mut lease: Lease<T>) {
    self.entities.insert(lease.id, lease.entity.take().unwrap());
}
```

### 5.1.3 实体身份：`EntityId`

每个 Entity 有唯一的 `EntityId`，由 slotmap 分配：

```rust
use gpui::EntityId;

slotmap::new_key_type! {
    pub struct EntityId;
}

impl EntityId {
    pub fn as_non_zero_u64(self) -> NonZeroU64;
    pub fn as_u64(self) -> u64;
}
```

`EntityId` 用于：
- 在 `SubscriberSet` 中标识观察者
- 追踪哪些 Entity 在当前帧被访问过
- 缓存失效判断

```rust
let counter = cx.new(|_cx| Counter { value: 0 });
let id: EntityId = counter.entity_id();
println!("Entity ID: {}", id); // 输出数字
```

## 5.2 创建实体

### 5.2.1 `cx.new(|cx| MyEntity { ... })`

```rust
use gpui::{App, Context, Entity};

struct TodoList {
    items: Vec<String>,
    filter: String,
}

impl TodoList {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            items: Vec::new(),
            filter: String::new(),
        }
    }
}

fn create_todos(cx: &mut App) -> Entity<TodoList> {
    cx.new(|cx| TodoList::new(cx))
}
```

创建时的 `Context<T>` 可以用于：
- 创建子 Entity
- 注册 `observe`/`subscribe`
- 设置全局状态
- 注册 `on_release` 回调

```rust
struct ParentView {
    child: Entity<ChildView>,
    _subscription: gpui::Subscription,
}

impl ParentView {
    fn new(cx: &mut Context<Self>) -> Self {
        let child = cx.new(|_cx| ChildView { count: 0 });

        // 观察子 Entity 的变化
        let subscription = cx.observe(&child, |this, child, cx| {
            cx.notify(); // 子 Entity 变化时通知父
        });

        Self {
            child,
            _subscription: subscription,
        }
    }
}
```

### 5.2.2 闭包中的初始化逻辑

```rust
struct AppState {
    config: Config,
    data: Entity<DataModel>,
}

fn init_app(cx: &mut App) -> Entity<AppState> {
    cx.new(|cx| {
        // 在闭包内可以创建其他 Entity
        let data = cx.new(|_cx| DataModel::default());

        AppState {
            config: Config::load(),
            data,
        }
    })
}
```

### 5.2.3 `reserve_entity()` 延迟创建

当需要循环引用时，先预留 slot 再填充：

```rust
use gpui::{Context, Entity, Reservation};

struct Node {
    value: i32,
    next: Option<Entity<Node>>,
}

fn create_cycle(cx: &mut Context<Self>) -> Entity<Node> {
    // 先预留 slot，获取 EntityId
    let reservation = cx.reserve_entity::<Node>();

    // 用 reservation 创建第一个 Node（此时 next 可以为 None 或指向自己）
    let first = cx.insert_entity(reservation, |cx| {
        Node {
            value: 1,
            next: None, // 延迟设置
        }
    });

    // 现在可以用 first 创建第二个
    let second = cx.new(|cx| Node {
        value: 2,
        next: Some(first.clone()),
    });

    // 但注意：Entity 创建后不可变，循环引用需要用其他方式
    // 实际场景中常用 WeakEntity 或 Option + 延迟初始化
    first
}
```

更实用的场景是提前获取 `EntityId`：

```rust
fn init_with_id(cx: &mut Context<Self>) -> (Entity<MyView>, EntityId) {
    let reservation = cx.reserve_entity::<MyView>();
    let id = reservation.entity_id();

    let entity = cx.insert_entity(reservation, |cx| MyView { id });
    (entity, id)
}
```

## 5.3 读写实体

### 5.3.1 `Entity::read()` 获取只读访问

```rust
use gpui::{Entity, App};

struct Counter {
    value: i32,
}

fn read_counter(counter: &Entity<Counter>, cx: &App) {
    // 直接读取
    let value = counter.read(cx).value;
    println!("Counter value: {}", value);
}

// 使用回调方式读取
fn read_with_callback(counter: &Entity<Counter>, cx: &impl AppContext) -> i32 {
    counter.read_with(cx, |counter, _app| counter.value)
}
```

`read()` 返回 `&T`，在读取期间其他代码仍然可以读取同一个 Entity。

### 5.3.2 `Entity::update()` 获取可变访问

```rust
fn increment_counter(counter: &Entity<Counter>, cx: &mut impl AppContext) {
    counter.update(cx, |counter, cx| {
        counter.value += 1;
        cx.notify(); // 通知观察者
    });
}
```

`update()` 使用 lease 模式：
1. 从 `EntityMap` 中移除 Entity 数据
2. 传入闭包进行可变访问
3. 闭包返回后数据放回 `EntityMap`

闭包中的 `Context<T>` 是专用上下文，可以：
- 调用 `cx.notify()` 通知观察者
- 调用 `cx.emit(event)` 发射事件
- 创建新的 Entity

### 5.3.3 借用检查与运行时 panic 场景

虽然 lease 模式避免了 `RefCell` 风格的 panic，但仍有需要注意的场景：

```rust
// 正确：链式更新不同 Entity
a.update(cx, |a, cx| {
    b.update(cx, |b, cx| {
        // 可以，a 和 b 是不同 Entity
    });
});

// 错误：在同一 Entity 的 update 闭包内调用 Entity::read(cx)
// 读取自身不会 panic（lease 模式下数据已在栈上）
// 但不能同时 update 两个相同的 Entity
```

在 GPUI 的 lease 模式下，以下操作会 panic：

```rust
// 在同一个 update 闭包内再次 update 同一个 Entity 会 panic
counter.update(cx, |counter, cx| {
    // counter 已经在栈上，不能再通过 Entity::update 访问
    // 下面的调用会触发 "cannot update Counter while it is already being updated"
    // counter.update(cx, |counter, cx| {}); // panic!
});
```

### 5.3.4 `WeakEntity<T>` 弱引用与生命周期

```rust
use gpui::{Entity, WeakEntity, Context, App};

struct Parent {
    child: Entity<Child>,
    weak_child: WeakEntity<Child>,
}

fn example(cx: &mut Context<Parent>) {
    let child = cx.new(|_cx| Child { value: 42 });
    let weak = child.downgrade();

    // 升级弱引用
    if let Some(strong) = weak.upgrade() {
        strong.update(cx, |child, cx| {
            child.value += 1;
        });
    } else {
        // Entity 已被销毁
        println!("Child entity is gone");
    }

    // 检查是否仍可升级
    if weak.is_upgradable() {
        println!("Child is still alive");
    }
}
```

`WeakEntity` 的典型使用场景：异步任务中避免持有强引用。

```rust
struct DataLoader {
    data: String,
}

impl DataLoader {
    fn load_in_background(&self, cx: &mut Context<Self>) {
        cx.spawn(|this, mut cx| async move {
            // 使用 WeakEntity 避免在异步任务生命周期内阻止 Entity 销毁
            let result = fetch_data().await;

            // 尝试升级，如果 Entity 已被销毁则跳过
            if let Some(this) = this.upgrade() {
                this.update(&mut cx, |this, cx| {
                    this.data = result;
                    cx.notify();
                }).ok();
            }
        }).detach();
    }
}
```

## 5.4 实体生命周期

### 5.4.1 引用计数与自动回收

`Entity<T>` 内部使用原子引用计数（`AtomicUsize`）跟踪强引用数量：

```rust
// 来自 GPUI 源码
pub(crate) struct EntityRefCounts {
    counts: SlotMap<EntityId, AtomicUsize>,
    dropped_entity_ids: Vec<EntityId>,
}
```

当最后一个 `Entity<T>` 或 `AnyEntity` 被 drop 时：
1. 引用计数减到 0
2. `EntityId` 被加入 `dropped_entity_ids`
3. 下一个 effect cycle 中 `EntityMap::take_dropped()` 清理数据

```rust
fn entity_lifecycle(cx: &mut App) {
    let entity = cx.new(|_cx| Counter { value: 0 });
    let weak = entity.downgrade();

    // Entity 存活
    assert!(weak.upgrade().is_some());

    drop(entity);

    // Entity 被销毁
    assert!(weak.upgrade().is_none());
}
```

### 5.4.2 何时实体会被销毁

Entity 在以下时机被销毁：
- 所有 `Entity<T>` 和 `AnyEntity` 强引用都被 drop
- 窗口关闭时，其根视图的所有权丢失（如果没有其他引用）

```rust
struct WindowRoot {
    child: Entity<ChildView>,
}

// 当窗口关闭，WindowRoot 被 drop
// 如果没有其他地方持有 child 的强引用，ChildView 也被 drop
```

### 5.4.3 悬垂引用的防护

`WeakEntity::upgrade()` 返回 `Option<Entity<T>>`，在 Entity 已销毁时返回 `None`：

```rust
fn safe_update(weak: &WeakEntity<Counter>, cx: &mut impl AppContext) {
    // 总是先 upgrade 再 update
    let _ = weak.update(cx, |counter, cx| {
        counter.value += 1;
        cx.notify();
    });
    // WeakEntity::update 内部处理了 upgrade 失败的情况
    // 返回 Result，失败时返回 Err
}
```

`WeakEntity` 自身也提供 `update` 方法：

```rust
impl<T> WeakEntity<T> {
    pub fn update<C, R>(&self, cx: &mut C, f: impl FnOnce(&mut T, &mut Context<T>) -> R) -> Result<R>
    where
        C: AppContext,
    {
        // 内部先 upgrade，失败则返回 Err
    }
}
```

## 5.5 Entity 的最佳实践

### 5.5.1 什么应该放入 Entity

**应该放入 Entity 的数据：**
- 需要跨视图共享的状态
- 需要异步任务更新的状态
- 需要观察者通知的状态
- 较大或较复杂的数据结构

```rust
// 好的做法：文档内容作为 Entity
struct Document {
    text: Rope,       // 大文本缓冲区
    cursor: Position,
}

// 多个编辑器视图可以共享同一个 Document Entity
struct EditorView {
    document: Entity<Document>,
    scroll_offset: f32,  // 视图特有状态，不放入 Entity
}
```

**不应该放入 Entity 的数据：**
- 纯计算属性（可以从其他数据推导）
- 仅在渲染时使用的临时状态
- 简单的值类型（直接拷贝即可）

```rust
// 不好的做法：每个小状态都做成 Entity
struct BadDesign {
    count: Entity<i32>,          // 太细碎
    name: Entity<String>,         // 不需要
}

// 好的做法：相关状态聚合到一个 Entity
struct GoodDesign {
    count: i32,
    name: String,
}
```

### 5.5.2 Entity 与 View 的对应关系

在 GPUI 中，View 是实现 `Render` trait 的 Entity。典型的层级关系：

```
App
 └── Window
      └── RootView (Entity<AppLayout>)
           ├── SidebarView (Entity<Sidebar>)
           ├── EditorView (Entity<Editor>)
           │    └── Document (Entity<Document>)  <-- 非 View 的 Entity
           └── StatusBarView (Entity<StatusBar>)
```

```rust
struct AppLayout {
    sidebar: Entity<Sidebar>,
    editor: Entity<Editor>,
    status_bar: Entity<StatusBar>,
}

impl Render for AppLayout {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .child(self.sidebar.clone())     // Entity<View> 自动转为元素
            .child(self.editor.clone())
            .child(self.status_bar.clone())
    }
}
```

### 5.5.3 避免 Entity 过多或过少

**Entity 过多的问题：**
- 每个 Entity 有运行时开销（EntityMap 查找、引用计数）
- 过多的观察者订阅增加通知成本
- 代码复杂度上升

**Entity 过少的问题：**
- 大范围的不必要重新渲染
- 状态耦合过度，难以独立更新
- 异步任务难以隔离

经验法则：
- 需要独立更新频率的部分拆分为独立 Entity
- 同一逻辑单元的状态放在同一个 Entity
- 异步任务的状态放到专用 Entity

```rust
// 合理拆分
struct FileTree {           // 独立更新频率：低
    entries: Vec<Entry>,
}

struct Editor {             // 独立更新频率：高
    document: Entity<Document>,
    cursor: Position,
}

struct StatusBar {          // 独立更新频率：中
    mode: EditorMode,
    position: Position,
}
```
