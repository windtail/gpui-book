# 第29章：事件传播机制

## 29.1 事件传播阶段

### 29.1.1 Capture 阶段（捕获）

事件从根元素向叶子元素传递，称为 Capture 阶段。

```rust
use gpui::{
    div, DispatchPhase, MouseDownEvent,
    InteractiveElement, ParentElement, IntoElement, Context, Window,
    px,
};

struct CaptureExample;

impl Render for CaptureExample {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("root")
            .size_full()
            // Capture 阶段监听
            .capture_any_mouse_down(cx.listener(|_, event: &MouseDownEvent, _, _| {
                println!("[Capture] root received: {:?}", event.position);
            }))
            .child(
                div()
                    .id("parent")
                    .size(px(300.0))
                    .capture_any_mouse_down(cx.listener(|_, event: &MouseDownEvent, _, _| {
                        println!("[Capture] parent received: {:?}", event.position);
                    }))
                    .child(
                        div()
                            .id("child")
                            .size(px(100.0))
                            .capture_any_mouse_down(cx.listener(|_, event: &MouseDownEvent, _, _| {
                                println!("[Capture] child received: {:?}", event.position);
                            }))
                    )
            )
    }
}
```

点击 child 区域时，打印顺序：
```
[Capture] root received: ...
[Capture] parent received: ...
[Capture] child received: ...
```

### 29.1.2 Bubble 阶段（冒泡）

事件到达目标元素后，从叶子向根返回，称为 Bubble 阶段。

```rust
div()
    .id("root")
    .on_mouse_down(MouseButton::Left, cx.listener(|_, _, _, _| {
        println!("[Bubble] root");
    }))
    .child(
        div()
            .id("parent")
            .on_mouse_down(MouseButton::Left, cx.listener(|_, _, _, _| {
                println!("[Bubble] parent");
            }))
            .child(
                div()
                    .id("child")
                    .on_mouse_down(MouseButton::Left, cx.listener(|_, _, _, _| {
                        println!("[Bubble] child");
                    }))
            )
    )
```

点击 child 区域时，打印顺序：
```
[Bubble] child
[Bubble] parent
[Bubble] root
```

### 29.1.3 `DispatchPhase` 枚举

事件处理器通过 `DispatchPhase` 判断当前处于哪个阶段：

```rust
pub enum DispatchPhase {
    Bubble,   // 冒泡阶段
    Capture,  // 捕获阶段
}
```

`.on_mouse_down()` 等默认方法只在 Bubble 阶段触发。`.capture_any_mouse_down()` 只在 Capture 阶段触发。

事件分发完整流程：
```
根元素 Capture
  → 子元素 Capture
    → 目标元素 Capture
    → 目标元素 Bubble
  → 子元素 Bubble
→ 根元素 Bubble
```

## 29.2 控制传播

### 29.2.1 `cx.stop_propagation()`

停止事件继续传播，后续元素不会收到此事件：

```rust
struct Modal;

impl Render for Modal {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("modal-backdrop")
            .size_full()
            .bg(black_500)
            .occlude()  // Hitbox 阻止底层事件
            .on_click(cx.listener(|_, _, _, cx| {
                // 关闭模态框
                cx.stop_propagation();  // 阻止向上传播
            }))
            .child(
                div()
                    .id("modal-content")
                    .on_click(cx.listener(|_, _, _, cx| {
                        // 点击内容区不关闭模态框
                        cx.stop_propagation();  // 阻止传播到 backdrop
                    }))
                    .child("模态框内容")
            )
    }
}
```

`stop_propagation()` 在同一阶段内的后续处理器仍然会执行，但不会传播到下一个阶段或父元素。

### 29.2.2 `cx.prevent_default()`

阻止事件的默认行为：

```rust
div()
    .id("custom-link")
    .on_click(cx.listener(|_, _, _, cx| {
        // 阻止默认点击行为
        cx.prevent_default();
    }))
    .child("自定义链接")
```

`prevent_default()` 和 `stop_propagation()` 可以同时使用：

```rust
.on_key_down(cx.listener(|this, event: &KeyDownEvent, _, cx| {
    if event.keystroke.to_string() == "ctrl-s" {
        this.save();
        cx.prevent_default();    // 阻止默认行为
        cx.stop_propagation();   // 阻止继续传播
    }
}))
```

## 29.3 事件分发树

### 29.3.1 事件路由算法

GPUI 的事件分发基于 Hitbox 树：

1. **命中检测**：根据鼠标/触摸位置，找出所有覆盖该位置的 Hitbox
2. **排序**：按渲染树深度排序，最深层优先
3. **Capture 分发**：从根到目标，依次调用 Capture 阶段处理器
4. **Bubble 分发**：从目标到根，依次调用 Bubble 阶段处理器

```rust
// 事件分发树示意：
//
// Window
// └── div#root (Hitbox A)
//     ├── div#sidebar (Hitbox B)
//     └── div#main (Hitbox C)
//         └── div#button (Hitbox D)
//
// 点击 button 区域：
// 目标 = Hitbox D
// Capture: A → C → D
// Bubble: D → C → A
// B 不在路径上，不接收事件
```

### 29.3.2 Hitbox 在事件分发中的作用

只有带 `id()` 的元素才创建 Hitbox：

```rust
// 没有 id() → 没有 Hitbox → 不接收鼠标事件
div()
    .size(px(100.0))
    .bg(red_500)
    // on_mouse_down 不会触发，因为没有 Hitbox

// 有 id() → 有 Hitbox → 接收鼠标事件
div()
    .id("clickable")  // 创建 Hitbox
    .size(px(100.0))
    .bg(red_500)
    .on_mouse_down(MouseButton::Left, listener)  // 会触发
```

`.occlude()` 创建不透明 Hitbox，阻止事件传递到下方：

```rust
div()
    .id("overlay")
    .size_full()
    .occlude()  // 不透明 Hitbox
```

`.occlude()` 常用于模态框、tooltip、context menu 等覆盖层。

## 29.4 事件传播实战

### 29.4.1 模态框阻止底层点击

```rust
struct App {
    show_modal: bool,
}

impl Render for App {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("app")
            .size_full()
            .on_click(cx.listener(|this, _, _, cx| {
                // 底层点击：打开模态框
                this.show_modal = true;
                cx.notify();
            }))
            .child("点击打开模态框")
            .when(self.show_modal, |d| {
                d.child(
                    div()
                        .id("modal-overlay")
                        .size_full()
                        .bg(black_500)
                        .occlude()  // 关键：阻止底层接收事件
                        .on_click(cx.listener(|this, _, _, cx| {
                            // 点击 backdrop 关闭
                            this.show_modal = false;
                            cx.notify();
                        }))
                        .child(
                            div()
                                .id("modal-content")
                                .on_click(cx.listener(|_, _, _, cx| {
                                    // 阻止传播到 overlay
                                    cx.stop_propagation();
                                }))
                                .child("模态框内容，点击不会关闭")
                        )
                )
            })
    }
}
```

### 29.4.2 嵌套列表的点击冲突

```rust
struct NestedList {
    items: Vec<ListItem>,
}

struct ListItem {
    text: String,
    children: Vec<ListItem>,
    expanded: bool,
}

impl Render for NestedList {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("nested-list")
            .children(self.render_items(&self.items, cx))
    }
}

impl NestedList {
    fn render_items<'a>(
        &'a self,
        items: &'a [ListItem],
        cx: &mut Context<Self>,
    ) -> impl IntoElement + 'a {
        let cx = cx.listener(|this, _: &ItemClick, window, cx| {
            // 处理项目点击
        });

        div().children(items.iter().map(|item| {
            div()
                .id(("item", &item.text))
                // 展开/折叠按钮
                .child(
                    div()
                        .id(("toggle", &item.text))
                        .on_click(cx.listener(move |this, _, _, cx| {
                            // 阻止传播到外层 item 点击
                            cx.stop_propagation();
                            // 切换展开状态
                        }))
                        .child(if item.expanded { "▼" } else { "▶" })
                )
                .child(&item.text)
                .when(item.expanded, |d| {
                    d.child(div().pl_4().children(/* 子项目 */))
                })
        }))
    }
}
```

嵌套列表中，子元素的点击通过 `stop_propagation()` 阻止事件冒泡到父级项目。

### 29.4.3 拖拽时的鼠标捕获

拖拽时需要全局跟踪鼠标，不受元素边界限制：

```rust
struct DraggableSlider {
    value: f32,
    dragging: bool,
}

impl Render for DraggableSlider {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let track_bounds = window.bounds();

        div()
            .id("slider-track")
            .size(px(300.0))
            .h(px(4.0))
            .bg(gray_300)
            .rounded_full()
            .on_mouse_down(MouseButton::Left, cx.listener(|this, event: &MouseDownEvent, window, cx| {
                this.dragging = true;
                this.update_value(event.position, window, cx);
                // 捕获鼠标，即使鼠标移出元素区域也继续接收事件
                window.capture_mouse(window.bounds(), cx);
                cx.notify();
            }))
            .when(self.dragging, |d| {
                d.on_mouse_move(cx.listener(|this, event: &MouseMoveEvent, window, cx| {
                    this.update_value(event.position, window, cx);
                    cx.notify();
                }))
                .on_mouse_up(MouseButton::Left, cx.listener(|this, _, window, cx| {
                    this.dragging = false;
                    window.release_mouse(cx);
                    cx.notify();
                }))
            })
    }
}

impl DraggableSlider {
    fn update_value(&mut self, position: Point<Pixels>, window: &mut Window, cx: &mut Context<Self>) {
        let track_width = 300.0;
        let x = position.x.0.clamp(0.0, track_width);
        self.value = x / track_width;
    }
}
```

`window.capture_mouse()` 让元素在拖拽期间持续接收鼠标事件，即使鼠标移动到元素外部。拖拽结束时用 `window.release_mouse()` 释放。
