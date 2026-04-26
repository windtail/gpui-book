# 第7章：数据流与所有权

## 7.1 GPUI 的数据流模型

### 7.1.1 数据流向图：Entity → View → Element

GPUI 中的数据遵循单向流动：

```
数据源（Entity）
    |
    v
View（实现 Render，持有 Entity 或其数据）
    |
    v
Element 树（div、text 等，纯描述性结构）
    |
    v
GPU Scene（PaintOperation 集合）
```

数据不会反向流动。Element 树没有状态，它只是 Entity 状态的投影。

```rust
struct Counter {
    value: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        // Element 树是 Entity 状态的纯函数投影
        div()
            .child(format!("Value: {}", self.value))
    }
}
```

### 7.1.2 单向数据流原则

GPUI 严格遵循单向数据流：

1. **数据存储在 Entity 中**
2. **View 的 render() 方法读取 Entity 数据并生成 Element 树**
3. **用户交互触发回调，回调修改 Entity 数据**
4. **`cx.notify()` 标记 View 为 dirty**
5. **下一帧重新渲染 dirty 的 View**

```
┌─────────────┐     render      ┌──────────┐
│   Entity    │ ──────────────► │  Element  │
│   (State)   │                 │   (View)  │
└──────┬──────┘                 └──────────┘
       │                              │
       │   user action                │
       │◄─────────────────────────────│
       │
       ▼
  cx.notify()
  触发重新渲染
```

### 7.1.3 为什么没有双向绑定

GPUI 不提供双向绑定，原因在于：
- 双向绑定隐藏了数据流向，使状态变化难以追踪
- 明确的数据流让每次变更的来源可追溯
- 编译器和开发者可以更容易推理状态

```rust
// 没有这样的写法：
// text_input.bind(&mut self.text)

// 取而代之的是显式的回调：
text_input
    .on_change(cx.listener(|this, new_text, _, _| {
        this.text = new_text;
        cx.notify();
    }))
```

## 7.2 update/read 模式

### 7.2.1 读多写少的典型场景

```rust
struct Dashboard {
    stats: Entity<Stats>,
    charts: Entity<Charts>,
}

impl Render for Dashboard {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 多次读取，零次写入
        let stats = self.stats.read(cx);
        let charts = self.charts.read(cx);

        div()
            .child(format!("Users: {}", stats.user_count))
            .child(format!("Revenue: {}", stats.revenue))
            .child(charts.clone())
    }
}
```

`Entity::read()` 比 `Entity::update()` 更轻量——它不触发 lease 模式，多个 `read()` 可以同时存在。

```rust
// 可以同时读取多个 Entity
fn aggregate_data(
    a: &Entity<DataSource>,
    b: &Entity<DataSource>,
    cx: &App,
) -> f64 {
    let data_a = a.read(cx);
    let data_b = b.read(cx);
    // 同时持有两个只读引用是安全的
    data_a.total() + data_b.total()
}
```

### 7.2.2 嵌套 update 的陷阱

GPUI 的 lease 模式使得同一个 Entity 不能被嵌套 update：

```rust
// 错误：嵌套 update 同一个 Entity
fn bad(cx: &mut impl AppContext, counter: &Entity<Counter>) {
    counter.update(cx, |counter, cx| {
        // counter 的数据已被移出到栈上
        // 再次 update 会触发 "cannot update Counter while it is already being updated"
        // counter.update(cx, |counter, cx| {
        //     counter.value += 1;
        // }); // panic!
    });
}

// 正确：如果需要嵌套更新不同的 Entity，顺序很重要
fn good(
    cx: &mut impl AppContext,
    a: &Entity<EntityA>,
    b: &Entity<EntityB>,
) {
    a.update(cx, |a, cx| {
        // 嵌套 update 不同的 Entity 是安全的
        b.update(cx, |b, cx| {
            // ...
        });
    });
}
```

### 7.2.3 批量更新技巧

当需要同时修改多个 Entity 时，避免在循环中逐个 update：

```rust
// 低效：逐个 update，每次都触发通知
fn update_all_counters_bad(
    counters: &[Entity<Counter>],
    cx: &mut impl AppContext,
) {
    for counter in counters {
        counter.update(cx, |counter, cx| {
            counter.value += 1;
            cx.notify(); // 每次都触发重新渲染
        });
    }
}

// 高效：批量收集更新
struct CounterBatch {
    counters: Vec<Entity<Counter>>,
}

impl CounterBatch {
    fn increment_all(&mut self, cx: &mut Context<Self>) {
        // 一次性修改所有值
        for counter in &self.counters {
            counter.update(cx, |counter, _cx| {
                counter.value += 1;
                // 不调用 cx.notify()，因为 Counter 不知道观察者是谁
            });
        }
        // 最后一次性通知
        cx.notify();
    }
}
```

`Entity::as_mut()` 用于获取可变引用而不触发回调：

```rust
// 使用 as_mut() 获取可变借用，适合需要直接修改后再手动通知的场景
fn batch_edit<C: AppContext>(entity: &Entity<Counter>, cx: &mut C) {
    let counter = entity.as_mut(cx);
    counter.value += 10;
    counter.value *= 2;
    // as_mut 返回的类型允许调用 notify
    // counter.notify();
}
```

## 7.3 数据流模式

### 7.3.1 状态下沉：将数据传递给子 View

父 View 持有数据，通过属性传递给子 View：

```rust
struct ParentView {
    items: Vec<String>,
}

impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .children(
                self.items.iter().enumerate().map(|(i, item)| {
                    // 子 View 接收不可变数据
                    ChildRow::new(i, item.clone())
                }),
            )
    }
}

struct ChildRow {
    index: usize,
    text: String,
}

impl ChildRow {
    fn new(index: usize, text: String) -> Self {
        Self { index, text }
    }
}

impl Render for ChildRow {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(format!("{}. {}", self.index + 1, self.text))
    }
}
```

当子 View 需要修改父的数据时，使用回调：

```rust
struct ChildRow {
    index: usize,
    text: String,
    on_delete: Option<Box<dyn FnMut()>>,
}

impl ChildRow {
    fn new(index: usize, text: String, on_delete: impl FnMut() + 'static) -> Self {
        Self {
            index,
            text,
            on_delete: Some(Box::new(on_delete)),
        }
    }
}

impl Render for ChildRow {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(&self.text)
            .child(
                button()
                    .label("Delete")
                    .on_click(cx.listener(|this, _, _, cx| {
                        if let Some(ref mut cb) = this.on_delete {
                            cb();
                        }
                    })),
            )
    }
}

impl ParentView {
    fn delete_item(&mut self, index: usize, cx: &mut Context<Self>) {
        self.items.remove(index);
        cx.notify();
    }
}
```

### 7.3.2 事件上浮：从子 View 向上传递事件

子 View 通过 `emit` 向父 View 传递事件：

```rust
// 子 View 定义事件
enum RowEvent {
    Deleted(usize),
    Edited(usize, String),
}

struct ChildRow {
    index: usize,
    text: String,
}

impl EventEmitter<RowEvent> for ChildRow {}

impl Render for ChildRow {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(&self.text)
            .child(
                button()
                    .label("X")
                    .on_click(cx.listener(|this, _, _, cx| {
                        cx.emit(RowEvent::Deleted(this.index));
                    })),
            )
    }
}

// 父 View 订阅子 View 的事件
struct ParentView {
    rows: Vec<Entity<ChildRow>>,
    _subs: Vec<gpui::Subscription>,
}

impl ParentView {
    fn new(cx: &mut Context<Self>) -> Self {
        let mut rows = Vec::new();
        let mut subs = Vec::new();

        for i in 0..5 {
            let row = cx.new(|_cx| ChildRow {
                index: i,
                text: format!("Item {}", i),
            });

            // 订阅事件
            let sub = cx.subscribe(&row, |this, row, event, cx| {
                match event {
                    RowEvent::Deleted(index) => {
                        this.rows.retain(|r| r != row);
                        cx.notify();
                    }
                    RowEvent::Edited(index, new_text) => {
                        cx.notify();
                    }
                }
            });

            rows.push(row);
            subs.push(sub);
        }

        Self { rows, _subs: subs }
    }
}
```

### 7.3.3 共享状态：Global vs Entity

选择哪种方式存储全局状态？

**使用 Global 的场景：**
- 应用级配置
- 主题设置
- 整个应用唯一的单例

```rust
struct ThemeConfig {
    dark_mode: bool,
    font_size: f32,
}

impl Global for ThemeConfig {}

// 任意地方访问
fn get_font_size(cx: &App) -> f32 {
    cx.global::<ThemeConfig>().font_size
}
```

**使用 Entity 的场景：**
- 需要观察者通知
- 有多个实例
- 需要异步更新

```rust
struct SharedPreferences {
    theme: Theme,
    language: String,
}

// 作为 Entity，可以观察其变化
struct ThemeObserver {
    prefs: Entity<SharedPreferences>,
}

impl ThemeObserver {
    fn new(prefs: Entity<SharedPreferences>, cx: &mut Context<Self>) -> Self {
        // 观察 Entity 的变化
        cx.observe(&prefs, |this, prefs, cx| {
            // prefs 变化时自动响应
            println!("Preferences changed");
        }).detach();

        Self { prefs }
    }
}
```

对比：

| 特性 | Global | Entity |
|------|--------|--------|
| 实例数量 | 每个类型一个 | 可以多个 |
| 观察者通知 | 通过 `notify_global_observers` | 内置 `cx.notify()` |
| 创建时机 | `set_global()` | `cx.new()` |
| 异步访问 | 可以（通过全局） | 通过 `WeakEntity` |
| 类型安全 | `Global` marker trait | 类型参数 |

## 7.4 常见反模式

### 7.4.1 在 render 中修改状态

```rust
// 错误：在 render 中修改状态
struct BadView {
    data: Vec<String>,
    loaded: bool,
}

impl Render for BadView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // render 中修改状态 — 可能导致无限渲染循环
        if !self.loaded {
            self.loaded = true;
            self.data.push("Loaded".to_string());
        }
        div().children(&self.data)
    }
}

// 正确：在初始化或事件中修改
impl BadView {
    fn new(cx: &mut Context<Self>) -> Self {
        let mut this = Self {
            data: Vec::new(),
            loaded: false,
        };
        this.loaded = true;
        this.data.push("Loaded".to_string());
        this
    }
}
```

如果需要在渲染完成后执行操作，使用 `on_next_frame`：

```rust
fn do_after_render(cx: &mut Context<Self>, window: &mut Window) {
    cx.on_next_frame(window, |this, window, cx| {
        // 在下一帧执行，不会干扰当前渲染
        this.some_flag = true;
        cx.notify();
    });
}
```

### 7.4.2 Entity 之间的循环引用

```rust
// 错误：循环强引用导致内存泄漏
struct A {
    b: Option<Entity<B>>, // A 持有 B
}

struct B {
    a: Option<Entity<A>>, // B 持有 A — 永远无法释放
}

// 正确：使用 WeakEntity 打破循环
struct A {
    b: Option<Entity<B>>,
}

struct B {
    a: Option<WeakEntity<A>>, // 弱引用，不阻止 A 被释放
}

impl B {
    fn access_a(&self, cx: &App) -> Option<&A> {
        self.a.as_ref().and_then(|weak| {
            // 临时升级
            weak.upgrade().map(|strong| strong.read(cx))
        })
    }
}
```

parent-child 关系中，子元素通常使用 `WeakEntity` 指向父元素：

```rust
struct Editor {
    document: Entity<Document>,
}

struct Document {
    editors: Vec<WeakEntity<Editor>>, // 文档知道哪些编辑器在引用它
}

impl Document {
    fn notify_editors(&mut self, cx: &mut Context<Self>) {
        // 清理已销毁的编辑器引用
        self.editors.retain(|weak| weak.is_upgradable());

        for editor in &self.editors {
            if let Some(editor) = editor.upgrade() {
                editor.update(cx, |editor, cx| {
                    cx.notify();
                });
            }
        }
    }
}
```

### 7.4.3 忘记调用 `cx.notify()`

```rust
// 错误：修改了状态但没有通知
struct Counter {
    value: i32,
}

impl Counter {
    fn increment_bad(&mut self, cx: &mut Context<Self>) {
        self.value += 1;
        // 忘记 cx.notify() — UI 不会更新
    }

    fn increment_good(&mut self, cx: &mut Context<Self>) {
        self.value += 1;
        cx.notify(); // 标记此 View 需要重新渲染
    }
}
```

以下情况**不需要**手动调用 `cx.notify()`：

- 使用 `cx.observe()` 时，被观察的 Entity 调用 `notify()` 后，观察者会自动收到通知
- 在 `Entity::update()` 闭包中，如果该 Entity 是 View（实现 `Render`），GPUI 会在 update 后自动标记为 dirty

```rust
// Entity::update 后，如果 Entity 是 View，GPUI 自动处理重新渲染
counter.update(cx, |counter, cx| {
    counter.value += 1;
    // 不需要 cx.notify()，因为 View 类型的 update 会自动触发重绘
});
```

但以下情况**必须**调用：

- 在 `Render::render()` 中的事件回调里修改了状态
- 在 `observe` 回调中修改了观察者自身的状态
- 异步任务完成后更新状态

```rust
// 必须调用 cx.notify() 的场景
impl DataLoader {
    fn on_data_loaded(&mut self, data: String, cx: &mut Context<Self>) {
        self.data = Some(data);
        cx.notify(); // 必须手动通知
    }
}

// observe 回调中修改自身状态
impl MyView {
    fn new(cx: &mut Context<Self>) -> Self {
        let counter = cx.new(|_cx| Counter { value: 0 });

        cx.observe(&counter, |this, counter, cx| {
            // 这里修改了自身状态，需要 notify
            this.counter_display = format!("Count: {}", counter.read(cx).value);
            cx.notify(); // 必须
        }).detach();

        Self {
            counter,
            counter_display: String::new(),
        }
    }
}
```
