# 第26章：鼠标事件

## 26.1 鼠标点击

### 26.1.1 `MouseDownEvent` 与 `MouseUpEvent`

鼠标按下和释放是两个独立事件。

```rust
use gpui::{
    div, MouseButton, MouseDownEvent, MouseUpEvent,
    InteractiveElement, ParentElement, IntoElement, Context, Window,
    px,
};

struct Counter {
    count: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("counter")
            .size(px(200.0))
            .bg(cx.theme().background)
            .on_mouse_down(MouseButton::Left, cx.listener(|this, event: &MouseDownEvent, _, cx| {
                // event.position: Point<Pixels> — 鼠标相对窗口位置
                // event.modifiers: Modifiers — 修饰键
                // event.click_count: usize — 连击次数
                // event.first_mouse: bool — 是否首次聚焦点击
                this.count += 1;
                cx.notify();
            }))
            .on_mouse_up(MouseButton::Left, cx.listener(|this, _event: &MouseUpEvent, _, cx| {
                this.count -= 1;
                cx.notify();
            }))
            .child(format!("count: {}", self.count))
    }
}
```

`MouseDownEvent` 结构：

```rust
pub struct MouseDownEvent {
    pub button: MouseButton,
    pub position: Point<Pixels>,
    pub modifiers: Modifiers,
    pub click_count: usize,
    pub first_mouse: bool,
}
```

`MouseUpEvent` 没有 `first_mouse` 字段，其余相同。

### 26.1.2 `MouseClickEvent` 双击检测

`MouseClickEvent` 在鼠标按下并释放在同一元素上时生成，包含 `down` 和 `up`。

```rust
use gpui::{ClickEvent, div, InteractiveElement};

div()
    .id("clickable")
    .on_click(cx.listener(|this, event: &ClickEvent, _, cx| {
        // 双击检测
        if event.click_count() >= 2 {
            println!("double click!");
        }

        // 右键点击
        if event.is_right_click() {
            println!("right click");
        }

        // 中键点击
        if event.is_middle_click() {
            println!("middle click");
        }

        // 标准左键点击（键盘生成的点击也返回 true）
        if event.standard_click() {
            this.count += 1;
            cx.notify();
        }

        // 是否键盘生成的点击
        if event.is_keyboard() {
            println!("keyboard click");
        }
    }))
    .child("点击我")
```

`ClickEvent` 还包含 `KeyboardClickEvent`（Enter/Space 键触发），用于无障碍访问。

### 26.1.3 `MouseButton` 枚举

```rust
pub enum MouseButton {
    Left,
    Right,
    Middle,
    Navigate(NavigationDirection),
}

pub enum NavigationDirection {
    Back,
    Forward,
}
```

监听特定按钮：

```rust
div()
    .id("multi-button")
    .on_mouse_down(MouseButton::Left, cx.listener(|_, _, _, _| {
        println!("左键按下");
    }))
    .on_mouse_down(MouseButton::Right, cx.listener(|_, _, _, _| {
        println!("右键按下");
    }))
```

监听任意鼠标按钮，使用 `.on_any_mouse_down()`（需要 `.id()` 返回 `Stateful<Div>`）：

```rust
div()
    .id("any-click")
    .on_any_mouse_down(cx.listener(|_, event: &MouseDownEvent, _, _| {
        match event.button {
            MouseButton::Left => println!("左"),
            MouseButton::Right => println!("右"),
            MouseButton::Middle => println!("中"),
            MouseButton::Navigate(NavigationDirection::Back) => println!("后退"),
            MouseButton::Navigate(NavigationDirection::Forward) => println!("前进"),
        }
    }))
```

## 26.2 鼠标移动

### 26.2.1 `MouseMoveEvent`

```rust
struct Tracker {
    position: Point<Pixels>,
    dragging: bool,
}

impl Render for Tracker {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("tracker")
            .size_full()
            .on_mouse_move(cx.listener(|this, event: &MouseMoveEvent, _, cx| {
                // event.position: Point<Pixels>
                // event.pressed_button: Option<MouseButton>
                // event.modifiers: Modifiers

                this.position = event.position;
                this.dragging = event.dragging();
                cx.notify();
            }))
            .child(format!("x: {}, y: {}, dragging: {}",
                this.position.x.0, this.position.y.0, this.dragging))
    }
}
```

### 26.2.2 拖拽检测

`MouseMoveEvent` 的 `pressed_button` 字段指示哪个按钮在按住：

```rust
struct DraggablePanel {
    offset: Point<Pixels>,
    drag_start: Option<Point<Pixels>>,
}

impl Render for DraggablePanel {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("draggable")
            .size(px(100.0))
            .bg(red_500)
            .on_mouse_down(MouseButton::Left, cx.listener(|this, event: &MouseDownEvent, _, cx| {
                this.drag_start = Some(event.position);
                cx.notify();
            }))
            .on_mouse_move(cx.listener(|this, event: &MouseMoveEvent, _, cx| {
                if event.pressed_button == Some(MouseButton::Left) {
                    if let Some(start) = this.drag_start {
                        this.offset += event.position - start;
                        this.drag_start = Some(event.position);
                        cx.notify();
                    }
                }
            }))
            .on_mouse_up(MouseButton::Left, cx.listener(|this, _, _, cx| {
                this.drag_start = None;
                cx.notify();
            }))
            .child("拖拽我")
    }
}
```

### 26.2.3 `MouseExitEvent`

鼠标离开窗口时触发：

```rust
struct HoverPanel {
    hovered: bool,
}

impl Render for HoverPanel {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("hover-panel")
            .on_mouse_move(cx.listener(|this, _, _, cx| {
                if !this.hovered {
                    this.hovered = true;
                    cx.notify();
                }
            }))
            .on_global_mouse_exit(cx.listener(|this, _, _, cx| {
                this.hovered = false;
                cx.notify();
            }))
            .when(self.hovered, |d| d.child("鼠标在窗口内"))
    }
}
```

## 26.3 滚动

### 26.3.1 `ScrollWheelEvent`

```rust
struct Scrollable {
    offset: f32,
    line_height: Pixels,
}

impl Render for Scrollable {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("scrollable")
            .size_full()
            .on_scroll_wheel(cx.listener(|this, event: &ScrollWheelEvent, _, cx| {
                let delta = event.delta.pixel_delta(this.line_height);
                this.offset += delta.y.0;
                cx.notify();
            }))
            .child(format!("scroll offset: {}", this.offset))
    }
}
```

### 26.3.2 `ScrollDelta`：Pixels vs Lines

```rust
pub enum ScrollDelta {
    Pixels(Point<Pixels>),  // 精确像素（触控板）
    Lines(Point<f32>),      // 行数（传统滚轮）
}

impl ScrollDelta {
    pub fn precise(&self) -> bool;
    pub fn pixel_delta(&self, line_height: Pixels) -> Point<Pixels>;
}
```

触控板产生 `Pixels`，传统滚轮产生 `Lines`。用 `pixel_delta(line_height)` 统一转换：

```rust
let line_height = px(20.0);
let delta = event.delta.pixel_delta(line_height);
// Lines(1.0, 0.0) → Pixels(20.0, 0.0)
// Pixels(5.0, 0.0) → Pixels(5.0, 0.0)
```

## 26.4 压力感应

### 26.4.1 `MousePressureEvent`

仅 macOS Force Touch 触控板：

```rust
pub struct MousePressureEvent {
    pub pressure: f32,           // 0.0 ~ 1.0
    pub stage: PressureStage,
    pub position: Point<Pixels>,
    pub modifiers: Modifiers,
}
```

### 26.4.2 `PressureStage`

```rust
pub enum PressureStage {
    Zero,     // 无压力
    Normal,   // 普通点击
    Force,    // Force Click
}

div()
    .id("pressure-sensitive")
    .on_mouse_pressure(cx.listener(|_, event: &MousePressureEvent, _, _| {
        match event.stage {
            PressureStage::Zero => println!("无压力"),
            PressureStage::Normal => println!("普通点击"),
            PressureStage::Force => println!("Force Click!"),
        }
    }))
    .child("用力按压")
```

## 26.5 文件拖放

### 26.5.1 `FileDropEvent` 状态机

```rust
pub enum FileDropEvent {
    Entered(ExternalPaths),   // 文件拖入区域，携带文件路径
    Pending,                  // 拖动中，位置更新
    Submit,                   // 释放文件
    Exited,                   // 拖出区域
}

pub struct ExternalPaths(pub SmallVec<[PathBuf; 2]>);
impl ExternalPaths {
    pub fn paths(&self) -> &[PathBuf];
}
```

### 26.5.2 完整拖放示例

```rust
use std::path::PathBuf;

struct FileDropArea {
    files: Option<Vec<PathBuf>>,
    is_hovering: bool,
}

impl Render for FileDropArea {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("drop-zone")
            .size(px(300.0))
            .border_2()
            .border_dashed()
            .border_color(if self.is_hovering { green_500 } else { gray_300 })
            .on_drag_move::<ExternalPaths>(
                cx.listener(|this, _event: &DragMoveEvent<ExternalPaths>, _, cx| {
                    this.is_hovering = true;
                    cx.notify();
                }),
            )
            .on_drop(cx.listener(|this, paths: Option<&ExternalPaths>, _, cx| {
                if let Some(ExternalPaths(paths)) = paths {
                    this.files = Some(paths.clone().into_vec());
                }
                this.is_hovering = false;
                cx.notify();
            }))
            .child(if self.is_hovering { "释放文件" } else { "拖放文件" })
            .when_some(&self.files, |d, files| {
                d.children(files.iter().map(|p| div().child(p.display().to_string())))
            })
    }
}
```

状态流转：`Entered`（附带路径）→ `Pending`（只更新位置）→ `Submit`（释放）或 `Exited`（拖出）。

## 26.6 事件注册

### 26.6.1 `.on_mouse_down()` / `.on_mouse_up()`

```rust
div()
    .id("button")
    .on_mouse_down(MouseButton::Left, listener)
    .on_mouse_up(MouseButton::Left, listener)
```

### 26.6.2 `.on_click()`

```rust
div()
    .id("clickable")
    .on_click(cx.listener(|this, event: &ClickEvent, _, cx| {
        if event.standard_click() {
            this.do_something();
            cx.notify();
        }
    }))
```

### 26.6.3 `.on_hover()`

```rust
div()
    .id("hoverable")
    .on_hover(cx.listener(|this, is_hovered: &bool, _, cx| {
        this.show_tooltip = *is_hovered;
        cx.notify();
    }))
```

### 26.6.4 `.on_scroll_wheel()`

```rust
div()
    .id("scrollable")
    .on_scroll_wheel(cx.listener(|this, event: &ScrollWheelEvent, _, cx| {
        let delta = event.delta.pixel_delta(this.line_height);
        this.scroll_offset += delta.y.0;
        cx.notify();
    }))
```

### 26.6.5 拖拽事件

```rust
struct TodoList {
    items: Vec<String>,
}

impl Render for TodoList {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("todo-list")
            .size_full()
            .on_drag_move::<String>(cx.listener(|_this, event: &DragMoveEvent<String>, _, _| {
                println!("dragging: {}", event.drag.value());
            }))
            .on_drop(cx.listener(|this, text: Option<&String>, _, cx| {
                if let Some(text) = text {
                    this.items.push(text.clone());
                    cx.notify();
                }
            }))
            .children(self.items.iter().enumerate().map(|(i, item)| {
                div()
                    .id(("item", i))
                    .draggable(item.clone())
                    .child(item.clone())
            }))
    }
}
```

`DragMoveEvent<T>` 携带类型化的拖拽数据，`event.drag.value()` 获取 `T`。

## 26.7 Hitbox 与命中检测

### 26.7.1 Hitbox 是什么

带 `id()` 的交互式元素会创建 Hitbox。Hitbox 是鼠标事件分发的矩形区域。事件分发时，GPUI 找出鼠标位置落在哪个 Hitbox 内，将事件分发给对应元素。

嵌套 Hitbox 中，最深层的优先接收事件。

### 26.7.2 自定义 Hitbox

`.occlude()` 阻止底层元素接收鼠标事件：

```rust
div()
    .id("overlay")
    .size_full()
    .occlude()  // 不透明 Hitbox，阻止事件传递到下方
    .on_mouse_down(MouseButton::Left, listener)
```

```rust
// 模态框阻止底层点击
div()
    .id("modal-backdrop")
    .size_full()
    .bg(black.opacity(0.5))
    .occlude()
    .on_click(cx.listener(|_, _, _, _| {
        // 点击 backdrop 关闭
    }))
```
