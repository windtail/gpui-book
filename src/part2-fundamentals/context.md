# 第6章：Context 类型

## 6.1 Context 的层次结构

GPUI 提供多种 Context 类型，每种类型适用于不同场景。

### 6.1.1 `App` —— 应用级引用

`App` 是应用状态的唯一所有者。在 `app.run(|cx| ...)` 回调中获得 `&mut App`。

```rust
use gpui::{Application, App, Entity};

fn main() {
    let app = Application::new();
    app.run(|cx: &mut App| {
        // 在 App 级别创建 Entity
        let counter = cx.new(|_cx| Counter { value: 0 });

        // 读取 Entity
        let value = counter.read(cx).value;

        // 更新 Entity
        counter.update(cx, |counter, cx| {
            counter.value += 1;
            cx.notify();
        });

        // 设置全局状态
        cx.set_global(AppSettings { theme: Theme::Dark });

        // 读取全局状态
        let settings = cx.global::<AppSettings>();
    });
}
```

### 6.1.2 `Context<T>` —— 实体专用上下文

`Context<T>` 是 Entity 操作时的上下文，持有对 `App` 的可变引用和当前 Entity 的弱引用。

```rust
pub struct Context<'a, T> {
    app: &'a mut App,
    entity_state: WeakEntity<T>,
}
```

`Context<T>` 通过 `Deref` 和 `DerefMut` 可以直接访问 `App` 的方法：

```rust
impl<'a, T> ops::Deref for Context<'a, T> {
    type Target = App;
    fn deref(&self) -> &Self::Target {
        self.app
    }
}

impl<'a, T> ops::DerefMut for Context<'a, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        self.app
    }
}
```

这意味着在 `Context<T>` 上可以调用所有 `App` 的方法，同时还有 Entity 专用的方法。

### 6.1.3 `AppContext` trait —— 统一接口

`AppContext` trait 统一了 `App`、`Context<T>`、`AsyncApp` 的能力子集：

```rust
pub trait AppContext {
    fn new<T: 'static>(&mut self, build: impl FnOnce(&mut Context<T>) -> T) -> Entity<T>;
    fn reserve_entity<T: 'static>(&mut self) -> Reservation<T>;
    fn insert_entity<T: 'static>(&mut self, reservation: Reservation<T>, build: impl FnOnce(&mut Context<T>) -> T) -> Entity<T>;
    fn update_entity<T, R>(&mut self, handle: &Entity<T>, update: impl FnOnce(&mut T, &mut Context<T>) -> R) -> R;
    fn read_entity<T, R>(&self, handle: &Entity<T>, read: impl FnOnce(&T, &App) -> R) -> R;
    fn background_spawn<R>(&self, future: impl Future<Output = R> + Send + 'static) -> Task<R>;
    fn read_global<G, R>(&self, callback: impl FnOnce(&G, &App) -> R) -> R;
}
```

函数签名使用 `impl AppContext` 而非具体类型，可以同时兼容多种上下文：

```rust
// 这个函数可以接受 App 或 Context<T>
fn create_and_configure(cx: &mut impl AppContext) -> Entity<MyView> {
    let entity = cx.new(|_cx| MyView::default());
    // 配置...
    entity
}
```

## 6.2 `Context<T>` 详解

### 6.2.1 `Deref<Target = App>` 的隐含能力

由于 `Context<T>` deref 到 `App`，你可以在 `Context` 上直接调用 `App` 的方法：

```rust
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 通过 Deref 访问 App 的方法
        let global = cx.global::<ThemeSettings>();

        // 创建新 Entity
        let child = cx.new(|_cx| ChildView {});

        // 观察全局变化
        cx.observe_global::<ThemeSettings>(|this, cx| {
            cx.notify();
        });

        // 在后台线程执行
        cx.background_spawn(async {
            let data = fetch_data().await;
        }).detach();

        div()
    }
}
```

### 6.2.2 `VisualContext` trait 与窗口操作

`VisualContext` 是需要窗口参与的操作接口，`WindowContext` 和 `ViewContext` 实现了它：

```rust
pub trait VisualContext: AppContext {
    type Result<T>;

    fn window_handle(&self) -> AnyWindowHandle;

    fn update_window_entity<T: 'static, R>(
        &mut self,
        entity: &Entity<T>,
        update: impl FnOnce(&mut T, &mut Window, &mut Context<T>) -> R,
    ) -> Self::Result<R>;

    fn new_window_entity<T: 'static>(
        &mut self,
        build_entity: impl FnOnce(&mut Window, &mut Context<T>) -> T,
    ) -> Self::Result<Entity<T>>;

    fn replace_root_view<V>(
        &mut self,
        handle: &WindowHandle<V>,
        build: impl FnOnce(&mut Window, &mut Context<V>) -> V,
    ) -> Self::Result<()>;

    fn focus(&mut self);
}
```

```rust
// ViewContext 实现 VisualContext
impl<V> VisualContext for ViewContext<'_, V> {
    type Result<T> = T; // 不会失败

    fn window_handle(&self) -> AnyWindowHandle {
        self.window.handle.into()
    }

    fn replace_root_view<T>(
        &mut self,
        handle: &WindowHandle<T>,
        build: impl FnOnce(&mut Window, &mut Context<T>) -> T,
    ) -> Self::Result<()> {
        // ...
    }
}
```

使用 `replace_root_view` 切换页面：

```rust
struct Router {
    current_page: Page,
}

enum Page {
    Home,
    Settings,
}

impl Router {
    fn navigate_to_settings(&mut self, cx: &mut Context<Self>, window: &mut Window) {
        self.current_page = Page::Settings;

        let handle = window.handle;
        // 替换窗口根视图
        // window.replace_root_view(...)
        cx.notify();
    }
}
```

### 6.2.3 `BorrowAppContext` 的使用场景

`BorrowAppContext` 是 `Global` 相关的操作的 trait：

```rust
pub trait BorrowAppContext {
    fn set_global<G: Global>(&mut self, global: G);
    fn try_global<G: Global>(&self) -> Option<&G>;
    fn global<G: Global>(&self) -> &G;
    fn global_mut<G: Global>(&mut self) -> &mut G;
    fn update_global<G: Global, R>(&mut self, f: impl FnOnce(&mut G, &mut Self) -> R) -> R;
    fn update_default_global<G: Global + Default>(&mut self);
    fn observe_global<G: Global>(&mut self, f: impl FnMut(&mut Self)) -> Subscription;
    fn notify_global_observers<G: Global>(&mut self);
}
```

`App` 和 `Context<T>` 都实现了 `BorrowAppContext`：

```rust
struct ThemeSettings {
    dark_mode: bool,
}

impl Global for ThemeSettings {}

fn setup_theme(cx: &mut impl BorrowAppContext) {
    cx.set_global(ThemeSettings { dark_mode: true });
}

fn toggle_theme(cx: &mut impl BorrowAppContext) {
    cx.update_global(|theme, cx| {
        theme.dark_mode = !theme.dark_mode;
        // 通知所有观察者
        cx.notify_global_observers::<ThemeSettings>();
    });
}
```

配合 `ReadGlobal` 和 `UpdateGlobal` trait 使用：

```rust
use gpui::{Global, ReadGlobal, UpdateGlobal, BorrowAppContext};

struct AppConfig {
    max_tabs: usize,
}

impl Global for AppConfig {}

fn init_config<C: BorrowAppContext>(cx: &mut C) {
    AppConfig::set_global(cx, AppConfig { max_tabs: 10 });
}

fn read_config<C: BorrowAppContext>(cx: &C) -> usize {
    AppConfig::global(cx).max_tabs
}

fn update_config<C: BorrowAppContext>(cx: &mut C) {
    AppConfig::update_global(cx, |config, cx| {
        config.max_tabs += 1;
    });
}
```

## 6.3 上下文方法分类

### 6.3.1 实体操作：`new()`、`observe()`、`subscribe()`

```rust
struct Parent {
    child: Entity<Child>,
    _observe_sub: gpui::Subscription,
    _subscribe_sub: gpui::Subscription,
}

impl Parent {
    fn new(cx: &mut Context<Self>) -> Self {
        let child = cx.new(|_cx| Child { count: 0 });

        // observe: 当 child 调用 cx.notify() 时触发
        let observe_sub = cx.observe(&child, |this, child, cx| {
            println!("Child notified, value = {}", child.read(cx).count);
            cx.notify();
        });

        // subscribe: 当 child 调用 cx.emit(ChildEvent::Xxx) 时触发
        let subscribe_sub = cx.subscribe(&child, |this, child, event, cx| {
            match event {
                ChildEvent::Updated => {
                    println!("Child updated");
                    cx.notify();
                }
            }
        });

        Self {
            child,
            _observe_sub: observe_sub,
            _subscribe_sub: subscribe_sub,
        }
    }
}
```

### 6.3.2 全局状态：`set_global()`、`global()`

```rust
use gpui::{Global, App};

struct AppState {
    user_name: String,
}

impl Global for AppState {}

fn init(cx: &mut App) {
    cx.set_global(AppState {
        user_name: "Alice".to_string(),
    });
}

fn greet(cx: &App) {
    let state = cx.global::<AppState>();
    println!("Hello, {}!", state.user_name);
}

fn rename(cx: &mut App) {
    cx.update_global(|state: &mut AppState, cx| {
        state.user_name = "Bob".to_string();
    });
}

// 安全访问（类型未设置时不 panic）
fn try_greet(cx: &App) {
    if let Some(state) = cx.try_global::<AppState>() {
        println!("Hello, {}!", state.user_name);
    } else {
        println!("No user logged in");
    }
}
```

### 6.3.3 异步操作：`spawn()`、`background_spawn()`

```rust
struct DataLoader {
    result: Option<String>,
    loading: bool,
}

impl DataLoader {
    fn start_loading(&mut self, cx: &mut Context<Self>) {
        self.loading = true;
        cx.notify();

        // spawn: 前台执行器，在主线程运行，可以访问 WeakEntity
        cx.spawn(|this, mut cx| async move {
            let data = async_fetch("https://api.example.com/data").await;

            // 升级弱引用，如果 Entity 已被销毁则跳过
            this.update(&mut cx, |this, cx| {
                this.result = Some(data);
                this.loading = false;
                cx.notify();
            }).ok();
        }).detach();
    }

    fn start_background_task(&mut self, cx: &mut Context<Self>) {
        // background_spawn: 后台线程池，需要 Send + Sync
        cx.background_spawn(async {
            // 在后台线程执行，不能访问 GPUI 状态
            let computed = heavy_computation();
            computed
        }).detach();
    }
}
```

### 6.3.4 事件操作：`emit()`、`notify()`

```rust
use gpui::{EventEmitter, Context, Entity};

// 定义事件类型
enum ChildEvent {
    CountChanged(i32),
    Reset,
}

struct Child {
    count: i32,
}

// 实现 EventEmitter trait
impl EventEmitter<ChildEvent> for Child {}

impl Child {
    fn increment(&mut self, cx: &mut Context<Self>) {
        self.count += 1;
        cx.notify();           // 通知观察者（无数据）
        cx.emit(ChildEvent::CountChanged(self.count)); // 发射事件（带数据）
    }

    fn reset(&mut self, cx: &mut Context<Self>) {
        self.count = 0;
        cx.emit(ChildEvent::Reset);
    }
}
```

`notify` vs `emit` 的区别：

| | `cx.notify()` | `cx.emit(Event)` |
|---|---|---|
| 接收者 | `observe` 注册的观察者 | `subscribe` 注册的订阅者 |
| 携带数据 | 否 | 是 |
| 需要 trait | 不需要 | 需要 `EventEmitter<E>` |
| 用途 | "我变了，请刷新" | "发生了某件事，请处理" |

## 6.4 ViewContext 与 WindowContext

### 6.4.1 ViewContext：视图渲染时的上下文

`ViewContext<V>` 在 `Render::render` 期间可用，本质上是 `Context<V>` 加上 `Window` 引用：

```rust
// ViewContext 提供了对 Window 的访问
struct ViewContext<'a, V> {
    cx: &'a mut Context<'a, V>,
    window: &'a mut Window,
}

impl<V> std::ops::Deref for ViewContext<'_, V> {
    type Target = Context<'_, V>;
    fn deref(&self) -> &Self::Target {
        self.cx
    }
}

impl<V> std::ops::DerefMut for ViewContext<'_, V> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        self.cx
    }
}
```

```rust
impl Render for MyView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 这里可以使用 window 进行焦点操作等
        let focused = self.focus_handle.is_focused(window);

        div()
            .child(format!("Focused: {}", focused))
    }
}
```

### 6.4.2 WindowContext：窗口事件处理时的上下文

窗口事件处理中接收到的是 `Window` + `App` 分离的形式：

```rust
// 典型的事件回调签名
.on_click(|event: &MouseDownEvent, window: &mut Window, cx: &mut App| {
    // window 提供窗口级操作
    // cx 提供应用级操作
})
```

通过 `WindowContext`（`Window` + `&mut App` 的组合），可以执行需要两者配合的操作：

```rust
// 使用 listener 模式获得完整的上下文
button.on_click(cx.listener(|this, event, window, cx| {
    // 修改自身状态
    this.selected = true;

    // 焦点操作需要 window
    this.focus_handle.focus(window, cx);

    // 通知重新渲染
    cx.notify();
}));
```

### 6.4.3 两者的转换与限制

```
ViewContext<V>
  |-- Deref 到 Context<V>
  |-- 可以访问 Window
  +-- 只能在 render 期间存在

WindowContext (概念上的)
  |-- 在窗口事件回调中存在
  |-- 可以访问 Window
  +-- 没有特定 Entity 的 Context
```

关键限制：
- `ViewContext` 只在 `render()` 执行期间存在
- 不能将 `ViewContext` 的引用存储到 struct 中
- 不能将 `ViewContext` 发送到其他线程

## 6.5 上下文所有权模式

### 6.5.1 闭包捕获规则

在 GPUI 回调中，闭包的捕获规则决定了生命周期：

```rust
struct MyView {
    counter: Entity<Counter>,
    selected: bool,
}

impl MyView {
    fn render_item(&mut self, cx: &mut Context<Self>) -> impl IntoElement {
        // 错误：不能捕获 &mut cx 到 'static 闭包
        // button.on_click(move |event, window, cx| {
        //     cx.update_entity(&self.counter, ...); // 错误
        // })

        // 正确：用 clone 移动 Entity 句柄
        let counter = self.counter.clone();
        button()
            .label("Click")
            .on_click(move |event, window, cx| {
                counter.update(cx, |counter, cx| {
                    counter.value += 1;
                    cx.notify();
                });
            })
    }
}
```

### 6.5.2 闭包中的上下文借用

`cx.listener()` 是处理闭包上下文的最常用方式：

```rust
impl MyView {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            counter: cx.new(|_cx| Counter { value: 0 }),
        }
    }
}

// listener 将 Context<Self> 的能力包装到 'static 闭包中
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(
                button()
                    .label("Increment")
                    .on_click(cx.listener(|this, _, window, cx| {
                        this.counter.update(cx, |counter, cx| {
                            counter.value += 1;
                            cx.notify();
                        });
                    })),
            )
            .child(
                button()
                    .label("Reset")
                    .on_click(cx.listener(|this, _, window, cx| {
                        this.counter.update(cx, |counter, cx| {
                            counter.value = 0;
                            cx.notify();
                        });
                    })),
            )
    }
}
```

### 6.5.3 常见的编译错误与解决方案

**错误 1：闭包生命周期不够**

```rust
// 错误：闭包需要 'static，但捕获了非 'static 的引用
fn bad(cx: &mut Context<Self>) -> impl FnMut() {
    let value = 42;
    move || {
        // value 不能跨越闭包边界
    }
}

// 解决：移动值而非借用
fn good(cx: &mut Context<Self>) -> impl FnMut() {
    let value = 42;
    move || {
        println!("{}", value); // value 被移动到闭包中
    }
}
```

**错误 2：在回调中使用了不正确的上下文类型**

```rust
// 错误：在 App 级别回调中尝试使用 Context<T> 专有方法
cx.open_window(options, |window, cx| {
    // cx 这里是 &mut App，不是 Context<T>
    // 不能调用 cx.notify() — 这只有在 Entity 的上下文中才可用
    // cx.notify(); // 编译错误
});
```

**错误 3：双重可变借用**

```rust
// 错误：同时持有两个可变借用
counter.update(cx, |counter, cx| {
    // 此时 cx 的 app 部分已被借用
    // 如果在这里再 update 另一个引用了同一个 App 的 Entity
    // 可能触发 RefCell 的双重借用
    other.update(cx, |other, cx| {}); // 可能 panic
});
```
