# 第18章：Flexbox 弹性布局

## 18.1 主轴与交叉轴

### 18.1.1 `flex_row()` vs `flex_col()`

Flexbox 的核心概念是主轴（main axis）和交叉轴（cross axis）。主轴方向由 `flex_direction` 决定，交叉轴始终垂直于主轴。

```rust
use gpui::{div, Styled, ParentElement, IntoElement};

// 水平排列（默认方向）
div()
    .flex()
    .flex_row()
    .gap_2()
    .child(div().size_10().bg(red).child("A"))
    .child(div().size_10().bg(green).child("B"))
    .child(div().size_10().bg(blue).child("C"))
```

`.flex_row()` 将主轴设为水平方向，子元素从左到右排列。这是 GPUI 的默认方向，即使不调用 `.flex_row()`，只要调用了 `.flex()` 就是行布局。

```rust
// 垂直排列
div()
    .flex()
    .flex_col()
    .gap_2()
    .child(div().w_full().h_10().bg(red).child("Header"))
    .child(div().flex_1().bg(green).child("Body"))
    .child(div().w_full().h_10().bg(blue).child("Footer"))
```

`.flex_col()` 将主轴设为垂直方向，子元素从上到下排列。上例中，`flex_1()` 让 Body 区域占据所有剩余空间。

### 18.1.2 反向布局：`flex_row_rev()`、`flex_col_rev()`

反向布局反转主轴方向，但 `justify_*` 方法的对齐方向也会跟随反转。

```rust
// 从右到左排列
div()
    .flex()
    .flex_row_rev()
    .gap_2()
    .child(div().child("First"))  // 视觉上在最右侧
    .child(div().child("Second")) // 视觉上在中间
    .child(div().child("Third"))  // 视觉上在最左侧

// 从下到上排列
div()
    .flex()
    .flex_col_rev()
    .gap_2()
    .child(div().child("Bottom")) // 视觉上在最下方
    .child(div().child("Middle"))
    .child(div().child("Top"))    // 视觉上在最上方
```

反向布局在实现消息列表（最新消息在底部）或从右到左语言支持时很有用。

## 18.2 弹性项目伸缩

### 18.2.1 `flex_grow()` —— 填充剩余空间

`flex_grow` 控制子元素在空间充裕时的放大比例。值为 `f32` 类型。

```rust
// 子元素 B 占剩余空间的 2 倍
div()
    .flex()
    .gap_2()
    .child(div().flex_grow().bg(red).child("A: 1份"))    // grow = 1
    .child(div().flex_grow().flex_grow_0().bg(green))    // grow = 0
    .child(div().flex_grow().bg(blue).child("C: 2份"))   // grow = 2
```

注意：GPUI 的 `flex_grow()` 方法不带参数，设置 grow 为 1。如需其他值，需要在 `.style()` 上直接操作。

```rust
// 直接通过 style 设置 grow 值
div()
    .flex()
    .child(div().flex_grow())  // grow = 1
    .child({
        let mut d = div();
        d.style().flex_grow = Some(2.0);
        d
    })
```

### 18.2.2 `flex_shrink()` —— 空间不足时收缩

`flex_shrink` 控制空间不足时的收缩行为。

```rust
// 当容器宽度不足以容纳所有子元素时：
// A 保持不变，B 和 C 按比例收缩
div()
    .flex()
    .w(px(300.))
    .child(div().w(px(200.)).flex_shrink_0().child("A: 200px, 不收缩"))
    .child(div().w(px(200.)).flex_shrink().child("B: 会收缩"))
    .child(div().w(px(200.)).flex_shrink().child("C: 会收缩"))
```

`flex_shrink_0()` 阻止元素收缩，常用于确保关键元素不被压缩。

### 18.2.3 `flex_basis()` —— 初始尺寸

`flex_basis` 设置弹性项目在分配剩余空间之前的初始大小。

```rust
use gpui::{Length, px, relative};

// 设置固定初始宽度
div()
    .flex()
    .child(div().flex_basis(px(200.)).child("固定200px"))
    .child(div().flex_1().child("填充剩余"))

// 设置百分比初始宽度
div()
    .flex()
    .child(div().flex_basis(relative(0.3)).child("30%"))
    .child(div().flex_basis(relative(0.7)).child("70%"))

// 使用 Auto（基于内容大小）
div()
    .flex()
    .child(div().flex_auto().child("自适应内容宽度"))
```

`flex_basis` 接受 `impl Into<Length>`，可以是 `px()`、`relative()`、`Length::Auto` 等。

### 18.2.4 `flex_1()` 快捷方法

`flex_1()` 是最常用的 Flexbox 快捷方法，等价于：

```rust
// flex_1() 的展开形式：
// grow = 1, shrink = 1, basis = relative(0.0)
div().flex_1()

// 等同于：
div()
    .with(|this| {
        this.style().flex_grow = Some(1.0);
        this.style().flex_shrink = Some(1.0);
        this.style().flex_basis = Some(relative(0.0).into());
    })
```

`flex_1()` 让元素占据所有可用空间，同时允许在空间不足时收缩。

```rust
// 经典侧边栏布局
div()
    .flex()
    .size_full()
    .child(div().w(px(250.)).flex_shrink_0().child("侧边栏"))
    .child(div().flex_1().child("主内容区域"))
```

## 18.3 对齐与分布

### 18.3.1 主轴对齐：`justify_*`

`justify_content` 控制子元素沿主轴的分布方式。

```rust
// justify_start —— 从主轴起点开始排列（默认行为）
div().flex().justify_start()
     .child("A").child("B").child("C")

// justify_center —— 居中对齐
div().flex().justify_center()
     .child("A").child("B").child("C")

// justify_end —— 从主轴终点开始排列
div().flex().justify_end()
     .child("A").child("B").child("C")

// justify_between —— 两端对齐，元素之间等距
div().flex().justify_between()
     .child("左").child("中").child("右")
// 输出：[左]      [中]      [右]

// justify_around —— 每个元素两侧空间相等
div().flex().justify_around()
     .child("A").child("B").child("C")
// 输出： [A]  [B]  [C]

// justify_evenly —— 所有间距完全相等
div().flex().justify_evenly()
     .child("A").child("B").child("C")
// 输出：  [A]  [B]  [C]
```

### 18.3.2 交叉轴对齐：`items_*`

`align_items` 控制子元素沿交叉轴的对齐方式。

```rust
// items_start —— 对齐到交叉轴起点
div()
    .flex()
    .h(px(200.))
    .items_start()
    .child(div().h(px(50.)).child("贴顶部"))

// items_center —— 交叉轴居中
div()
    .flex()
    .h(px(200.))
    .items_center()
    .child(div().h(px(50.)).child("垂直居中"))

// items_end —— 对齐到交叉轴终点
div()
    .flex()
    .h(px(200.))
    .items_end()
    .child(div().h(px(50.)).child("贴底部"))

// items_stretch —— 拉伸以填满交叉轴（默认）
div()
    .flex()
    .h(px(200.))
    .items_stretch()
    .child(div().child("高度自动填满200px"))

// items_baseline —— 按文本基线对齐
div()
    .flex()
    .items_baseline()
    .child(div().text_2xl().child("大字"))
    .child(div().text_xs().child("小字"))
```

### 18.3.3 单项目对齐：`self_*`

`align_self` 允许单个子元素覆盖父容器的 `align_items` 设置。

```rust
div()
    .flex()
    .h(px(200.))
    .items_center()  // 默认居中
    .child(div().h(px(40.)).child("居中"))
    .child(
        div()
            .h(px(40.))
            .self_start()  // 覆盖为顶部
            .child("贴顶")
    )
    .child(
        div()
            .h(px(40.))
            .self_end()  // 覆盖为底部
            .child("贴底")
    )
    .child(div().h(px(40.)).child("居中"))
```

可用的 `self_*` 方法：`self_start()`、`self_end()`、`self_center()`、`self_stretch()`、`self_flex_start()`、`self_flex_end()`、`self_baseline()`。

## 18.4 换行

### 18.4.1 `flex_wrap()` 与 `flex_nowrap()`

默认情况下，Flexbox 不换行（`flex_nowrap()`）。当子元素总宽度超过容器宽度时，它们会被压缩或溢出。

```rust
// flex_wrap —— 允许换行
div()
    .flex()
    .flex_wrap()
    .gap_2()
    .w(px(300.))
    .child(div().w(px(120.)).h_10().bg(red).child("1"))
    .child(div().w(px(120.)).h_10().bg(green).child("2"))
    .child(div().w(px(120.)).h_10().bg(blue).child("3"))
// 第三项会换到第二行

// flex_nowrap —— 不换行（默认）
div()
    .flex()
    .flex_nowrap()
    .w(px(300.))
    .child(div().w(px(150.)).child("A"))
    .child(div().w(px(150.)).child("B"))
    .child(div().w(px(150.)).child("C"))
// 所有项强制在一行，可能被压缩
```

### 18.4.2 `flex_wrap_reverse`

反转交叉轴方向的同时进行换行。

```rust
div()
    .flex()
    .flex_wrap_reverse()
    .gap_2()
    .child(div().child("第一行"))
    .child(div().child("第二行"))
    .child(div().child("第三行"))
```

`flex_wrap_reverse` 较少使用，主要用于需要反转换行方向的特殊布局。

## 18.5 间距

### 18.5.1 `gap()` 统一间距

`gap()` 在 Flexbox 子元素之间添加统一间距。

```rust
div()
    .flex()
    .gap_2()       // gap = 8px (2 * 4px 单位)
    .child(div().child("A"))
    .child(div().child("B"))
    .child(div().child("C"))
// 输出：[A] 8px [B] 8px [C]
// 注意：首尾没有额外的 gap
```

`gap()` 接受 `impl Into<AbsoluteLength>`：

```rust
div().flex().gap(px(16.)).child("A").child("B")
div().flex().gap_0()   // gap = 0
div().flex().gap_1()   // gap = 4px
div().flex().gap_2()   // gap = 8px
div().flex().gap_3()   // gap = 12px
div().flex().gap_4()   // gap = 16px
div().flex().gap_6()   // gap = 24px
div().flex().gap_8()   // gap = 32px
```

### 18.5.2 `col_gap()` 与 `row_gap()`

分别控制主轴和交叉轴方向的间距。

```rust
div()
    .flex()
    .flex_wrap()
    .col_gap_4()  // 列间距 16px
    .row_gap_2()  // 行间距 8px
    .child(div().child("1"))
    .child(div().child("2"))
    .child(div().child("3"))
    .child(div().child("4"))
```

## 18.6 实战：常见布局模式

### 18.6.1 经典三栏布局

```rust
struct ThreeColumnLayout {
    sidebar_left: String,
    main_content: String,
    sidebar_right: String,
}

impl Render for ThreeColumnLayout {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .size_full()
            .child(
                div()
                    .w(px(240.))
                    .flex_shrink_0()
                    .border_r_1()
                    .border_color(cx.theme().border)
                    .p_4()
                    .child(&self.sidebar_left)
            )
            .child(
                div()
                    .flex_1()
                    .p_4()
                    .child(&self.main_content)
            )
            .child(
                div()
                    .w(px(200.))
                    .flex_shrink_0()
                    .border_l_1()
                    .border_color(cx.theme().border)
                    .p_4()
                    .child(&self.sidebar_right)
            )
    }
}
```

### 18.6.2 居中卡片

```rust
div()
    .flex()
    .items_center()
    .justify_center()
    .size_full()
    .bg(cx.theme().background)
    .child(
        div()
            .flex()
            .flex_col()
            .gap_4()
            .p_6()
            .rounded_lg()
            .bg(cx.theme().container)
            .shadow_md()
            .w(px(400.))
            .child(div().text_xl().font_weight(gpui::FontWeight::BOLD).child("标题"))
            .child(div().text_sm().child("卡片内容描述"))
            .child(
                div()
                    .flex()
                    .justify_end()
                    .gap_2()
                    .child(div().child("取消"))
                    .child(div().child("确认"))
            )
    )
```

### 18.6.3 自适应表单布局

```rust
struct FormLayout {
    label: String,
    error: Option<String>,
}

impl Render for FormLayout {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .gap_1()
            .w_full()
            .child(
                div()
                    .flex()
                    .flex_col()
                    .gap_1()
                    .child(div().text_sm().font_weight(gpui::FontWeight::SEMIBOLD).child(&self.label))
                    .child(div()
                        .w_full()
                        .h(px(36.))
                        .px_3()
                        .rounded_md()
                        .border_1()
                        .border_color(cx.theme().border)
                    )
            )
            .when_some(&self.error, |this, error| {
                this.child(div().text_xs().text_color(red).child(error))
            })
    }
}
```
