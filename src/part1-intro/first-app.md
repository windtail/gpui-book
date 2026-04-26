# 第 3 章：第一个应用

## 3.1 最小可运行示例

### 3.1.1 Application::new() + App::run()

一个 GPUI 应用从 `Application::new()` 开始：

```rust
use gpui::*;

fn main() {
    let app = Application::new();
    app.run(|cx| {
        // 应用开始运行后执行
        // cx 是 &mut App，你可以用它创建窗口、注册菜单等
    });
}
```

`Application::new()` 做了三件事：

1. 检测并初始化平台后端（macOS 用 Metal，Linux 用 wgpu，Windows 用 DirectX 11）
2. 创建 `AppCell`——一个包含 `RefCell<App>` 的全局存储
3. 设置前台和后台异步执行器

`app.run(callback)` 启动事件循环。回调在平台层准备好之后执行一次——它不是每帧调用的，它只调用一次。

> **注意**：`run` 是阻塞的。它会一直运行，直到用户关闭窗口或调用 `cx.quit()`。

### 3.1.2 创建窗口与设置标题

光有应用没有窗口，用户看不到任何东西。使用 `open_window()` 创建窗口：

```rust
use gpui::*;

fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.open_window(WindowOptions::default(), |window, cx| {
            cx.new(|cx| HelloWorld)
        });
    });
}

struct HelloWorld;

impl Render for HelloWorld {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .child("Hello, GPUI!")
    }
}
```

`open_window` 的签名：

```rust
pub fn open_window<V: 'static + Render>(
    &mut self,
    options: WindowOptions,
    build_root_view: impl FnOnce(&mut Window, &mut App) -> Entity<V>,
) -> anyhow::Result<WindowHandle<V>>
```

它接收两个参数：

- `WindowOptions`：窗口的配置（大小、标题、装饰等）
- `build_root_view`：闭包，返回一个 `Entity<V>`，其中 `V` 必须实现 `Render`

窗口会显示这个 View 渲染的内容。

### 3.1.3 完整的 "Hello, GPUI!"

把上面的代码整合成一个完整的、可编译运行的程序：

```rust
use gpui::*;

fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(800.), px(600.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("My GPUI App")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| cx.new(|cx| HelloWorld),
        );
    });
}

struct HelloWorld;

impl Render for HelloWorld {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .flex()
            .items_center()
            .justify_center()
            .bg(rgb(0x1e1e1e))
            .child(
                div()
                    .text_size(px(48.))
                    .text_color(rgb(0xffffff))
                    .font_family("Helvetica")
                    .child("Hello, GPUI!"),
            )
    }
}
```

运行：

```bash
cargo run
```

你会看到一个 800x600 的深色窗口，中间显示白色大字 "Hello, GPUI!"。

## 3.2 窗口配置

### 3.2.1 WindowBounds 与尺寸设置

`WindowBounds` 控制窗口的初始状态：

```rust
// 固定大小的窗口
let options = WindowOptions {
    window_bounds: Some(WindowBounds::Windowed(
        Bounds::new(
            Point::new(px(50.), px(50.)),      // 位置（屏幕坐标）
            size(px(1024.), px(768.)),          // 大小
        ),
    )),
    ..Default::default()
};

// 最大化打开，但记住还原后的大小
let options = WindowOptions {
    window_bounds: Some(WindowBounds::Maximized(
        Bounds::new(Point::zero(), size(px(1024.), px(768.))),
    )),
    ..Default::default()
};

// 全屏
let options = WindowOptions {
    window_bounds: Some(WindowBounds::Fullscreen(
        Bounds::new(Point::zero(), size(px(1920.), px(1080.))),
    )),
    ..Default::default()
};
```

`Pixels` 是逻辑像素单位，独立于屏幕的物理像素密度（DPI）。GPUI 会自动处理缩放。

```rust
// 快捷函数
let size = size(px(800.), px(600.));  // Size<Pixels>
let point = point(px(0.), px(0.));     // Point<Pixels>
let bounds = Bounds::new(point, size); // Bounds<Pixels>
```

### 3.2.2 窗口选项：title_bar、window_kind

**TitlebarOptions** 配置标题栏：

```rust
let options = WindowOptions {
    titlebar: Some(TitlebarOptions {
        title: Some(SharedString::from("My App")),
        appears_transparent: true,  // macOS：隐藏原生标题栏
        traffic_light_position: Some(point(px(20.), px(20.))),  // macOS 红绿灯位置
    }),
    ..Default::default()
};
```

**WindowKind** 设置窗口类型：

```rust
let options = WindowOptions {
    kind: WindowKind::Normal,   // 普通窗口
    // kind: WindowKind::PopUp,    // 弹出窗口（始终置顶）
    // kind: WindowKind::Floating, // 浮动窗口
    ..Default::default()
};
```

完整的 WindowOptions 常用字段：

```rust
WindowOptions {
    window_bounds: Some(...),       // 窗口大小和状态
    titlebar: Some(...),            // 标题栏配置
    focus: true,                    // 创建时是否获取焦点
    show: true,                     // 创建时是否显示
    kind: WindowKind::Normal,       // 窗口类型
    ..Default::default()
}
```

### 3.2.3 全屏与无边框模式

**全屏**：

```rust
fn enter_fullscreen(cx: &mut App, handle: WindowHandle<HelloWorld>) {
    if let Err(e) = cx.update_window(handle, |_, window, cx| {
        window.set_fullscreen(true, cx);
    }) {
        eprintln!("Failed to toggle fullscreen: {e}");
    }
}
```

**无边框**（隐藏系统装饰）：

```rust
let options = WindowOptions {
    window_decorations: Some(WindowDecorations::Server),  // 无装饰
    ..Default::default()
};
```

无边框模式下，你需要自己绘制标题栏并处理拖拽逻辑。这在 Zed 编辑器中就是如此——它隐藏了原生标题栏，用自定义标题栏替代。

## 3.3 视图与 Render

### 3.3.1 定义你的第一个 View struct

View 就是一个实现了 `Render` trait 的 struct：

```rust
struct Counter {
    count: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .flex()
            .flex_col()
            .items_center()
            .justify_center()
            .gap_4()
            .child(
                div()
                    .text_size(px(64.))
                    .child(format!("{}", self.count)),
            )
            .child(
                div()
                    .flex()
                    .gap_2()
                    .child(
                        button("decrement", "－")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count -= 1;
                                cx.notify();
                            })),
                    )
                    .child(
                        button("increment", "＋")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count += 1;
                                cx.notify();
                            })),
                    ),
            )
    }
}
```

创建 View 使用 `cx.new()`：

```rust
let counter: Entity<Counter> = cx.new(|cx| Counter { count: 0 });
```

`cx.new()` 做了以下事情：

1. 在 App 的 EntityMap 中分配一个 slot
2. 调用闭包创建 struct 实例
3. 返回 `Entity<T>` 句柄

### 3.3.2 实现 Render trait

`Render` trait 的定义：

```rust
pub trait Render: 'static + Sized {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement;
}
```

关键点：

- **`&mut self`**：你可以访问并修改自己的状态。这与立即模式框架不同——在 egui 中你不持有状态，状态在调用者那里。
- **`Window`**：提供窗口级操作（焦点、尺寸、外观等）
- **`Context<Self>`**：提供 Entity 级操作（创建子 Entity、订阅、触发重新渲染）
- **返回值 `impl IntoElement`**：返回元素树的根

`cx.listener` 创建一个事件处理器，它自动将 `&mut App` 转换为 `&mut Context<Self>`：

```rust
// cx.listener 的签名等价于：
fn listener(
    this: &mut Self,           // View 实例的可变引用
    event: &ClickEvent,        // 事件
    window: &mut Window,       // 窗口
    cx: &mut Context<Self>,    // 上下文
) {
    // ...
}
```

### 3.3.3 用 div() 构建 UI

`div()` 是 GPUI 的主要容器元素。它支持：

```rust
div()
    // 布局
    .flex()                          // 激活 flexbox
    .flex_row()                      // 水平排列（默认）
    .flex_col()                      // 垂直排列
    .items_center()                  // 交叉轴居中
    .justify_center()                // 主轴居中
    .gap_2()                         // 子元素间距
    .flex_1()                        // 填充剩余空间

    // 尺寸
    .size_full()                     // width: 100%, height: 100%
    .w(px(200.))                     // 固定宽度
    .h(px(100.))                     // 固定高度
    .min_h(px(50.))                  // 最小高度

    // 样式
    .bg(rgb(0x1e1e1e))              // 背景色
    .rounded_lg()                    // 圆角
    .border_1()                      // 1px 边框
    .border_color(rgb(0x333333))     // 边框颜色
    .shadow_sm()                     // 阴影
    .p_4()                           // 内边距

    // 交互
    .on_click(|_, _, cx| {})         // 点击事件
    .on_hover(|_, _, cx| {})         // 悬停事件
    .cursor_pointer()                // 鼠标指针

    // 子元素
    .child(div())                    // 单个子元素
    .children(vec![div(), div()])    // 多个子元素
    .when(condition, |div| div.child(extra))  // 条件子元素
```

文本可以直接作为子元素——字符串实现了 `IntoElement`：

```rust
div()
    .child("直接写字符串")              // 自动转为文本元素
    .child(format!("Count: {}", 42))    // 格式化字符串也可以
```

### 3.3.4 完整的计数器应用

```rust
use gpui::*;

fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.open_window(WindowOptions {
            window_bounds: Some(WindowBounds::Windowed(
                Bounds::new(Point::new(px(100.), px(100.)), size(px(400.), px(300.))),
            )),
            titlebar: Some(TitlebarOptions {
                title: Some(SharedString::from("Counter")),
                ..Default::default()
            }),
            ..Default::default()
        }, |window, cx| {
            cx.new(|cx| Counter { count: 0 })
        });
    });
}

struct Counter {
    count: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let count_color = if self.count < 0 {
            rgb(0xff6b6b)  // 红色
        } else if self.count > 0 {
            rgb(0x51cf66)  // 绿色
        } else {
            rgb(0xffffff)  // 白色
        };

        div()
            .size_full()
            .flex()
            .flex_col()
            .items_center()
            .justify_center()
            .gap_6()
            .bg(rgb(0x2d2d2d))
            .child(
                div()
                    .text_size(px(72.))
                    .text_color(count_color)
                    .font_family("SF Mono")
                    .child(format!("{}", self.count)),
            )
            .child(
                div()
                    .flex()
                    .gap_3()
                    .child(
                        button("decrement", "－")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count -= 1;
                                cx.notify();
                            })),
                    )
                    .child(
                        button("increment", "＋")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count += 1;
                                cx.notify();
                            })),
                    ),
            )
    }
}
```

## 3.4 编译、运行、调试

### 3.4.1 cargo run 与常见编译错误

```bash
cargo run
```

**常见编译错误：**

**1. 忘记实现 Render：**

```
error[E0277]: the trait bound `MyView: Render` is not satisfied
  --> src/main.rs:10:14
   |
10 |     cx.new(|cx| MyView);
   |                 ^^^^^^ the trait `Render` is not implemented for `MyView`
```

修复：添加 `impl Render for MyView`。

**2. 生命周期 `'static` 问题：**

```rust
// 错误：MyView 包含非 'static 引用
struct MyView<'a> {
    data: &'a str,  // 'a 不是 'static
}

impl Render for MyView<'_> { ... }  // 错误：Render 需要 'static
```

修复：使用 `String` 或 `SharedString`：

```rust
struct MyView {
    data: SharedString,
}
```

**3. 忘记导入 prelude trait：**

```rust
// 错误：方法 `child` 不存在
div().child("text")
//   ^^^^^ method not found
```

修复：添加 `use gpui::prelude::*;`。

### 3.4.2 热重载的可能性

GPUI 本身不提供热重载，但有几种方式可以在开发时实现类似效果：

**1. 手动刷新：**

在代码中绑定快捷键触发刷新：

```rust
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .on_key_down(cx.listener(|_, event: &KeyDownEvent, _, cx| {
                // Cmd+R 刷新
                if event.keystroke.key == "r" && event.keystroke.modifiers.command {
                    cx.refresh();
                }
            }))
            .child("Press Cmd+R to refresh")
    }
}
```

**2. 文件监听 + 自动重启：**

```bash
cargo install cargo-watch
cargo watch -x run
```

这会在文件变化时自动重新编译并运行。对于小型应用效果很好。

**3. 动态库加载（高级）：**

将 UI 逻辑编译为动态库，主程序在运行时重新加载 `.so`/`.dylib` 文件。这是 Zed 开发过程中使用的方法之一，但设置较复杂。

### 3.4.3 本章小结与下一步

你现在已经可以：

1. 创建并运行一个 GPUI 应用
2. 配置窗口大小和标题
3. 定义 View 并实现 Render
4. 使用 `div()` 构建 UI 布局
5. 处理点击事件和更新状态

这些是 GPUI 开发的核心概念。接下来的章节会深入讲解：

- **Application 与应用生命周期**：`App` 的内部机制、事件循环、多窗口
- **Entity 系统**：`Entity<T>` 的读写、弱引用、借用检查
- **Context 类型**：`Context<T>`、`WindowContext`、`ViewContext` 的区别和用法

你的第一个应用已经跑起来了。下一章我们深入理解 GPUI 的核心架构。
