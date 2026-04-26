# 第22章：Styled Trait

## 22.1 `Styled` trait 概览

### 22.1.1 Tailwind-like 的 Rust 实现

`Styled` trait 为 GPUI 元素提供类 Tailwind CSS 的工具方法链式 API：

```rust
use gpui::{div, Styled, ParentElement, IntoElement};

// Tailwind: class="flex flex-col gap-2 p-4 rounded-lg bg-gray-100"
// GPUI:
div()
    .flex()
    .flex_col()
    .gap_2()
    .p_4()
    .rounded_lg()
    .bg(gray_100)
    .child("内容")
```

每个样式方法直接修改元素内部的 `StyleRefinement`，返回 `Self` 以支持链式调用。

### 22.1.2 `style() -> &mut StyleRefinement`

`Styled` trait 只有一个必需方法：

```rust
pub trait Styled: Sized {
    fn style(&mut self) -> &mut StyleRefinement;
    // 其余方法通过宏自动生成
}
```

`StyleRefinement` 采用增量模式：每个字段都是 `Option`，未设置的字段保持 `None`，最终与父元素的样式合并。

```rust
// StyleRefinement 的部分字段（简化）
pub struct StyleRefinement {
    pub display: Option<Display>,
    pub flex_direction: Option<FlexDirection>,
    pub flex_grow: Option<f32>,
    pub flex_shrink: Option<f32>,
    pub flex_basis: Option<Length>,
    pub align_items: Option<AlignItems>,
    pub justify_content: Option<JustifyContent>,
    pub background: Option<Fill>,
    pub opacity: Option<f32>,
    pub text: Option<TextStyleRefinement>,
    // ...
}
```

`Div` 实现了 `Styled` trait：

```rust
impl Styled for Div {
    fn style(&mut self) -> &mut StyleRefinement {
        &mut self.interactivity.style
    }
}
```

### 22.1.3 方法链与 fluent API

所有样式方法都返回 `Self`，支持无限链式调用：

```rust
div()
    .flex()
    .items_center()
    .justify_between()
    .p_4()
    .gap_2()
    .rounded_md()
    .border_1()
    .border_color(cx.theme().border)
    .bg(cx.theme().background)
    .child(div().child("左侧"))
    .child(div().child("右侧"))
```

方法调用顺序不影响最终样式，因为所有设置都写入同一个 `StyleRefinement` 结构。

## 22.2 状态驱动的样式

### 22.2.1 hover 状态

通过 `InteractiveElement` 扩展，`Div` 支持 hover 状态样式：

```rust
use gpui::{div, Styled, InteractiveElement, IntoElement};

div()
    .p_3()
    .rounded_md()
    .bg(cx.theme().background)
    .hover(|this| {
        this.bg(cx.theme().container)
            .cursor_pointer()
    })
    .child("悬停时变色")
```

`hover()` 接收一个闭包，在 hover 状态下应用闭包返回的样式。

### 22.2.2 active 状态

`active` 对应鼠标按下的状态：

```rust
div()
    .p_3()
    .rounded_md()
    .bg(cx.theme().background)
    .active(|this| {
        this.bg(cx.theme().selected)
    })
    .on_mouse_down(MouseButton::Left, |_, _, _| { /* 点击处理 */ })
    .child("按下时变色")
```

### 22.2.3 focus 状态

`focus` 对应元素获得焦点时的样式：

```rust
div()
    .id("input-wrapper")  // 需要 id 来追踪焦点状态
    .p_2()
    .rounded_md()
    .border_1()
    .border_color(cx.theme().border)
    .focus(|this| {
        this.border_color(cx.theme().selected)
    })
    .child(text_input.clone())
```

需要配合 `id()` 使用，因为焦点状态需要跨帧追踪。

### 22.2.4 selected 状态

`selected` 用于列表项等被选中的场景：

```rust
div()
    .p_2()
    .rounded_md()
    .when(selected, |this| {
        this.bg(cx.theme().selected)
            .text_color(cx.theme().selected_text)
    })
    .child(&item_text)
```

### 22.2.5 disabled 状态

`disabled` 用于不可交互的元素：

```rust
div()
    .p_3()
    .rounded_md()
    .when(disabled, |this| {
        this.opacity(0.5).cursor_not_allowed()
    })
    .child("禁用状态的按钮")
```

## 22.3 分组样式

### 22.3.1 `id()` 与元素标识

`id()` 为元素赋予唯一标识，用于状态追踪和样式引用：

```rust
div()
    .id("header")          // 固定字符串 id
    .p_4()
    .on_click(|_, _, _| { /* 点击事件 */ })

// 使用 ElementId 枚举
div()
    .id(ElementId::Name("header".into()))
    .id(ElementId::View(0))  // 数值 id
```

需要 `id()` 的场景：
- 焦点追踪（`focus()` 样式）
- hover 状态跨帧持久化
- 事件注册

### 22.3.2 `group()` 与组选择器

`group()` 创建命名组，子元素可以引用父组的状态：

```rust
div()
    .id("card-group")
    .group("card")
    .hover(|this| {
        // 当整个卡片 hover 时
        this.child(
            div()
                .group_hover("card", |this| {
                    this.visible()  // 子元素在父 hover 时显示
                })
        )
    })
```

`group_hover()` 允许子元素响应父组的 hover 状态，类似于 CSS 的 `.group:hover .child` 选择器。

### 22.3.3 跨组联动

```rust
// 菜单项：鼠标悬停在父项时显示子操作按钮
div()
    .id("menu-item")
    .group("item")
    .flex()
    .items_center()
    .justify_between()
    .p_2()
    .child(div().child("菜单文字"))
    .child(
        div()
            .invisible()  // 默认隐藏
            .group_hover("item", |this| {
                this.visible()  // 父项 hover 时显示
            })
            .child("...")
    )
```

## 22.4 条件样式

### 22.4.1 `.when()` 方法

`.when()` 根据布尔条件应用样式：

```rust
div()
    .p_4()
    .rounded_md()
    .when(is_selected, |this| {
        this.bg(cx.theme().selected)
    })
    .when(is_hovered, |this| {
        this.bg(cx.theme().container)
    })
    .when(is_active_task, |this| {
        this.border_2()
            .border_color(cx.theme().selected)
    })
    .child(&task_name)
```

链式调用多个 `.when()` 可以组合多个条件。

### 22.4.2 `.when_some()` 方法

`.when_some()` 在 `Option` 有值时应用样式：

```rust
div()
    .p_4()
    .when_some(border_width, |this, w| {
        this.border_1().border_color(cx.theme().border)
    })
    .when_some(custom_bg, |this, color| {
        this.bg(color)
    })
    .child("内容")
```

## 22.5 自定义样式扩展

### 22.5.1 扩展 `StyleRefinement`

通过实现 `Styled` trait，可以为自定义元素添加样式能力：

```rust
struct CustomElement {
    style: StyleRefinement,
    // 其他字段
}

impl Styled for CustomElement {
    fn style(&mut self) -> &mut StyleRefinement {
        &mut self.style
    }
}

// 现在可以使用所有 Styled 方法
let element = CustomElement {
    style: StyleRefinement::default(),
};

element
    .flex()
    .p_4()
    .bg(red);
```

### 22.5.2 自定义状态

`StyleRefinement` 的增量设计允许你通过扩展来实现自定义状态样式。例如，为特定的业务场景添加状态：

```rust
// 自定义样式方法扩展
trait TaskStyle: Styled {
    fn task_priority(self, priority: Priority) -> Self;
}

impl TaskStyle for Div {
    fn task_priority(self, priority: Priority) -> Self {
        match priority {
            Priority::High => self.border_l_2().border_color(red),
            Priority::Medium => self.border_l_2().border_color(yellow),
            Priority::Low => self,
        }
    }
}

// 使用
div()
    .p_3()
    .task_priority(task.priority)
    .child(&task.title)
```

通过 trait 扩展，可以在不修改 GPUI 源码的情况下添加领域特定的样式方法。
