# 第 67 章：简单文本编辑器

本章构建一个支持焦点管理、Action/Keymap 系统、基本编辑操作的文本编辑器。

## Cargo.toml

```toml
[package]
name = "text-editor"
version = "0.1.0"
edition = "2024"

[dependencies]
gpui = "0.2.2"
```

## 定义 Action

```rust
use gpui::*;

actions!(editor, [Cut, Copy, Paste, SelectAll]);

#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema)]
struct InsertText {
    chars: SharedString,
}
```

`actions!` 宏为每个名称生成一个 unit struct action。`InsertText` 是带参数的 action，需要手动实现 `Action` trait 或使用 `#[derive(Action)]`。

## Editor View

```rust
struct Editor {
    text: SharedString,
    cursor: usize,
    selection: Option<Range<usize>>,
    focus_handle: FocusHandle,
    input_handler: Option<ElementInputHandler<Editor>>,
}

impl Editor {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            text: SharedString::default(),
            cursor: 0,
            selection: None,
            focus_handle: cx.focus_handle(),
            input_handler: None,
        }
    }

    fn insert_char(&mut self, ch: char, cx: &mut Context<Self>) {
        if let Some(sel) = self.selection.take() {
            self.text = SharedString::from(&self.text[..sel.start.min(self.cursor)]);
            self.cursor = self.text.len();
        }
        let mut s = self.text.to_string();
        s.insert(self.cursor, ch);
        self.text = SharedString::from(s);
        self.cursor += ch.len_utf8();
        cx.notify();
    }

    fn backspace(&mut self, cx: &mut Context<Self>) {
        if self.cursor == 0 {
            return;
        }
        if let Some(sel) = self.selection.take() {
            let start = sel.start.min(self.cursor);
            let end = sel.end.max(self.cursor);
            let mut s = self.text.to_string();
            s.replace_range(start..end, "");
            self.text = SharedString::from(s);
            self.cursor = start;
        } else {
            let mut s = self.text.to_string();
            let before = self.text[..self.cursor].chars().rev().next().unwrap();
            s.replace_range(self.cursor - before.len_utf8()..self.cursor, "");
            self.text = SharedString::from(s);
            self.cursor -= before.len_utf8();
        }
        cx.notify();
    }

    fn move_cursor_left(&mut self, cx: &mut Context<Self>) {
        if self.cursor > 0 {
            let before = self.text[..self.cursor].chars().rev().next().unwrap();
            self.cursor -= before.len_utf8();
            cx.notify();
        }
    }

    fn move_cursor_right(&mut self, cx: &mut Context<Self>) {
        let remaining = &self.text[self.cursor..];
        if let Some(ch) = remaining.chars().next() {
            self.cursor += ch.len_utf8();
            cx.notify();
        }
    }

    fn delete_selected_or_forward(&mut self, cx: &mut Context<Self>) {
        if let Some(sel) = self.selection.take() {
            let start = sel.start.min(self.cursor);
            let end = sel.end.max(self.cursor);
            let mut s = self.text.to_string();
            s.replace_range(start..end, "");
            self.text = SharedString::from(s);
            self.cursor = start;
        } else {
            let remaining = &self.text[self.cursor..];
            if let Some(ch) = remaining.chars().next() {
                let mut s = self.text.to_string();
                s.replace_range(self.cursor..self.cursor + ch.len_utf8(), "");
                self.text = SharedString::from(s);
            }
        }
        cx.notify();
    }

    fn select_all(&mut self, cx: &mut Context<Self>) {
        self.selection = Some(0..self.text.len());
        self.cursor = self.text.len();
        cx.notify();
    }
}
```

## 实现 EntityInputHandler

```rust
impl EntityInputHandler for Editor {
    fn text_for_range(
        &mut self,
        range: Range<usize>,
        adjusted_range: &mut Option<Range<usize>>,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<String> {
        if range.start > self.text.len() || range.end > self.text.len() {
            return None;
        }
        Some(self.text[range.start..range.end].to_string())
    }

    fn selected_text_range(
        &mut self,
        _ignore_disabled_input: bool,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<UTF16Selection> {
        None
    }

    fn marked_text_range(
        &self,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<Range<usize>> {
        None
    }

    fn unmark_text(&mut self, _window: &mut Window, _cx: &mut Context<Self>) {}

    fn replace_text_in_range(
        &mut self,
        range: Option<Range<usize>>,
        new_text: &str,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        let range = range.unwrap_or(self.cursor..self.cursor);
        let mut s = self.text.to_string();
        let start = range.start.min(self.text.len());
        let end = range.end.min(self.text.len());
        s.replace_range(start..end, new_text);
        self.text = SharedString::from(s);
        self.cursor = start + new_text.len();
        cx.notify();
    }

    fn replace_and_mark_text_in_range(
        &mut self,
        _range_utf16: Option<Range<usize>>,
        _new_text: &str,
        _new_selected_range: Option<Range<usize>>,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        // IME 标记文本处理（简化版本）
    }

    fn bounds_for_range(
        &mut self,
        range_utf16: Range<usize>,
        element_bounds: Bounds<Pixels>,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<Bounds<Pixels>> {
        Some(element_bounds)
    }

    fn character_index_for_point(
        &mut self,
        point: Point<Pixels>,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<usize> {
        Some(self.cursor)
    }
}
```

## Render 实现

```rust
impl Render for Editor {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 注册输入处理器
        self.input_handler = Some(ElementInputHandler::new(
            Bounds::new(Point::zero(), size(px(800.), px(600.))),
            cx.entity().clone(),
        ));
        if let Some(handler) = self.input_handler.take() {
            window.set_input_handler(handler);
        }

        let is_focused = cx.focused() == Some(self.focus_handle.clone());

        div()
            .size_full()
            .flex()
            .flex_col()
            .bg(rgb(0x1e1e1e))
            .child(
                // 顶部栏
                div()
                    .flex()
                    .items_center()
                    .px_3()
                    .py_1()
                    .bg(rgb(0x2d2d2d))
                    .border_b_1()
                    .border_color(rgb(0x3a3a3a))
                    .child("Editor")
                    .text_size(px(14.))
                    .text_color(rgb(0xcccccc)),
            )
            .child(
                // 编辑区域
                div()
                    .id("editor-area")
                    .flex_1()
                    .p_4()
                    .track_focus(&self.focus_handle)
                    .on_click(cx.listener(|this, _: &ClickEvent, window, cx| {
                        cx.focus(&this.focus_handle);
                    }))
                    .on_key_down(cx.listener(Self::on_key_down))
                    .child(
                        div()
                            .font_family("monospace")
                            .text_size(px(16.))
                            .line_height(px(24.))
                            .text_color(rgb(0xd4d4d4))
                            .child(self.render_text(cx)),
                    ),
            )
            .child(
                // 底部状态栏
                div()
                    .flex()
                    .justify_between()
                    .px_3()
                    .py_1()
                    .bg(rgb(0x2d2d2d))
                    .border_t_1()
                    .border_color(rgb(0x3a3a3a))
                    .child(if is_focused { "INSERT" } else { "NORMAL" })
                    .text_size(px(12.))
                    .text_color(if is_focused { rgb(0x51cf66) } else { rgb(0x888888) })
                    .child(format!("cursor: {}", self.cursor))
                    .text_size(px(12.))
                    .text_color(rgb(0x888888)),
            )
    }

    fn render_text(&self, cx: &mut Context<Self>) -> impl IntoElement {
        // 使用 StyledText 实现带选区高亮的渲染
        if let Some(sel) = &self.selection {
            let start = sel.start.min(self.cursor);
            let end = sel.end.max(self.cursor);

            let before = &self.text[..start];
            let selected = &self.text[start..end];
            let after = &self.text[end..];

            div()
                .child(before.to_string())
                .child(
                    span()
                        .bg(rgb(0x264f78))
                        .text_color(rgb(0xffffff))
                        .child(selected.to_string()),
                )
                .child(after.to_string())
        } else {
            div().child(self.text.to_string())
        }
    }
}
```

## 键盘事件处理

```rust
impl Editor {
    fn on_key_down(&mut self, event: &KeyDownEvent, window: &mut Window, cx: &mut Context<Self>) {
        let mods = &event.keystroke.modifiers;

        // Ctrl/Cmd + A: Select All
        if event.keystroke.key == "a" && (mods.control || mods.command) {
            self.select_all(cx);
            cx.stop_propagation();
            return;
        }

        // Ctrl/Cmd + X: Cut
        if event.keystroke.key == "x" && (mods.control || mods.command) {
            // 复制选区到剪贴板，然后删除
            cx.stop_propagation();
            return;
        }

        // Ctrl/Cmd + C: Copy
        if event.keystroke.key == "c" && (mods.control || mods.command) {
            cx.stop_propagation();
            return;
        }

        // Ctrl/Cmd + V: Paste
        if event.keystroke.key == "v" && (mods.control || mods.command) {
            // 从剪贴板粘贴
            cx.stop_propagation();
            return;
        }

        match event.keystroke.key.as_str() {
            "backspace" => self.backspace(cx),
            "delete" => self.delete_selected_or_forward(cx),
            "left" => self.move_cursor_left(cx),
            "right" => self.move_cursor_right(cx),
            "up" => { /* 简化为不处理多行 */ }
            "down" => { /* 简化为不处理多行 */ }
            "enter" => self.insert_char('\n', cx),
            "tab" => {
                for _ in 0..4 {
                    self.insert_char(' ', cx);
                }
            }
            "escape" => {
                self.selection = None;
                cx.notify();
            }
            _ => {
                // 处理普通字符输入
                if let Some(ch) = event.keystroke.key.chars().next() {
                    if !mods.control && !mods.command {
                        self.insert_char(ch, cx);
                    }
                }
            }
        }
    }
}
```

## 注册 Action 与 Keymap

在 `main` 中注册键绑定：

```rust
fn main() {
    let app = Application::new();
    app.run(|cx| {
        // 注册键绑定
        cx.bind_keys(vec![
            KeyBinding::new("ctrl-a", SelectAll, None),
            KeyBinding::new("cmd-a", SelectAll, None),
        ]);

        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(800.), px(600.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Text Editor")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| cx.new(|cx| Editor::new(cx)),
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

actions!(editor, [Cut, Copy, Paste, SelectAll]);

struct Editor {
    text: SharedString,
    cursor: usize,
    selection: Option<Range<usize>>,
    focus_handle: FocusHandle,
}

impl Editor {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            text: SharedString::default(),
            cursor: 0,
            selection: None,
            focus_handle: cx.focus_handle(),
        }
    }

    fn insert_char(&mut self, ch: char, cx: &mut Context<Self>) {
        if let Some(sel) = self.selection.take() {
            let start = sel.start.min(self.cursor);
            let end = sel.end.max(self.cursor);
            let mut s = self.text.to_string();
            s.replace_range(start..end, "");
            self.text = SharedString::from(&s);
            self.cursor = start;
        }
        let mut s = self.text.to_string();
        s.insert(self.cursor, ch);
        self.text = SharedString::from(s);
        self.cursor += ch.len_utf8();
        cx.notify();
    }

    fn backspace(&mut self, cx: &mut Context<Self>) {
        if self.cursor == 0 { return; }
        if let Some(sel) = self.selection.take() {
            let start = sel.start.min(self.cursor);
            let end = sel.end.max(self.cursor);
            let mut s = self.text.to_string();
            s.replace_range(start..end, "");
            self.text = SharedString::from(s);
            self.cursor = start;
        } else {
            let before = self.text[..self.cursor].chars().rev().next().unwrap();
            let mut s = self.text.to_string();
            s.replace_range(self.cursor - before.len_utf8()..self.cursor, "");
            self.text = SharedString::from(s);
            self.cursor -= before.len_utf8();
        }
        cx.notify();
    }

    fn move_cursor_left(&mut self, cx: &mut Context<Self>) {
        if self.cursor > 0 {
            let before = self.text[..self.cursor].chars().rev().next().unwrap();
            self.cursor -= before.len_utf8();
            cx.notify();
        }
    }

    fn move_cursor_right(&mut self, cx: &mut Context<Self>) {
        let remaining = &self.text[self.cursor..];
        if let Some(ch) = remaining.chars().next() {
            self.cursor += ch.len_utf8();
            cx.notify();
        }
    }

    fn delete_forward(&mut self, cx: &mut Context<Self>) {
        if let Some(sel) = self.selection.take() {
            let start = sel.start.min(self.cursor);
            let end = sel.end.max(self.cursor);
            let mut s = self.text.to_string();
            s.replace_range(start..end, "");
            self.text = SharedString::from(s);
            self.cursor = start;
        } else {
            let remaining = &self.text[self.cursor..];
            if let Some(ch) = remaining.chars().next() {
                let mut s = self.text.to_string();
                s.replace_range(self.cursor..self.cursor + ch.len_utf8(), "");
                self.text = SharedString::from(s);
            }
        }
        cx.notify();
    }

    fn select_all(&mut self, cx: &mut Context<Self>) {
        self.selection = Some(0..self.text.len());
        self.cursor = self.text.len();
        cx.notify();
    }

    fn on_key_down(&mut self, event: &KeyDownEvent, _window: &mut Window, cx: &mut Context<Self>) {
        let mods = &event.keystroke.modifiers;

        if event.keystroke.key == "a" && (mods.control || mods.command) {
            self.select_all(cx);
            cx.stop_propagation();
            return;
        }

        match event.keystroke.key.as_str() {
            "backspace" => self.backspace(cx),
            "delete" => self.delete_forward(cx),
            "left" => self.move_cursor_left(cx),
            "right" => self.move_cursor_right(cx),
            "enter" => self.insert_char('\n', cx),
            "tab" => {
                for _ in 0..4 { self.insert_char(' ', cx); }
            }
            "escape" => {
                self.selection = None;
                cx.notify();
            }
            _ => {
                if let Some(ch) = event.keystroke.key.chars().next() {
                    if !mods.control && !mods.command {
                        self.insert_char(ch, cx);
                    }
                }
            }
        }
    }
}

impl EntityInputHandler for Editor {
    fn text_for_range(&mut self, range: Range<usize>, _: &mut Option<Range<usize>>, _: &mut Window, _: &mut Context<Self>) -> Option<String> {
        if range.start <= self.text.len() && range.end <= self.text.len() {
            Some(self.text[range.start..range.end].to_string())
        } else { None }
    }

    fn selected_text_range(&mut self, _: bool, _: &mut Window, _: &mut Context<Self>) -> Option<UTF16Selection> { None }
    fn marked_text_range(&self, _: &mut Window, _: &mut Context<Self>) -> Option<Range<usize>> { None }
    fn unmark_text(&mut self, _: &mut Window, _: &mut Context<Self>) {}
    fn replace_text_in_range(&mut self, range: Option<Range<usize>>, new_text: &str, _: &mut Window, cx: &mut Context<Self>) {
        let range = range.unwrap_or(self.cursor..self.cursor);
        let mut s = self.text.to_string();
        let start = range.start.min(self.text.len());
        let end = range.end.min(self.text.len());
        s.replace_range(start..end, new_text);
        self.text = SharedString::from(s);
        self.cursor = start + new_text.len();
        cx.notify();
    }
    fn replace_and_mark_text_in_range(&mut self, _: Option<Range<usize>>, _: &str, _: Option<Range<usize>>, _: &mut Window, _: &mut Context<Self>) {}
    fn bounds_for_range(&mut self, _: Range<usize>, element_bounds: Bounds<Pixels>, _: &mut Window, _: &mut Context<Self>) -> Option<Bounds<Pixels>> { Some(element_bounds) }
    fn character_index_for_point(&mut self, _: Point<Pixels>, _: &mut Window, _: &mut Context<Self>) -> Option<usize> { Some(self.cursor) }
}

impl Render for Editor {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let input_handler = ElementInputHandler::new(
            Bounds::new(Point::zero(), size(px(800.), px(600.))),
            cx.entity().clone(),
        );
        window.set_input_handler(input_handler);

        let is_focused = cx.focused() == Some(self.focus_handle.clone());
        let chars = self.text.chars().count();

        div()
            .size_full()
            .flex()
            .flex_col()
            .bg(rgb(0x1e1e1e))
            .child(
                div().flex().items_center().px_3().py_1().bg(rgb(0x2d2d2d))
                    .border_b_1().border_color(rgb(0x3a3a3a))
                    .child("Editor").text_size(px(14.)).text_color(rgb(0xcccccc)),
            )
            .child(
                div().id("editor-area").flex_1().p_4()
                    .track_focus(&self.focus_handle)
                    .on_click(cx.listener(|this, _: &ClickEvent, _, cx| {
                        cx.focus(&this.focus_handle);
                    }))
                    .on_key_down(cx.listener(Self::on_key_down))
                    .child(
                        div().font_family("monospace").text_size(px(16.))
                            .line_height(px(24.)).text_color(rgb(0xd4d4d4))
                            .child(self.text.to_string()),
                    ),
            )
            .child(
                div().flex().justify_between().px_3().py_1().bg(rgb(0x2d2d2d))
                    .border_t_1().border_color(rgb(0x3a3a3a))
                    .child(if is_focused { "INSERT" } else { "NORMAL" })
                        .text_size(px(12.))
                        .text_color(if is_focused { rgb(0x51cf66) } else { rgb(0x888888) })
                    .child(format!("{} chars, cursor: {}", chars, self.cursor))
                        .text_size(px(12.)).text_color(rgb(0x888888)),
            )
    }
}

fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.bind_keys(vec![
            KeyBinding::new("ctrl-a", SelectAll, None),
            KeyBinding::new("cmd-a", SelectAll, None),
        ]);
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(800.), px(600.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Text Editor")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| cx.new(|cx| Editor::new(cx)),
        );
    });
}
```

## 关键要点

- `actions!` 宏快速定义无参数 action，适合快捷键绑定
- `cx.bind_keys()` 注册全局键绑定，`KeyBinding::new("ctrl-a", Action, context)`
- `cx.focus_handle()` 创建焦点，`track_focus()` 绑定到元素，`cx.focus()` 设置焦点
- `EntityInputHandler` trait 实现 IME 和文本输入支持
- `ElementInputHandler` 桥接 View 和输入系统
- `window.set_input_handler()` 在 `render()` 中注册输入处理器
- `cx.stop_propagation()` 阻止事件冒泡
- 光标位置用 UTF-8 字节偏移，字符操作需用 `char.len_utf8()`
