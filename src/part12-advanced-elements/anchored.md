# 第 45 章：Anchored 弹出层

## 45.1 `anchored()` 函数

### 45.1.1 锚点定位原理

`anchored()` 创建一个相对于锚点定位的元素，并自动避免溢出窗口边界。它的核心机制：

1. 在 `prepaint` 阶段计算子元素的实际尺寸
2. 根据锚点位置计算初始位置
3. 检测是否溢出窗口边界
4. 根据 `fit_mode` 调整位置

```rust
use gpui::anchored;

// 在 (100, 200) 位置显示一个弹出层
anchored()
    .position(point(px(100.0), px(200.0)))
    .child(div().p_2().bg_white().child("Popup"))
```

### 45.1.2 基本用法

```rust
fn build_popup() -> impl IntoElement {
    anchored()
        .anchor(Anchor::TopLeft)     // 用弹出层的左上角对齐锚点
        .position(point(px(50.0), px(50.0)))
        .offset(point(px(5.0), px(5.0))) // 额外偏移
        .child(
            div()
                .w_48()
                .rounded_md()
                .shadow_md()
                .bg_white()
                .p_2()
                .child("Hello Popup")
        )
}
```

## 45.2 `Anchored` 元素

### 45.2.1 锚点方向

`Anchor` 枚举定义哪个角对齐锚点：

```rust
use gpui::Anchor;

// 弹出层的左上角对齐锚点
anchored().anchor(Anchor::TopLeft)

// 弹出层的右上角对齐锚点
anchored().anchor(Anchor::TopRight)

// 弹出层的左下角对齐锚点
anchored().anchor(Anchor::BottomLeft)

// 弹出层的右下角对齐锚点
anchored().anchor(Anchor::BottomRight)
```

选择原则：按钮下方的菜单通常用 `TopLeft` 或 `TopRight`，上方的 tooltip 用 `BottomLeft`。

### 45.2.2 偏移量

`offset()` 在锚点定位后添加额外位移：

```rust
anchored()
    .position(trigger_bounds.origin)
    .offset(point(px(0.0), px(4.0))) // 向下偏移 4px，产生间隙
    .child(menu_content)
```

## 45.3 适配模式

### 45.3.1 `AnchoredFitMode`

默认使用 `SwitchAnchor`，弹出层空间不足时自动翻转锚点方向：

```rust
// 默认：锚点翻转
anchored()
    .anchor(Anchor::TopLeft)
    .position(point(px(900.0), px(50.0)))
    .child(div().w_64().h_48())
// 如果向右弹出会超出窗口，自动翻转为向左弹出
```

### 45.3.2 `SnapToWindow`

`snap_to_window()` 将弹出层吸附到窗口边缘而不是翻转锚点：

```rust
anchored()
    .snap_to_window()
    .position(point(px(900.0), px(50.0)))
    .child(div().w_64().child("Content"))
// 向右溢出时，弹出层被截断在窗口右边缘
```

### 45.3.3 `SnapToWindowWithMargin`

`snap_to_window_with_margin()` 吸附时保留边距：

```rust
anchored()
    .snap_to_window_with_margin(px(8.0))
    .position(point(px(900.0), px(50.0)))
    .child(div().w_64().child("Content"))
// 弹出层右边缘距离窗口右边缘 8px
```

### 45.3.4 `SwitchAnchor` 翻转锚点

`SwitchAnchor`（默认模式）的翻转逻辑：

```rust
// 假设锚点在窗口右上角，弹出层宽度超过剩余水平空间

// 初始：TopLeft → 向右展开 → 超出右边界
// 翻转：TopRight → 向左展开 → 不超出边界 ✓

// 水平方向翻转后，如果垂直方向也超出，再翻转垂直方向
// 最终保证弹出层在窗口内
```

## 45.4 定位模式

### 45.4.1 `Window` 模式

`AnchoredPositionMode::Window`（默认）：位置相对于窗口原点。

```rust
anchored()
    .position_mode(AnchoredPositionMode::Window)
    .position(point(px(100.0), px(100.0)))
    .child(div().child("Window-relative"))
```

### 45.4.2 `Local` 模式

`AnchoredPositionMode::Local`：位置相对于父元素原点。

```rust
div()
    .relative()
    .p_4()
    .child(
        anchored()
            .position_mode(AnchoredPositionMode::Local)
            .position(point(px(10.0), px(10.0)))
            .child(div().child("Parent-relative"))
    )
```

## 45.5 实战：下拉菜单、Tooltip、Popover

### 下拉菜单

```rust
use gpui::{anchored, anchored::AnchoredFitMode, Anchor, Context, div, IntoElement, Styled, prelude::*};

struct Dropdown {
    open: bool,
    trigger_bounds: Option<gpui::Bounds<gpui::Pixels>>,
    _menu_sub: gpui::Subscription,
}

impl Dropdown {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let mut elements = Vec::new();

        // 触发按钮
        elements.push(
            div()
                .id("dropdown-trigger")
                .on_click(cx.listener(|this, _, window, cx| {
                    this.open = !this.open;
                    cx.notify();
                }))
                .child("Menu")
        );

        // 弹出菜单
        if self.open {
            let menu = anchored()
                .anchor(Anchor::TopLeft)
                .position(point(px(0.0), px(28.0)))
                .snap_to_window_with_margin(px(4.0))
                .child(
                    div()
                        .w_48()
                        .rounded_md()
                        .shadow_md()
                        .bg_white()
                        .child(div().p_2().child("Option 1"))
                        .child(div().p_2().child("Option 2"))
                        .child(div().p_2().child("Option 3"))
                );

            elements.push(menu.into_any_element());
        }

        div().relative().children(elements)
    }
}
```

### Tooltip

```rust
fn tooltip(text: &str) -> impl IntoElement {
    anchored()
        .anchor(Anchor::TopLeft)
        .offset(point(px(0.0), px(-4.0)))
        .snap_to_window_with_margin(px(4.0))
        .child(
            div()
                .px_2()
                .py_1()
                .rounded_sm()
                .bg(rgb(0x333333))
                .text_white()
                .text_sm()
                .child(text)
        )
}
```

### Popover

```rust
fn popover(content: impl IntoElement) -> impl IntoElement {
    anchored()
        .anchor(Anchor::TopLeft)
        .position_mode(AnchoredPositionMode::Window)
        .snap_to_window_with_margin(px(8.0))
        .child(
            div()
                .w_72()
                .rounded_lg()
                .shadow_lg()
                .bg_white()
                .p_4()
                .child(content)
        )
}
```

Anchored 的关键规则：
- `anchored()` 创建自动避边界的弹出层
- `anchor()` 设置对齐方向
- `position()` 设置锚点坐标
- `offset()` 添加额外偏移
- `snap_to_window()` / `SwitchAnchor` 控制溢出处理
- 通常配合 `deferred()` 使用，确保弹出层绘制在其他内容之上
