# 第4章：Application 与应用生命周期

## 4.1 `App` 是唯一的真理

GPUI 应用的状态全部保存在 `App` struct 中。你无法直接构造 `App`，而是通过 `Application::new()` 获取一个 `Application` 句柄，它内部持有 `Rc<AppCell>`，而 `AppCell` 包装了 `RefCell<App>`。

```rust
use gpui::{Application, App};

fn main() {
    let app = Application::new();
    app.run(|cx: &mut App| {
        // cx 是 &mut App，应用状态的唯一入口
    });
}
```

`App` 拥有：
- 所有 Entity 的存储（`EntityMap`）
- 所有窗口（`SlotMap<WindowId, Option<Box<Window>>>`）
- 平台后端句柄（`platform: Rc<dyn Platform>`）
- 全局状态（`FxHashMap<TypeId, Box<dyn Any>>`）
- 前后台执行器
- 事件分发系统

### 4.1.2 `AppCell` 全局存储

```rust
// 来自 GPUI 源码
pub struct AppCell {
    app: RefCell<App>,
}

impl AppCell {
    pub fn borrow(&self) -> AppRef<'_> {
        AppRef(self.app.borrow())
    }

    pub fn borrow_mut(&self) -> AppRefMut<'_> {
        AppRefMut(self.app.borrow_mut())
    }
}
```

`Application` 包装了 `Rc<AppCell>`，通过 `RefCell` 实现运行时借用检查。同一时刻只能有一个可变借用或多个不可变借用。

### 4.1.3 为什么不能直接持有可变引用

GPUI 采用运行时借用检查而非编译期借用检查。原因在于回调链嵌套深度不可预测——你在 `Entity::update()` 内部可能再调用另一个 `Entity::update()`，编译期无法验证这种动态嵌套。

```rust
// 错误示例：嵌套 update 同一个 Entity 导致 panic
counter.update(cx, |counter, cx| {
    // 此时 counter 的数据已被移出到栈上（lease 模式）
    // 如果再尝试 update 同一个 counter，会 panic
    // "cannot update Counter while it is already being updated"
});
```

## 4.2 应用生命周期

### 4.2.1 `Application::new()` 初始化

```rust
use gpui::{Application, App, WindowOptions};

fn main() {
    let app = Application::new();

    app.run(|cx: &mut App| {
        cx.open_window(WindowOptions::default(), |window, cx| {
            // 创建根视图
            cx.new(|cx| MyApp { count: 0 })
        });
    });
}
```

`Application::new()` 在内部调用 `App::new_app()`，初始化平台后端、文本系统、执行器。

### 4.2.2 事件循环内部

`app.run()` 将控制权交给平台的事件循环。核心流程：

1. 平台派发事件（鼠标、键盘、窗口事件）
2. GPUI 执行对应回调
3. 回调中修改 Entity 状态、调用 `cx.notify()`
4. 当前帧回调结束后，GPUI 收集所有 dirty 视图
5. 重新渲染 dirty 视图，提交到 GPU
6. 等待下一个事件

```rust
use gpui::{
    div, InteractiveElement, ParentElement, IntoElement, Context, Window,
    px, SharedString,
};

fn button(id: impl Into<ElementId>, text: impl Into<SharedString>) -> impl IntoElement {
    div()
        .id(id)
        .px_3()
        .py_1()
        .rounded_md()
        .bg(rgb(0x3b82f6))
        .text_white()
        .cursor_pointer()
        .child(text.into())
}

struct Counter {
    count: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(format!("Count: {}", self.count))
            .child(
                button("increment", "Increment")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.count += 1;
                        cx.notify(); // 触发重新渲染
                    })),
            )
    }
}
```

### 4.2.3 退出与清理

```rust
use gpui::{Application, App, QuitMode};

fn main() {
    let app = Application::new();

    // 注册重新打开回调（macOS dock 点击）
    app.on_reopen(|cx| {
        println!("App reopened");
    });

    app.run(|cx: &mut App| {
        // 设置自动退出模式
        cx.set_quit_mode(QuitMode::LastWindowClosed);

        // 注册退出时的异步清理
        cx.on_quit(|cx| {
            async move {
                // 异步清理逻辑，最多运行 SHUTDOWN_TIMEOUT (100ms)
            }.boxed_local()
        });
    });
}
```

`QuitMode` 有三种：

| 模式 | 行为 |
|------|------|
| `QuitMode::Default` | macOS 用 `Explicit`，其他平台用 `LastWindowClosed` |
| `QuitMode::LastWindowClosed` | 最后一个窗口关闭时自动退出 |
| `QuitMode::Explicit` | 只在显式调用 `cx.quit()` 时退出 |

编程式退出：

```rust
fn close_app(cx: &mut App) {
    cx.quit();
}
```

## 4.3 `AppContext` trait

### 4.3.1 trait 定义与核心方法

```rust
pub trait AppContext {
    fn new<T: 'static>(&mut self, build: impl FnOnce(&mut Context<T>) -> T) -> Entity<T>;
    fn reserve_entity<T: 'static>(&mut self) -> Reservation<T>;
    fn insert_entity<T: 'static>(
        &mut self,
        reservation: Reservation<T>,
        build: impl FnOnce(&mut Context<T>) -> T,
    ) -> Entity<T>;
    fn update_entity<T, R>(
        &mut self,
        handle: &Entity<T>,
        update: impl FnOnce(&mut T, &mut Context<T>) -> R,
    ) -> R;
    fn read_entity<T, R>(
        &self,
        handle: &Entity<T>,
        read: impl FnOnce(&T, &App) -> R,
    ) -> R;
    fn update_window<T, F>(
        &mut self,
        window: AnyWindowHandle,
        f: F,
    ) -> Result<T>;
    fn read_window<T, R>(
        &self,
        window: &WindowHandle<T>,
        read: impl FnOnce(Entity<T>, &App) -> R,
    ) -> Result<R>;
    fn background_spawn<R>(
        &self,
        future: impl Future<Output = R> + Send + 'static,
    ) -> Task<R>;
    fn read_global<G, R>(
        &self,
        callback: impl FnOnce(&G, &App) -> R,
    ) -> R;
}
```

`AppContext` 是 GPUI 中所有上下文的统一接口。`App`、`Context<T>` 都实现了这个 trait。

### 4.3.2 `App` 与上下文的关系

```
AppContext (trait)
  |-- App (实现 AppContext)
  |-- Context<T> (实现 AppContext, Deref<Target=App>)
  +-- AsyncApp (实现异步场景的 AppContext)
```

```rust
// Context<T> 将 AppContext 委托给内部的 App
impl<T> AppContext for Context<'_, T> {
    fn new<U: 'static>(&mut self, build: impl FnOnce(&mut Context<U>) -> U) -> Entity<U> {
        self.app.new(build)
    }

    fn update_entity<U, R>(
        &mut self,
        handle: &Entity<U>,
        update: impl FnOnce(&mut U, &mut Context<U>) -> R,
    ) -> R {
        self.app.update_entity(handle, update)
    }
}
```

### 4.3.3 在回调中使用上下文

不同回调接收不同类型的上下文：

```rust
// Render 回调：Context<Self>
impl Render for MyView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // cx 是 Context<Self>，可以创建子 Entity
        let child = cx.new(|cx| ChildView { data: self.data.clone() });
        div()
    }
}

// 事件回调：Window + App
button.on_click(|event, window: &mut Window, cx: &mut App| {
    // cx 是 &mut App，只能做应用级操作
    // 不能直接修改视图状态
});

// 通过 listener 模式将 Context<Self> 带入事件回调
button.on_click(cx.listener(|this, event, window, cx| {
    // cx 是 Context<Self>，可以修改 this
    this.count += 1;
    cx.notify();
}));
```

## 4.4 应用级服务

### 4.4.1 剪贴板操作

```rust
use gpui::{ClipboardItem, App};

fn copy_text(cx: &mut App) {
    let item = ClipboardItem::new_string("Hello from GPUI".to_string());
    cx.write_to_clipboard(item);
}

fn paste_text(cx: &mut App) {
    if let Some(item) = cx.read_from_clipboard() {
        let text = item.text();
        println!("Pasted: {}", text);
    }
}
```

### 4.4.2 文件对话框

```rust
use gpui::{App, PathPromptOptions};

fn open_file_dialog(cx: &mut App) {
    let options = PathPromptOptions {
        files: true,
        directories: false,
        multiple: true,
    };

    // 返回 Task<Option<Vec<std::path::PathBuf>>>
    let task = cx.prompt_for_paths(options, None, None);
    // 需要在前台执行器中等待结果
}
```

### 4.4.3 打开 URL

```rust
use gpui::App;

fn open_docs(cx: &mut App) {
    cx.open_url("https://zed.dev/docs");
}
```

### 4.4.4 应用菜单（macOS）

```rust
use gpui::{Application, App, Menu, MenuItem};

fn main() {
    let app = Application::new();

    let menu = vec![
        Menu {
            name: "MyApp".into(),
            items: vec![
                MenuItem::separator(),
                MenuItem::os_app("About MyApp".into()),
                MenuItem::separator(),
                MenuItem::quit(),
            ],
        },
    ];

    app.set_menus(menu);
    app.activate();

    app.run(|cx: &mut App| {});
}
```

## 4.5 多窗口管理

### 4.5.1 `open_window()` 创建新窗口

```rust
use gpui::{
    Application, App, Window, WindowOptions, WindowBounds,
    Size, Pixels, Entity, Context, IntoElement,
};

struct Document {
    content: String,
}

impl Render for Document {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div().child(&self.content)
    }
}

fn open_new_window(cx: &mut App) {
    let options = WindowOptions {
        window_bounds: Some(WindowBounds::Windowed(
            gpui::Bounds::new(
                gpui::Point::new(Pixels(100.0), Pixels(100.0)),
                Size::new(Pixels(800.0), Pixels(600.0)),
            ),
        )),
        titlebar: Some(gpui::Titlebar {
            title: "New Document",
            ..Default::default()
        }),
        ..Default::default()
    };

    cx.open_window(options, |window, cx| {
        cx.new(|cx| Document {
            content: "New document".to_string(),
        })
    });
}
```

### 4.5.2 窗口间的通信

通过 `WindowHandle` 跨窗口通信：

```rust
struct MainWindow {
    documents: Vec<Entity<Document>>,
}

impl MainWindow {
    fn open_document(&mut self, cx: &mut Context<Self>) {
        let handle = cx.open_window(WindowOptions::default(), |window, cx| {
            cx.new(|cx| Document {
                content: "Shared data".to_string(),
            })
        });

        // 通过窗口句柄更新窗口内容
        // handle.update(cx, |view, window, cx| { ... })
    }
}
```

通过共享 Entity 实现窗口间数据同步：

```rust
struct SharedState {
    value: i32,
}

struct WindowA {
    shared: Entity<SharedState>,
}

struct WindowB {
    shared: Entity<SharedState>,
}

// 两个窗口持有同一个 Entity，任意一方修改都会通知另一方
fn create_windows(cx: &mut App) {
    let shared = cx.new(|_cx| SharedState { value: 0 });

    cx.open_window(WindowOptions::default(), |_window, cx| {
        cx.new(|cx| WindowA {
            shared: shared.clone(),
        })
    });

    cx.open_window(WindowOptions::default(), |_window, cx| {
        cx.new(|cx| WindowB {
            shared: shared.clone(),
        })
    });
}
```

### 4.5.3 窗口事件监听

```rust
use gpui::{Window, App, Context};

struct MyView {
    // ...
}

impl MyView {
    fn init(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        // 监听窗口尺寸变化
        cx.observe_window_bounds(window, |this, window, cx| {
            let bounds = window.window_bounds();
            println!("Window resized to: {:?}", bounds);
            cx.notify();
        }).detach();

        // 监听窗口激活/失活
        cx.observe_window_activation(window, |this, window, cx| {
            let active = window.is_window_active();
            println!("Window active: {}", active);
        }).detach();

        // 监听窗口外观变化（暗色/亮色模式切换）
        cx.observe_window_appearance(window, |this, window, cx| {
            let appearance = window.window_appearance();
            println!("Appearance: {:?}", appearance);
            cx.notify();
        }).detach();
    }
}
```
