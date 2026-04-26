# 第 64 章：Hello World

本章从零开始构建最简 GPUI 应用——一个窗口，居中显示一行文字。

## Cargo.toml

```toml
[package]
name = "hello-gpui"
version = "0.1.0"
edition = "2024"

[dependencies]
gpui = "0.2.2"
```

## main.rs 入口

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
            .flex()
            .items_center()
            .justify_center()
            .bg(rgb(0x1e1e1e))
            .child(
                div()
                    .text_size(px(48.))
                    .text_color(rgb(0xffffff))
                    .child("Hello, GPUI!"),
            )
    }
}
```

运行：

```bash
cargo run
```

你会看到一个深色窗口，白色文字 "Hello, GPUI!" 居中显示。

## 逐行解释

```rust
let app = Application::new();
```

`Application::new()` 完成三件事：初始化平台后端（macOS 用 Metal，Linux 用 wgpu）、创建 `AppCell` 全局存储、设置异步执行器。

```rust
app.run(|cx| { ... });
```

`run` 消耗 `self` 并启动事件循环。闭包只调用一次，在平台层准备好之后。`cx` 是 `&mut App`，你可以用它创建窗口、注册菜单、设置全局状态。

```rust
cx.open_window(WindowOptions::default(), |window, cx| {
    cx.new(|cx| HelloWorld)
});
```

`open_window` 接收窗口配置和视图构建闭包。闭包返回 `Entity<V>`，其中 `V` 必须实现 `Render`。

```rust
struct HelloWorld;
```

View 是一个 unit struct。它不需要任何字段，因为这是静态显示。

```rust
impl Render for HelloWorld {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
```

`Render` trait 定义了视图如何将自己渲染为元素树。`&mut self` 允许你在 render 中访问和修改自身状态。

```rust
div()
    .size_full()
    .flex()
    .items_center()
    .justify_center()
    .bg(rgb(0x1e1e1e))
```

`div()` 是 GPUI 的主要容器。这些方法链设置了：
- `size_full()`：宽高 100%
- `flex()` + `items_center()` + `justify_center()`：flexbox 居中
- `bg(rgb(0x1e1e1e))`：深色背景

```rust
.child(
    div()
        .text_size(px(48.))
        .text_color(rgb(0xffffff))
        .child("Hello, GPUI!"),
)
```

子 div 设置字号 48px、白色文字。字符串 `"Hello, GPUI!"` 自动转换为文本元素。

## 带窗口配置的版本

添加窗口标题和尺寸：

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
                    title: Some(SharedString::from("Hello GPUI")),
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
                    .child("Hello, GPUI!"),
            )
    }
}
```

## 关键要点

- `Application::new()` + `app.run()` 是 GPUI 应用的固定入口模式
- `cx.open_window()` 创建窗口，闭包返回根 View 的 Entity
- `Render` trait 是视图的核心接口，`render()` 返回 `impl IntoElement`
- `div()` 是主要容器，通过方法链配置布局和样式
- `size_full()` 让元素填满父容器
- `flex()` + `items_center()` + `justify_center()` 是最常用的居中组合
- `rgb(0xRRGGBB)` 创建颜色，`px(n)` 创建像素单位
