# 第19章：Grid 网格布局

## 19.1 网格基础

### 19.1.1 `grid()` 激活网格

Grid 布局使用二维网格系统，同时控制行和列。与 Flexbox 的一维布局不同，Grid 适合需要对齐到网格线的场景。

```rust
use gpui::{div, Styled, ParentElement, IntoElement};

// 最简单的网格：2行2列
div()
    .grid()
    .grid_cols(2)
    .grid_rows(2)
    .gap_2()
    .child(div().bg(red).p_4().child("1"))
    .child(div().bg(green).p_4().child("2"))
    .child(div().bg(blue).p_4().child("3"))
    .child(div().bg(yellow).p_4().child("4"))
```

`.grid()` 设置 `display: Grid`，激活网格布局。`.grid_cols(n)` 和 `.grid_rows(n)` 定义轨道数量。

### 19.1.2 `grid_cols()` 与 `grid_rows()` 定义轨道

轨道定义可以使用多种尺寸单位：

```rust
use gpui::{div, Styled, ParentElement};

// 等分列 —— 三列等宽
div()
    .grid()
    .grid_cols(3)
    .child(div().child("列1"))
    .child(div().child("列2"))
    .child(div().child("列3"))

// 固定宽度列
div()
    .grid()
    .grid_cols(3)
    .child(div().w(px(200.)).child("200px"))
    .child(div().w(px(300.)).child("300px"))
    .child(div().flex_1().child("剩余空间"))

// 混合行高
div()
    .grid()
    .grid_rows(3)
    .child(div().h(px(60.)).child("Header"))
    .child(div().flex_1().child("Body - 自适应"))
    .child(div().h(px(40.)).child("Footer"))
```

### 19.1.3 隐式网格与显式网格

当子元素数量超过定义的轨道数时，Grid 会创建隐式轨道。

```rust
// 定义了 2 列，但放入 5 个子元素
div()
    .grid()
    .grid_cols(2)
    .gap_2()
    .child(div().child("1"))  // 第1行第1列
    .child(div().child("2"))  // 第1行第2列
    .child(div().child("3"))  // 第2行第1列（隐式行）
    .child(div().child("4"))  // 第2行第2列（隐式行）
    .child(div().child("5"))  // 第3行第1列（隐式行）
```

GPUI 的 Grid 默认为隐式行自动分配空间。使用 `grid_rows()` 可以显式控制行高。

## 19.2 网格放置

### 19.2.1 `col_start()`、`col_end()`、`col_span()`

控制单个子元素在网格中的位置。

```rust
// col_span —— 跨越多列
div()
    .grid()
    .grid_cols(3)
    .gap_2()
    .child(div().col_span(2).bg(red).child("跨2列"))  // 占据第1-2列
    .child(div().bg(green).child("普通"))              // 占据第3列
    .child(div().bg(blue).child("第2行第1列"))
    .child(div().bg(yellow).child("第2行第2列"))

// col_span_full —— 跨越所有列
div()
    .grid()
    .grid_cols(3)
    .gap_2()
    .child(div().col_span_full().bg(red).child("整行标题"))
    .child(div().bg(green).child("1"))
    .child(div().bg(blue).child("2"))
    .child(div().bg(yellow).child("3"))

// col_start / col_end —— 精确控制
div()
    .grid()
    .grid_cols(4)
    .gap_2()
    .child(div().col_start(2).col_end(4).child("从第2列开始到第4列"))
```

`col_start(2)` 表示从第 2 条网格线开始。`col_end(4)` 表示在第 4 条网格线结束。

### 19.2.2 `row_start()`、`row_end()`、`row_span()`

行方向的控制方法与列完全对称。

```rust
// row_span —— 跨越多行
div()
    .grid()
    .grid_cols(3)
    .grid_rows(3)
    .gap_2()
    .child(div().row_span(2).bg(red).child("跨2行"))
    .child(div().bg(green).child("普通"))
    .child(div().bg(blue).child("普通"))
    .child(div().bg(yellow).child("第2行"))
    .child(div().bg(purple).child("第3行"))

// row_span_full —— 跨越所有行
div()
    .grid()
    .grid_cols(3)
    .gap_2()
    .child(
        div()
            .row_span_full()
            .bg(red)
            .child("侧边栏（跨满所有行）")
    )
    .child(div().bg(green).child("主内容1"))
    .child(div().bg(blue).child("主内容2"))
```

## 19.3 内容尺寸

### 19.3.1 `min-content` 与 `max-content`

`grid_cols_min_content()` 和 `grid_cols_max_content()` 提供基于内容大小的轨道尺寸。

```rust
// min-content —— 轨道最小化到能容纳内容
div()
    .grid()
    .grid_cols_min_content(2)
    .gap_2()
    .child(div().child("短文本"))
    .child(div().child("这是一段很长的文本，轨道会扩展以容纳它"))
    .child(div().child("另一列"))

// max-content —— 轨道扩展到能容纳内容的最大尺寸
div()
    .grid()
    .grid_cols_max_content(2)
    .gap_2()
    .child(div().child("短"))
    .child(div().child("长文本"))
```

`min-content` 让轨道收缩到内容所需的最小宽度。`max-content` 让轨道扩展到内容所需的最大宽度。

### 19.3.2 `col_span` 跨列

`col_span(n)` 让子元素跨越多个网格列。GPUI 的 `grid_cols(n)` 创建 `n` 个等宽列，配合 `col_span` 实现不同比例分配：

```rust
// 不等的列宽比例：1:2:1（共4列，中间占2列）
div()
    .grid()
    .grid_cols(4)
    .gap_2()
    .child(
        div()
            .col_span(1)
            .bg(rgb(0xef4444))
            .child("25%")
    )
    .child(
        div()
            .col_span(2)
            .bg(rgb(0x22c55e))
            .child("50%")
    )
    .child(
        div()
            .col_span(1)
            .bg(rgb(0x3b82f6))
            .child("25%")
    )
```

同理 `row_span(n)` 让子元素跨越多行。注意：GPUI 的 `grid_cols` / `grid_rows` 只接受列/行数（`u16`），不支持类似 CSS `grid-template-columns: 1fr 2fr 1fr` 的精细轨道控制——需要不等比例时，应通过增加列数配合 `col_span` 实现。

## 19.4 Grid vs Flex

### 19.4.1 何时选 Grid

Grid 适合二维布局，需要同时控制行和列的对齐：

```rust
// 仪表盘布局：需要精确的网格对齐
div()
    .grid()
    .grid_cols(4)
    .grid_rows(3)
    .gap_4()
    .child(div().col_span(2).row_span(2).child("大图表"))   // 2x2 区域
    .child(div().col_span(2).child("统计指标"))             // 1x2 区域
    .child(div().child("指标1"))
    .child(div().child("指标2"))
    .child(div().col_span(4).child("底部数据表格"))          // 全宽行
```

选择 Grid 的场景：
- 需要二维对齐（行和列同时对齐）
- 需要元素跨越多个单元格
- 需要精确的网格线定位
- 布局需要适应不同屏幕尺寸时保持一致的网格结构

### 19.4.2 何时选 Flex

Flexbox 适合一维布局，元素沿单一方向排列：

```rust
// 工具栏：只需要水平排列
div()
    .flex()
    .items_center()
    .justify_between()
    .gap_2()
    .p_2()
    .child(div().child("Logo"))
    .child(
        div().flex().gap_2().child("Nav1").child("Nav2").child("Nav3")
    )
    .child(div().child("用户头像"))
```

选择 Flex 的场景：
- 元素沿一个方向排列
- 需要动态分配剩余空间（`flex_1()`）
- 不需要严格的网格线对齐
- 子元素数量不确定

### 19.4.3 嵌套使用

实际项目中，Grid 和 Flex 通常嵌套使用：

```rust
// 外层 Grid 定义页面结构，内层 Flex 处理组件内部布局
div()
    .grid()
    .grid_cols(12)
    .gap_4()
    .p_4()
    .size_full()
    // 导航栏占满整行
    .child(
        div()
            .col_span_full()
            .flex()            // 内部用 Flex
            .items_center()
            .justify_between()
            .child(div().child("Logo"))
            .child(div().child("导航"))
            .child(div().child("头像"))
    )
    // 侧边栏占 3 列
    .child(
        div()
            .col_span(3)
            .flex()            // 内部用 Flex
            .flex_col()
            .gap_2()
            .child(div().child("菜单1"))
            .child(div().child("菜单2"))
            .child(div().child("菜单3"))
    )
    // 主内容占 9 列
    .child(
        div()
            .col_span(9)
            .child(div().child("主内容"))
    )
```

## 19.5 实战：响应式数据表格

```rust
struct DashboardGrid {
    cards: Vec<DashboardCard>,
}

struct DashboardCard {
    title: String,
    value: String,
    span: u16,
}

impl Render for DashboardGrid {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .grid()
            .grid_cols(4)
            .gap_4()
            .p_4()
            .children(
                self.cards.iter().enumerate().map(|(_, card)| {
                    div()
                        .col_span(card.span)
                        .p_4()
                        .rounded_lg()
                        .border_1()
                        .border_color(cx.theme().border)
                        .bg(cx.theme().background)
                        .child(
                            div()
                                .flex()
                                .flex_col()
                                .gap_2()
                                .child(div().text_sm().text_color(cx.theme().text).child(&card.title))
                                .child(div().text_2xl().font_weight(gpui::FontWeight::BOLD).child(&card.value))
                        )
                })
            )
    }
}
```

每个卡片通过 `span` 字段控制占用的列数，实现灵活的面板布局。
