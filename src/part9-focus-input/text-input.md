# 第35章：文本输入与 IME

## 35.1 `EntityInputHandler` trait

### 35.1.1 trait 方法概览

`EntityInputHandler` 让 View 支持文本输入和 IME：

```rust
pub trait EntityInputHandler: 'static + Sized {
    fn text_for_range(
        &mut self,
        range: Range<usize>,
        adjusted_range: &mut Option<Range<usize>>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Option<String>;

    fn selected_text_range(
        &mut self,
        ignore_disabled_input: bool,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Option<UTF16Selection>;

    fn marked_text_range(
        &self,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Option<Range<usize>>;

    fn unmark_text(&mut self, window: &mut Window, cx: &mut Context<Self>);

    fn replace_text_in_range(
        &mut self,
        range: Option<Range<usize>>,
        text: &str,
        window: &mut Window,
        cx: &mut Context<Self>,
    );

    fn replace_and_mark_text_in_range(
        &mut self,
        range: Option<Range<usize>>,
        new_text: &str,
        new_selected_range: Option<Range<usize>>,
        window: &mut Window,
        cx: &mut Context<Self>,
    );

    fn bounds_for_range(
        &mut self,
        range_utf16: Range<usize>,
        element_bounds: Bounds<Pixels>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Option<Bounds<Pixels>>;

    fn character_index_for_point(
        &mut self,
        point: Point<Pixels>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Option<usize>;

    fn accepts_text_input(&self, _window: &mut Window, _cx: &mut Context<Self>) -> bool {
        true
    }
}
```

所有范围参数都是 UTF-16 索引。

### 35.1.2 `text_for_range()` 获取文本

平台（如 macOS 的 NSTextInputClient）调用此方法获取指定范围的文本：

```rust
fn text_for_range(
    &mut self,
    range: Range<usize>,       // UTF-16 范围
    adjusted_range: &mut Option<Range<usize>>,
    _window: &mut Window,
    _cx: &mut Context<Self>,
) -> Option<String> {
    // 将 UTF-16 范围转为 UTF-8 范围
    let utf8_range = self.utf16_to_utf8_range(range)?;

    // 返回对应文本
    adjusted_range.replace(self.utf8_to_utf16_range(utf8_range.clone())?);
    Some(self.content[utf8_range].to_string())
}
```

### 35.1.3 `replace_text_in_range()` 替换文本

平台在用户输入时调用此方法：

```rust
fn replace_text_in_range(
    &mut self,
    range: Option<Range<usize>>,  // UTF-16 范围，None 表示在光标处插入
    text: &str,
    _window: &mut Window,
    _cx: &mut Context<Self>,
) {
    // 将 UTF-16 范围转为 UTF-8 范围
    let utf8_range = range
        .map(|r| self.utf16_to_utf8_range(r))
        .flatten();

    if let Some(range) = utf8_range {
        self.content.replace_range(range, text);
    } else {
        // 在光标处插入
        let cursor = self.cursor_utf8();
        self.content.insert_str(cursor, text);
        self.move_cursor_forward(text.len());
    }

    self.marked_range = None;  // 替换后清除标记
}
```

### 35.1.4 `selected_text_range()` 选区

返回当前选区的 UTF-16 范围：

```rust
fn selected_text_range(
    &mut self,
    _ignore_disabled_input: bool,
    _window: &mut Window,
    _cx: &mut Context<Self>,
) -> Option<UTF16Selection> {
    if self.selection_len > 0 {
        Some(UTF16Selection {
            range: self.selection_start_utf16..self.selection_end_utf16,
            reversed: false,
        })
    } else {
        // 无选区，返回光标位置
        Some(UTF16Selection {
            range: self.cursor_utf16..self.cursor_utf16,
            reversed: false,
        })
    }
}
```

## 35.2 `ElementInputHandler`

### 35.2.1 关联 View 与输入处理

`ElementInputHandler<V>` 在元素渲染时将 View 与平台输入系统连接：

```rust
use gpui::{ElementInputHandler, Window, Context, Bounds, px};

struct TextInput {
    content: String,
    cursor: usize,
    marked_range: Option<Range<usize>>,
}

impl TextInput {
    fn paint_input_handler(
        &mut self,
        element_bounds: Bounds<gpui::Pixels>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        // 在 paint 阶段注册输入处理器
        let handler = ElementInputHandler::new(element_bounds, cx.entity());
        window.handle_input(handler, cx);
    }
}
```

### 35.2.2 注册与注销

`ElementInputHandler` 在 `Element::paint` 中注册：

```rust
impl Element for TextInputElement {
    fn paint(
        &mut self,
        id: Option<&GlobalElementId>,
        bounds: Bounds<Pixels>,
        window: &mut Window,
        cx: &mut App,
    ) {
        // 1. 获取 View 的 bounds
        let element_bounds = bounds;

        // 2. 创建输入处理器
        let handler = ElementInputHandler::new(element_bounds, self.view.clone());

        // 3. 注册到窗口
        window.handle_input(handler, cx);

        // 4. 正常绘制文本
        self.paint_text(bounds, window, cx);
    }
}
```

注销不需要手动操作。`handle_input` 在每帧开始时设置，新的调用自动覆盖旧的。

## 35.3 IME 支持

### 35.3.1 标记文本 (marked text)

IME 输入过程中，被标记的文本处于"候选"状态，尚未确认：

```
输入 "你好" 的 IME 流程：

1. 用户按 n → marked text = "n"
2. 用户按 i → marked text = "ni"
3. 选择候选词 → marked text = "你"
4. 确认 → marked text = None, content += "你"
5. 用户按 n → marked text = "n"
6. 用户按 i → marked text = "ni"
7. 选择候选词 → marked text = "你好"
8. 确认 → marked text = None, content += "你好"
```

### 35.3.2 `marked_text_range()`

返回当前标记文本的范围：

```rust
fn marked_text_range(
    &self,
    _window: &mut Window,
    _cx: &mut Context<Self>,
) -> Option<Range<usize>> {
    // 返回 UTF-16 范围
    self.marked_range.clone()
}
```

### 35.3.3 `replace_and_mark_text_in_range()`

IME 引擎在用户选择候选词时调用：

```rust
fn replace_and_mark_text_in_range(
    &mut self,
    range_utf16: Option<Range<usize>>,
    new_text: &str,
    new_selected_range: Option<Range<usize>>,
    _window: &mut Window,
    _cx: &mut Context<Self>,
) {
    // 替换文本并设置新的标记范围
    let utf8_range = range_utf16
        .and_then(|r| self.utf16_to_utf8_range(r));

    if let Some(range) = utf8_range {
        self.content.replace_range(range, new_text);
    } else {
        // 在光标处插入
        let cursor = self.cursor_utf8();
        self.content.insert_str(cursor, new_text);
    }

    // 更新标记范围
    if let Some(selected) = new_selected_range {
        // 转换为 UTF-8 并记录
        self.marked_range = Some(self.utf16_to_utf8_range(selected)?);
    }
}
```

### 35.3.4 `unmark_text()`

IME 输入结束时调用，清除标记状态：

```rust
fn unmark_text(&mut self, _window: &mut Window, _cx: &mut Context<Self>) {
    self.marked_range = None;
}
```

## 35.4 文本范围操作

### 35.4.1 UTF-8 vs UTF-16 范围

平台 API 使用 UTF-16 索引，而 Rust 字符串是 UTF-8。需要在两者之间转换：

```rust
impl TextInput {
    /// UTF-16 索引转 UTF-8 字节索引
    fn utf16_to_utf8_index(&self, utf16_index: usize) -> Option<usize> {
        if utf16_index == 0 {
            return Some(0);
        }

        let mut utf16_count = 0;
        for (byte_idx, ch) in self.content.char_indices() {
            let ch_len = ch.len_utf16();
            if utf16_count + ch_len > utf16_index {
                return Some(byte_idx);
            }
            utf16_count += ch_len;
        }

        Some(self.content.len())
    }

    /// UTF-8 字节索引转 UTF-16 索引
    fn utf8_to_utf16_index(&self, utf8_index: usize) -> Option<usize> {
        let mut utf16_count = 0;
        for (byte_idx, ch) in self.content.char_indices() {
            if byte_idx >= utf8_index {
                return Some(utf16_count);
            }
            utf16_count += ch.len_utf16();
        }
        Some(utf16_count)
    }

    /// UTF-16 范围转 UTF-8 范围
    fn utf16_to_utf8_range(&self, utf16_range: Range<usize>) -> Option<Range<usize>> {
        let start = self.utf16_to_utf8_index(utf16_range.start)?;
        let end = self.utf16_to_utf8_index(utf16_range.end)?;
        Some(start..end)
    }
}
```

### 35.4.2 `bounds_for_range()` 光标定位

平台需要知道文本范围的屏幕位置，用于定位 IME 候选窗口：

```rust
fn bounds_for_range(
    &mut self,
    range_utf16: Range<usize>,
    element_bounds: Bounds<Pixels>,
    _window: &mut Window,
    _cx: &mut Context<Self>,
) -> Option<Bounds<Pixels>> {
    // 将 UTF-16 范围转为光标位置
    let start = self.utf16_to_utf8_index(range_utf16.start)?;

    // 计算光标的屏幕位置（简化版，实际需要文本布局引擎）
    let x = self.compute_x_for_byte_index(start);
    let y = self.compute_y_for_line(self.line_for_byte_index(start));

    Some(Bounds {
        origin: point(element_bounds.origin.x + x, element_bounds.origin.y + y),
        size: size(px(1.0), self.line_height()),
    })
}
```

### 35.4.3 `character_index_for_point()` 点击定位

用户点击文本时，平台调用此方法确定对应哪个字符：

```rust
fn character_index_for_point(
    &mut self,
    point: Point<Pixels>,
    _window: &mut Window,
    _cx: &mut Context<Self>,
) -> Option<usize> {
    // 将像素坐标转为 UTF-16 字符索引
    let line = self.line_for_y(point.y);
    let char_idx = self.char_index_for_x(line, point.x);

    // 返回 UTF-16 索引
    self.utf8_to_utf16_index(self.byte_index_for_char(line, char_idx))
}
```

## 35.5 实战：实现一个简单的文本输入框

```rust
use gpui::{
    ElementInputHandler, FocusHandle, Bounds, Pixels, Point,
    Context, Window, div, InteractiveElement, IntoElement,
    ParentElement, px, SharedString,
};
use std::ops::Range;

struct SimpleInput {
    focus_handle: FocusHandle,
    content: String,
    cursor: usize,           // UTF-8 字节索引
    marked_range: Option<Range<usize>>,  // UTF-8 范围
}

impl SimpleInput {
    fn new(cx: &mut Context<Self>) -> Self {
        SimpleInput {
            focus_handle: cx.focus_handle(),
            content: String::new(),
            cursor: 0,
            marked_range: None,
        }
    }
}

impl EntityInputHandler for SimpleInput {
    fn text_for_range(
        &mut self,
        range_utf16: Range<usize>,
        adjusted_range: &mut Option<Range<usize>>,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<String> {
        let range = self.utf16_to_utf8_range(range_utf16)?;
        adjusted_range.replace(self.utf8_to_utf16_range(range.clone())?);
        Some(self.content.get(range)?.to_string())
    }

    fn selected_text_range(
        &mut self,
        _ignore_disabled_input: bool,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<gpui::UTF16Selection> {
        Some(gpui::UTF16Selection {
            range: self.cursor_utf16()..self.cursor_utf16(),
            reversed: false,
        })
    }

    fn marked_text_range(
        &self,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<Range<usize>> {
        self.marked_range
            .as_ref()
            .map(|r| self.utf8_to_utf16_range(r.clone()))
    }

    fn unmark_text(&mut self, _window: &mut Window, _cx: &mut Context<Self>) {
        self.marked_range = None;
    }

    fn replace_text_in_range(
        &mut self,
        range: Option<Range<usize>>,
        new_text: &str,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) {
        // 清除标记
        if let Some(range) = &self.marked_range {
            self.content.drain(range.clone());
            self.cursor = range.start;
            self.marked_range = None;
        }

        if let Some(range) = range
            .and_then(|r| self.utf16_to_utf8_range(r))
        {
            self.content.replace_range(range.clone(), new_text);
            self.cursor = range.start + new_text.len();
        } else {
            self.content.insert_str(self.cursor, new_text);
            self.cursor += new_text.len();
        }
    }

    fn replace_and_mark_text_in_range(
        &mut self,
        range_utf16: Option<Range<usize>>,
        new_text: &str,
        new_selected_range: Option<Range<usize>>,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) {
        // 先清除旧的标记文本
        if let Some(old_range) = &self.marked_range {
            self.content.drain(old_range.clone());
            self.cursor = old_range.start;
        }

        // 插入新文本
        self.content.insert_str(self.cursor, new_text);

        // 设置新的标记范围
        if let Some(selected) = new_selected_range {
            if let Some(range) = self.utf16_to_utf8_range(selected) {
                // 调整到实际插入位置
                let adjusted = (range.start + self.cursor)..(range.end + self.cursor);
                self.marked_range = Some(adjusted);
            }
        } else {
            self.marked_range = Some(self.cursor..(self.cursor + new_text.len()));
        }

        self.cursor += new_text.len();
    }

    fn bounds_for_range(
        &mut self,
        range_utf16: Range<usize>,
        element_bounds: Bounds<Pixels>,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<Bounds<Pixels>> {
        let utf8_idx = self.utf16_to_utf8_index(range_utf16.start)?;
        let x = self.compute_cursor_x(utf8_idx);

        Some(Bounds {
            origin: point(element_bounds.origin.x + x, element_bounds.origin.y),
            size: size(px(1.0), px(20.0)),
        })
    }

    fn character_index_for_point(
        &mut self,
        point: Point<Pixels>,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> Option<usize> {
        let byte_idx = self.byte_index_for_x(point.x);
        self.utf8_to_utf16_index(byte_idx)
    }

    fn accepts_text_input(&self, _window: &mut Window, _cx: &mut Context<Self>) -> bool {
        true
    }
}

impl SimpleInput {
    // UTF-8/UTF-16 转换辅助方法
    fn cursor_utf16(&self) -> usize {
        self.content[..self.cursor].chars().map(|c| c.len_utf16()).sum()
    }

    fn utf16_to_utf8_index(&self, utf16_index: usize) -> Option<usize> {
        let mut count = 0;
        for (byte_idx, ch) in self.content.char_indices() {
            if count >= utf16_index {
                return Some(byte_idx);
            }
            count += ch.len_utf16();
        }
        Some(self.content.len())
    }

    fn utf8_to_utf16_index(&self, utf8_index: usize) -> usize {
        self.content[..utf8_index].chars().map(|c| c.len_utf16()).sum()
    }

    fn utf16_to_utf8_range(&self, range: Range<usize>) -> Option<Range<usize>> {
        let start = self.utf16_to_utf8_index(range.start)?;
        let end = self.utf16_to_utf8_index(range.end)?;
        Some(start..end)
    }

    fn utf8_to_utf16_range(&self, range: Range<usize>) -> Range<usize> {
        let start = self.utf8_to_utf16_index(range.start);
        let end = self.utf8_to_utf16_index(range.end);
        start..end
    }

    fn compute_cursor_x(&self, byte_idx: usize) -> Pixels {
        // 简化：假设等宽字体，每字符 8px
        let char_count = self.content[..byte_idx].chars().count();
        px(char_count as f32 * 8.0)
    }

    fn byte_index_for_x(&self, x: Pixels) -> usize {
        // 简化：等宽字体反向计算
        let char_count = (x.0 / 8.0).round() as usize;
        let mut count = 0;
        for (byte_idx, _ch) in self.content.char_indices() {
            if count >= char_count {
                return byte_idx;
            }
            count += 1;
        }
        self.content.len()
    }
}

impl Render for SimpleInput {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 获取元素 bounds 用于输入处理器
        // 实际使用中需要在自定义 Element 的 paint 中注册
        div()
            .id("simple-input")
            .track_focus(&self.focus_handle)
            .border_1()
            .border_color(if cx.has_focus(&self.focus_handle) { blue_500 } else { gray_300 })
            .p_2()
            .child(&self.content)
            .when_some(&self.marked_range, |d, range| {
                // 显示 IME 标记文本（可以用下划线或背景色区分）
                d.child(format!(" [IME: {}]", &self.content[range.clone()]))
            })
    }
}
```

这个实现覆盖了：
- UTF-8/UTF-16 范围转换
- 基本文本插入和删除
- IME 标记文本生命周期
- 光标位置与屏幕坐标的互转

实际项目中还需要处理键盘事件（方向键移动光标、Backspace 删除等），这些通常通过 Action 系统完成。
