# 第 66 章：Todo 应用

本章构建一个完整的 Todo 应用，展示虚拟列表、文本输入、键盘快捷键和状态操作。

## Cargo.toml

```toml
[package]
name = "todo-app"
version = "0.1.0"
edition = "2024"

[dependencies]
gpui = "0.2.2"
```

## 数据模型

```rust
#[derive(Clone)]
struct TodoItem {
    text: SharedString,
    completed: bool,
}
```

每个 Todo 项包含文本和完成状态。`SharedString` 是 GPUI 的引用计数字符串，比 `String` 更高效。

## View 结构

```rust
struct TodoApp {
    items: Vec<TodoItem>,
    input_text: SharedString,
    focus_handle: FocusHandle,
    list_state: ListState,
}
```

- `items`: 存储所有 Todo 项
- `input_text`: 当前输入框文本
- `focus_handle`: 输入框焦点句柄
- `list_state`: 虚拟列表状态

## 创建 TodoApp

```rust
impl TodoApp {
    fn new(cx: &mut Context<Self>) -> Self {
        let list_state = ListState::new(0, ListAlignment::Top, px(400.));
        Self {
            items: Vec::new(),
            input_text: SharedString::default(),
            focus_handle: cx.focus_handle(),
            list_state,
        }
    }

    fn add_item(&mut self, cx: &mut Context<Self>) {
        let text = self.input_text.trim();
        if text.is_empty() {
            return;
        }
        self.items.push(TodoItem {
            text: SharedString::from(text),
            completed: false,
        });
        self.input_text = SharedString::default();
        let index = self.items.len() - 1;
        self.list_state.splice(index..index, 1);
    }

    fn delete_item(&mut self, index: usize, cx: &mut Context<Self>) {
        self.items.remove(index);
        self.list_state.splice(index..index + 1, 0);
    }

    fn toggle_item(&mut self, index: usize, cx: &mut Context<Self>) {
        if let Some(item) = self.items.get_mut(index) {
            item.completed = !item.completed;
        }
        cx.notify();
    }
}
```

`ListState::splice` 通知列表数据变更。第一个参数是范围，第二个是新增数量。删除时范围不变，新增数量为 0。

## Render 实现

```rust
impl Render for TodoApp {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let input_focus = self.focus_handle.clone();
        let list_state = self.list_state.clone();
        let item_count = self.items.len();

        div()
            .size_full()
            .flex()
            .flex_col()
            .bg(rgb(0x1e1e1e))
            // 顶部输入区
            .child(
                div()
                    .flex()
                    .gap_2()
                    .p_3()
                    .border_b_1()
                    .border_color(rgb(0x333333))
                    .child(
                        div()
                            .id("input")
                            .flex_1()
                            .px_2()
                            .py_1()
                            .border_1()
                            .border_color(rgb(0x555555))
                            .rounded_md()
                            .text_color(rgb(0xffffff))
                            .text_size(px(16.))
                            .track_focus(&input_focus)
                            .on_key_down(cx.listener(Self::on_key_down))
                            .child(
                                div().child(&self.input_text)
                            ),
                    )
                    .child(
                        button("add", "Add")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.add_item(cx);
                            })),
                    ),
            )
            // 列表区域
            .child(
                div()
                    .flex_1()
                    .overflow_hidden()
                    .child(list(
                        list_state,
                        cx.listener(Self::render_item),
                    )),
            )
            // 底部统计
            .child(
                div()
                    .flex()
                    .justify_between()
                    .p_3()
                    .border_t_1()
                    .border_color(rgb(0x333333))
                    .child(format!("{} items", item_count))
                    .child({
                        let completed = self.items.iter().filter(|i| i.completed).count();
                        format!("{} done", completed)
                    }),
            )
    }

    fn render_item(
        &self,
        index: usize,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> AnyElement {
        // 渲染单个 Todo 项（见下文）
        unimplemented!()
    }

    fn on_key_down(&mut self, event: &KeyDownEvent, _window: &mut Window, cx: &mut Context<Self>) {
        if event.keystroke.key == "enter" && !event.keystroke.modifiers.shift {
            self.add_item(cx);
        }
    }
}
```

## 渲染列表项

```rust
impl Render for TodoApp {
    // ... 上面的 render 方法 ...

    fn render_item(
        &self,
        index: usize,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> AnyElement {
        let Some(item) = self.items.get(index) else {
            return div().into_any_element();
        };

        let text = item.text.clone();
        let completed = item.completed;

        div()
            .id(SharedString::from(format!("todo-{}", index)))
            .flex()
            .items_center()
            .gap_2()
            .px_3()
            .py_2()
            .border_b_1()
            .border_color(rgb(0x2a2a2a))
            .when(completed, |div| {
                div.opacity(0.5)
            })
            // 复选框
            .child(
                div()
                    .id(SharedString::from(format!("checkbox-{}", index)))
                    .w(px(20.))
                    .h(px(20.))
                    .border_1()
                    .border_color(rgb(0x666666))
                    .rounded_sm()
                    .when(completed, |div| {
                        div.bg(rgb(0x51cf66)).border_color(rgb(0x51cf66))
                    })
                    .on_click(cx.listener(move |this, _, _, cx| {
                        this.toggle_item(index, cx);
                    }))
                    .child(if completed {
                        div()
                            .text_size(px(14.))
                            .text_color(rgb(0xffffff))
                            .child("✓")
                    } else {
                        div()
                    }),
            )
            // 文本
            .child(
                div()
                    .flex_1()
                    .text_size(px(16.))
                    .text_color(rgb(0xffffff))
                    .when(completed, |div| {
                        div.text_color(rgb(0x888888))
                    })
                    .child(text),
            )
            // 删除按钮
            .child(
                div()
                    .id(SharedString::from(format!("delete-{}", index)))
                    .px_2()
                    .py_1()
                    .text_size(px(12.))
                    .text_color(rgb(0xff6b6b))
                    .cursor_pointer()
                    .on_click(cx.listener(move |this, _, _, cx| {
                        this.delete_item(index, cx);
                    }))
                    .child("del"),
            )
            .into_any_element()
    }
}
```

## 键盘快捷键处理

```rust
impl TodoApp {
    fn on_key_down(&mut self, event: &KeyDownEvent, _window: &mut Window, cx: &mut Context<Self>) {
        // Enter 添加新项
        if event.keystroke.key == "enter" && !event.keystroke.modifiers.shift {
            self.add_item(cx);
        }
        // Escape 清空输入
        if event.keystroke.key == "escape" {
            self.input_text = SharedString::default();
            cx.notify();
        }
    }
}
```

## 主函数

```rust
fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(500.), px(600.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Todo")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| cx.new(|cx| TodoApp::new(cx)),
        );
    });
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

#[derive(Clone)]
struct TodoItem {
    text: SharedString,
    completed: bool,
}

struct TodoApp {
    items: Vec<TodoItem>,
    input_text: SharedString,
    focus_handle: FocusHandle,
    list_state: ListState,
}

impl TodoApp {
    fn new(cx: &mut Context<Self>) -> Self {
        let list_state = ListState::new(0, ListAlignment::Top, px(400.));
        Self {
            items: Vec::new(),
            input_text: SharedString::default(),
            focus_handle: cx.focus_handle(),
            list_state,
        }
    }

    fn add_item(&mut self, cx: &mut Context<Self>) {
        let text = self.input_text.trim();
        if text.is_empty() {
            return;
        }
        self.items.push(TodoItem {
            text: SharedString::from(text),
            completed: false,
        });
        let index = self.items.len() - 1;
        self.list_state.splice(index..index, 1);
        self.input_text = SharedString::default();
    }

    fn delete_item(&mut self, index: usize, cx: &mut Context<Self>) {
        self.items.remove(index);
        self.list_state.splice(index..index + 1, 0);
        cx.notify();
    }

    fn toggle_item(&mut self, index: usize, cx: &mut Context<Self>) {
        if let Some(item) = self.items.get_mut(index) {
            item.completed = !item.completed;
        }
        cx.notify();
    }

    fn on_key_down(&mut self, event: &KeyDownEvent, _window: &mut Window, cx: &mut Context<Self>) {
        if event.keystroke.key == "enter" && !event.keystroke.modifiers.shift {
            self.add_item(cx);
        }
        if event.keystroke.key == "escape" {
            self.input_text = SharedString::default();
            cx.notify();
        }
    }
}

impl Render for TodoApp {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let input_focus = self.focus_handle.clone();
        let list_state = self.list_state.clone();
        let item_count = self.items.len();
        let completed_count = self.items.iter().filter(|i| i.completed).count();

        div()
            .size_full()
            .flex()
            .flex_col()
            .bg(rgb(0x1e1e1e))
            .child(
                div()
                    .flex()
                    .gap_2()
                    .p_3()
                    .border_b_1()
                    .border_color(rgb(0x333333))
                    .child(
                        div()
                            .id("input-area")
                            .flex_1()
                            .px_2()
                            .py_1()
                            .border_1()
                            .border_color(rgb(0x555555))
                            .rounded_md()
                            .text_color(rgb(0xffffff))
                            .text_size(px(16.))
                            .track_focus(&input_focus)
                            .on_key_down(cx.listener(Self::on_key_down))
                            .child(div().child(&self.input_text)),
                    )
                    .child(
                        button("add-btn", "Add")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.add_item(cx);
                            })),
                    ),
            )
            .child(
                div()
                    .flex_1()
                    .overflow_hidden()
                    .child(list(
                        list_state,
                        cx.listener(Self::render_item),
                    )),
            )
            .child(
                div()
                    .flex()
                    .justify_between()
                    .p_3()
                    .border_t_1()
                    .border_color(rgb(0x333333))
                    .child(format!("{} items", item_count))
                    .text_color(rgb(0xaaaaaa))
                    .child(format!("{} done", completed_count))
                    .text_color(rgb(0x51cf66)),
            )
    }

    fn render_item(&self, index: usize, _window: &mut Window, _cx: &mut Context<Self>) -> AnyElement {
        let Some(item) = self.items.get(index) else {
            return div().into_any_element();
        };

        let text = item.text.clone();
        let completed = item.completed;

        div()
            .id(SharedString::from(format!("todo-{}", index)))
            .flex()
            .items_center()
            .gap_2()
            .px_3()
            .py_2()
            .border_b_1()
            .border_color(rgb(0x2a2a2a))
            .when(completed, |div| div.opacity(0.5))
            .child(
                div()
                    .id(SharedString::from(format!("checkbox-{}", index)))
                    .w(px(20.))
                    .h(px(20.))
                    .border_1()
                    .border_color(rgb(0x666666))
                    .rounded_sm()
                    .when(completed, |div| div.bg(rgb(0x51cf66)).border_color(rgb(0x51cf66)))
                    .on_click(cx.listener(move |this, _, _, cx| {
                        this.toggle_item(index, cx);
                    }))
                    .child(if completed {
                        div().text_size(px(14.)).text_color(rgb(0xffffff)).child("✓")
                    } else {
                        div()
                    }),
            )
            .child(
                div()
                    .flex_1()
                    .text_size(px(16.))
                    .text_color(if completed { rgb(0x888888) } else { rgb(0xffffff) })
                    .child(text),
            )
            .child(
                div()
                    .id(SharedString::from(format!("delete-{}", index)))
                    .px_2()
                    .py_1()
                    .text_size(px(12.))
                    .text_color(rgb(0xff6b6b))
                    .cursor_pointer()
                    .on_click(cx.listener(move |this, _, _, cx| {
                        this.delete_item(index, cx);
                    }))
                    .child("del"),
            )
            .into_any_element()
    }
}

fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(500.), px(600.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Todo")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| cx.new(|cx| TodoApp::new(cx)),
        );
    });
}
```

## 关键要点

- `ListState::new(count, alignment, overdraw)` 创建虚拟列表状态
- `ListState::splice(range, new_count)` 通知列表数据变更
- `list(state, render_item)` 渲染虚拟列表，只渲染可视区域内的项
- `cx.focus_handle()` 创建焦点句柄，`track_focus()` 绑定到元素
- `cx.listener` 可以捕获变量（如 index）用于闭包
- `.when(condition, |div| div.style())` 条件样式
- Enter/Escape 键通过 `on_key_down` 处理
