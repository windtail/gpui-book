# 第 65 章：计数器应用

本章构建一个带加减按钮的计数器，展示 GPUI 的 Entity 状态管理和 `cx.notify()` 更新模式。

## Cargo.toml

```toml
[package]
name = "counter"
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
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(400.), px(300.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Counter")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| cx.new(|cx| Counter { count: 0 }),
        );
    });
}
```

## 定义 Counter View

```rust
struct Counter {
    count: i32,
}
```

Counter 只持有一个 `i32` 字段。当用户点击按钮时，这个值会被修改并触发重新渲染。

## 实现 Render 与按钮点击

```rust
impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
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
                    .text_color(if self.count < 0 {
                        rgb(0xff6b6b)
                    } else if self.count > 0 {
                        rgb(0x51cf66)
                    } else {
                        rgb(0xffffff)
                    })
                    .child(format!("{}", self.count)),
            )
            .child(
                div()
                    .flex()
                    .gap_3()
                    .child(
                        button("decrement", "-")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count -= 1;
                                cx.notify();
                            })),
                    )
                    .child(
                        button("increment", "+")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count += 1;
                                cx.notify();
                            })),
                    ),
            )
    }
}
```

## cx.listener 与 cx.notify()

`cx.listener` 创建一个事件处理器，它自动将上下文转换为 `&mut Context<Self>`：

```rust
cx.listener(|this, _, _, cx| {
    this.count += 1;  // 修改状态
    cx.notify();       // 通知视图重新渲染
})
```

四参数顺序：
1. `this`: `&mut Counter`——View 实例的可变引用
2. 第二个参数：事件对象（`ClickEvent`）
3. `_window`: `&mut Window`
4. `cx`: `&mut Context<Self>`——当前 View 的上下文

**`cx.notify()` 是关键**。没有它，状态虽然被修改了，但 GPUI 不会重新调用 `render()`，用户看不到变化。

## 添加重置按钮与悬停效果

在按钮组中添加第三个按钮，并给数字显示添加悬停高亮：

```rust
impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let count_color = if self.count < 0 {
            rgb(0xff6b6b)
        } else if self.count > 0 {
            rgb(0x51cf66)
        } else {
            rgb(0xffffff)
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
                    .font_weight(gpui::FontWeight::BOLD)
                    .child(format!("{}", self.count)),
            )
            .child(
                div()
                    .flex()
                    .gap_3()
                    .child(
                        button("decrement", "-")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count -= 1;
                                cx.notify();
                            })),
                    )
                    .child(
                        button("reset", "0")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count = 0;
                                cx.notify();
                            })),
                    )
                    .child(
                        button("increment", "+")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count += 1;
                                cx.notify();
                            })),
                    ),
            )
    }
}
```

## 完整代码

```rust
use gpui::*;

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

fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(400.), px(300.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Counter")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| cx.new(|cx| Counter { count: 0 }),
        );
    });
}

struct Counter {
    count: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let count_color = if self.count < 0 {
            rgb(0xff6b6b)
        } else if self.count > 0 {
            rgb(0x51cf66)
        } else {
            rgb(0xffffff)
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
                    .font_weight(gpui::FontWeight::BOLD)
                    .child(format!("{}", self.count)),
            )
            .child(
                div()
                    .flex()
                    .gap_3()
                    .child(
                        button("decrement", "-")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count -= 1;
                                cx.notify();
                            })),
                    )
                    .child(
                        button("reset", "0")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count = 0;
                                cx.notify();
                            })),
                    )
                    .child(
                        button("increment", "+")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.count += 1;
                                cx.notify();
                            })),
                    ),
            )
    }
}
```

## 关键要点

- View struct 持有可变状态（`count: i32`）
- `cx.new(|cx| Counter { count: 0 })` 创建 View 的 Entity
- `cx.listener(|this, _, _, cx| { ... })` 创建事件处理器，自动转换上下文类型
- `cx.notify()` 标记视图需要重新渲染——忘记调用它是最常见的新手错误
- `button(id, label)` 是 GPUI 内置按钮元素，`.on_click()` 绑定点击事件
- 颜色可以根据状态动态计算（负数红、零白、正数绿）
