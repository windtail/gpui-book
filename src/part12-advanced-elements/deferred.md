# 第 46 章：Deferred 延迟渲染

## 46.1 `deferred()` 函数

### 46.1.1 延迟布局与绘制

`deferred(child)` 将子元素的绘制推迟到所有祖先元素之后。布局仍然在正常阶段进行，但绘制在最后执行。

```rust
use gpui::deferred;

// 延迟绘制 tooltip
div()
    .relative()
    .child(div().child("Content"))
    .child(
        deferred(
            div()
                .absolute()
                .top_full()
                .mt_1()
                .child("Tooltip")
        )
    )
```

内部机制：`deferred` 在 `prepaint` 阶段调用 `window.defer_draw()`，将子元素存入延迟绘制队列。

```rust
// Deferred 的 prepaint 实现
fn prepaint(&mut self, ..., window: &mut Window, _cx: &mut App) {
    let child = self.child.take().unwrap();
    let element_offset = window.element_offset();
    window.defer_draw(child, element_offset, self.priority, None)
}
```

### 46.1.2 渲染在正常内容之上

正常绘制顺序是树的前序遍历。deferred 元素在所有正常绘制完成后按优先级排序绘制。

```
正常绘制顺序：
1. 背景 div
2. 内容 div
3. 边框 div

deferred 绘制顺序（在以上完成后）：
4. tooltip (priority=1)
5. 上下文菜单 (priority=2)
6. 模态框 (priority=3)
```

## 46.2 `Deferred` 元素

### 46.2.1 `with_priority()` 设置优先级

`priority` 值决定多个 deferred 元素之间的绘制顺序。值越大越后绘制（显示在上方）。

```rust
deferred(tooltip)
    .with_priority(1)

deferred(context_menu)
    .with_priority(10)

deferred(modal_overlay)
    .with_priority(100)

// 绘制顺序：tooltip → context_menu → modal_overlay
// modal_overlay 显示在最上层
```

### 46.2.2 优先级排序

多个 deferred 元素按 priority 值排序。相同 priority 时按注册顺序绘制。

```rust
// 常用优先级约定
const TOOLTIP_PRIORITY: usize = 1;
const CONTEXT_MENU_PRIORITY: usize = 10;
const DRAG_PREVIEW_PRIORITY: usize = 50;
const MODAL_PRIORITY: usize = 100;
```

## 46.3 使用场景

### 46.3.1 Tooltip

Tooltip 需要浮动在所有内容之上：

```rust
struct TooltipView {
    hovered: bool,
    mouse_position: gpui::Point<gpui::Pixels>,
}

impl TooltipView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let mut elements: Vec<AnyElement> = vec![
            div()
                .id("hover-target")
                .on_mouse_move(cx.listener(|this, event, _, cx| {
                    this.hovered = true;
                    this.mouse_position = event.position;
                    cx.notify();
                }))
                .on_hovered(cx.listener(|this, hovered, _, cx| {
                    this.hovered = *hovered;
                    cx.notify();
                }))
                .child("Hover me")
                .into_any_element(),
        ];

        if self.hovered {
            elements.push(
                deferred(
                    div()
                        .absolute()
                        .left(self.mouse_position.x)
                        .top(self.mouse_position.y + px(16.0))
                        .px_2()
                        .py_1()
                        .rounded_sm()
                        .bg(rgb(0x333333))
                        .text_white()
                        .text_sm()
                        .child("This is a tooltip")
                )
                .with_priority(1)
                .into_any_element()
            );
        }

        div().relative().children(elements)
    }
}
```

### 46.3.2 Context Menu

右键菜单需要遮挡正常内容：

```rust
fn build_context_menu(position: gpui::Point<gpui::Pixels>) -> impl IntoElement {
    deferred(
        anchored()
            .position(position)
            .snap_to_window_with_margin(px(4.0))
            .child(
                div()
                    .w_48()
                    .rounded_md()
                    .shadow_lg()
                    .bg_white()
                    .child(div().p_2().child("Cut"))
                    .child(div().p_2().child("Copy"))
                    .child(div().p_2().child("Paste"))
            )
    )
    .with_priority(CONTEXT_MENU_PRIORITY)
}
```

### 46.3.3 Modal

模态框需要最高优先级：

```rust
fn build_modal(content: impl IntoElement) -> impl IntoElement {
    deferred(
        div()
            .absolute()
            .inset_0()
            .bg(rgb(0x000000).opacity(0.5)) // 半透明遮罩
            .flex()
            .items_center()
            .justify_center()
            .child(
                div()
                    .w_96()
                    .rounded_lg()
                    .shadow_xl()
                    .bg_white()
                    .p_6()
                    .child(content)
            )
    )
    .with_priority(MODAL_PRIORITY)
}
```

### 46.3.4 拖拽预览

拖拽时跟随鼠标的预览元素：

```rust
struct DragPreview {
    dragging: bool,
    drag_position: gpui::Point<gpui::Pixels>,
}

impl DragPreview {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let mut elements: Vec<AnyElement> = vec![
            div()
                .id("drag-source")
                .on_mouse_down(cx.listener(|this, _, _, cx| {
                    this.dragging = true;
                    cx.notify();
                }))
                .child("Drag me")
                .into_any_element(),
        ];

        if self.dragging {
            elements.push(
                deferred(
                    div()
                        .absolute()
                        .left(self.drag_position.x)
                        .top(self.drag_position.y)
                        .opacity(0.8)
                        .child("Preview")
                )
                .with_priority(DRAG_PREVIEW_PRIORITY)
                .into_any_element()
            );
        }

        div().relative().children(elements)
    }
}
```

## 46.4 多层 Deferred 的排序

多个 deferred 共存时，优先级决定了它们的相对顺序：

```rust
div()
    .relative()
    .children(vec![
        // 正常内容
        div().child("Main content").into_any_element(),

        // Tooltip (priority=1)
        deferred(div().child("Tooltip"))
            .with_priority(1)
            .into_any_element(),

        // Context menu (priority=10)
        deferred(div().child("Menu"))
            .with_priority(10)
            .into_any_element(),

        // Modal overlay (priority=100)
        deferred(div().child("Modal"))
            .with_priority(100)
            .into_any_element(),
    ])
```

绘制顺序：Main content → Tooltip → Menu → Modal

Deferred 的关键规则：
- `deferred(child)` 延迟子元素的绘制到正常内容之后
- `with_priority(n)` 控制多个 deferred 之间的顺序
- 布局在正常阶段进行，只延迟绘制
- 适用于 tooltip、context menu、modal、drag preview 等浮层元素
