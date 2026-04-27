# 第20章：尺寸与间距

## 20.1 像素单位系统

### 20.1.1 `Pixels` —— 逻辑像素

`Pixels` 是 GPUI 中最常用的尺寸单位。它表示逻辑像素（与设备无关的像素），在不同 DPI 的屏幕上保持一致的物理大小。

```rust
use gpui::{Pixels, px};

// 创建方式
let size: Pixels = px(16.0);      // 函数构造
let size: Pixels = Pixels(16.0);  // 直接构造

// 算术运算
let doubled = px(16.0) * 2.0;     // Pixels(32.0)
let half = px(16.0) / 2.0;        // Pixels(8.0)
let diff = px(32.0) - px(16.0);   // Pixels(16.0)
```

在样式中使用：

```rust
div()
    .w(px(300.))     // 宽度 300 逻辑像素
    .h(px(200.))     // 高度 200 逻辑像素
    .size(px(100.))  // 宽高均为 100 逻辑像素
```

### 20.1.2 `ScaledPixels` —— 物理像素（考虑 DPR）

`ScaledPixels` 表示经过设备像素比（DPR）缩放后的物理像素。主要用于底层渲染计算。

```rust
use gpui::{Pixels, ScaledPixels};

let logical = px(16.0);
let scale_factor = 2.0;  // Retina 屏幕
let physical: ScaledPixels = logical.scale(scale_factor);
// physical = ScaledPixels(32.0) —— 在 2x DPR 屏幕上占 32 个物理像素
```

日常开发中很少直接使用 `ScaledPixels`。GPUI 内部在布局计算和 GPU 渲染管线中自动处理转换。

### 20.1.3 `DevicePixels` —— 设备像素（整数）

`DevicePixels` 表示整数形式的设备像素，用于 GPU 纹理和缓冲区操作。

```rust
use gpui::DevicePixels;

let dp = DevicePixels(128);
// 通常用于纹理尺寸、图片分辨率等
```

### 20.1.4 `px()`、`sp()` 转换

`px()` 是最常用的构造函数。GPUI 不提供 `sp()` 函数（`sp` 是 Android 的缩放像素单位），但可以使用 `rems()` 实现类似的字体缩放效果。

```rust
// px() 用于精确控制
div().w(px(16.))

// rems() 用于跟随系统字体设置（见 20.2 节）
div().text_size(rems(1.0))
```

## 20.2 相对单位

### 20.2.1 `Rems` —— 根 em 单位

`Rems` 基于根元素字体大小（通常为 16px）进行缩放。适合用于与文本相关的尺寸。

```rust
use gpui::{Rems, rems};

let one_rem: Rems = rems(1.0);   // 通常 = 16px
let half_rem: Rems = rems(0.5);  // 通常 = 8px
let two_rem: Rems = rems(2.0);   // 通常 = 32px
```

### 20.2.2 `rems()` 构造函数

`rems()` 创建 `Rems` 值，在文本样式中广泛使用。

```rust
div()
    .text_size(rems(1.5))    // 1.5rem = 24px（假设根字体 16px）
    .line_height(rems(1.75)) // 行高 28px
```

### 20.2.3 与系统字体大小的关系

`Rems` 的值基于 GPUI 的根字体大小。如果系统或用户改变了根字体大小，所有使用 `rems()` 的尺寸会自动跟随缩放。

## 20.3 Length 类型

### 20.3.1 `Absolute` vs `Relative` vs `Auto`

`Length` 是 GPUI 中表示长度值的枚举：

```rust
enum Length {
    Definite(DefiniteLength),  // 确定长度：绝对值或分数
    Auto,                       // 自动尺寸
}

enum DefiniteLength {
    Absolute(AbsoluteLength),  // 像素、rems 等
    Fraction(f32),             // 百分比/分数
}

enum AbsoluteLength {
    Pixels(Pixels),            // 逻辑像素
    Rems(Rems),                // 根 em 单位
}
```

### 20.3.2 `relative()` 分数长度

`relative(f32)` 创建分数形式的 `DefiniteLength`，值 0.0~1.0 表示百分比。

```rust
use gpui::{relative, Length};

div().w(relative(0.5))   // 宽度 = 父容器宽度的 50%
div().w(relative(1.0))   // 宽度 = 父容器宽度的 100%
div().h(relative(0.25))  // 高度 = 父容器高度的 25%
```

`relative()` 常用于创建响应式布局：

```rust
div()
    .w_full()            // 宽度 100%
    .child(div().w(relative(0.5)).child("占一半宽度"))
```

### 20.3.3 `auto()` 自动尺寸

`auto()` 创建 `Length::Auto`，让布局引擎根据内容自动决定尺寸。

```rust
use gpui::auto;

div().w(auto())  // 宽度由内容决定
```

## 20.4 Margin 与 Padding

### 20.4.1 `m()`、`mt()`、`mr()`、`mb()`、`ml()`

Margin（外边距）控制元素与相邻元素的间距。

```rust
// 四边统一 margin
div().m(px(16.))      // 四个方向都是 16px
div().m_0()           // 0px
div().m_1()           // 4px
div().m_2()           // 8px
div().m_3()           // 12px
div().m_4()           // 16px
div().m_auto()        // 自动外边距（常用于居中）

// 单边 margin
div().mt(px(8.))      // margin-top
div().mr(px(16.))     // margin-right
div().mb(px(8.))      // margin-bottom
div().ml(px(16.))     // margin-left
```

### 20.4.2 `p()`、`pt()`、`pr()`、`pb()`、`pl()`

Padding（内边距）控制元素内容与元素边界的间距。

```rust
// 四边统一 padding
div().p(px(16.))      // 四个方向都是 16px
div().p_0()           // 0px
div().p_1()           // 4px
div().p_2()           // 8px
div().p_3()           // 12px
div().p_4()           // 16px
div().p_6()           // 24px
div().p_8()           // 32px
div().p_12()          // 48px
div().p_16()          // 64px

// 单边 padding
div().pt(px(8.))      // padding-top
div().pr(px(16.))     // padding-right
div().pb(px(8.))      // padding-bottom
div().pl(px(16.))     // padding-left
```

### 20.4.3 `mx/my` 与 `px/py` 快捷方法

水平（x）和垂直（y）方向的批量设置：

```rust
// mx = ml + mr
div().mx(px(16.))   // 水平 margin = 16px
div().my(px(8.))    // 垂直 margin = 8px

// px = pl + pr
div().px(px(16.))   // 水平 padding = 16px
div().py(px(8.))    // 垂直 padding = 8px
```

实用示例：

```rust
// 水平居中的卡片
div()
    .flex()
    .justify_center()
    .my(px(40.))
    .child(
        div()
            .mx_auto()        // 水平自动外边距
            .px_6()
            .py_4()
            .rounded_lg()
            .child("卡片内容")
    )
```

### 20.4.4 负边距的可能性

GPUI 支持负边距，用于元素重叠的布局技巧：

```rust
// 负 margin 实现重叠效果
div()
    .flex()
    .gap_2()
    .child(div().ml(px(-8.)).child("向左偏移"))
    .child(div().child("正常位置"))
```

负 margin 的使用场景有限，谨慎使用以避免布局混乱。

## 20.5 溢出处理

### 20.5.1 `overflow_hidden()` 裁剪

`overflow_hidden()` 裁剪超出元素边界的内容。

```rust
// 裁剪超出部分
div()
    .w(px(200.))
    .h(px(100.))
    .overflow_hidden()
    .child(div().w(px(400.)).child("这段文字会被裁剪"))
```

在滚动容器中，`overflow_hidden()` 常与 `ScrollHandle` 配合使用。

### 20.5.2 `overflow_scroll()` 滚动

GPUI 使用 `Overflow::Scroll` 来启用滚动。具体实现依赖 `overflow_x_hidden()` 和 `overflow_y_hidden()` 的组合：

```rust
use gpui::{div, Styled, ParentElement};

// 垂直滚动容器
struct ScrollableList {
    items: Vec<String>,
}

impl Render for ScrollableList {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .w(px(300.))
            .h(px(400.))
            .overflow_y_hidden()  // 垂直方向允许滚动（不隐藏 = 允许溢出）
            .children(
                self.items.iter().map(|item| {
                    div().p_2().border_b_1().child(item)
                })
            )
    }
}
```

### 20.5.3 `content_mask` 内部机制

`overflow_hidden()` 在底层使用 `ContentMask` 实现裁剪。当元素被标记为 `overflow: hidden` 时，GPUI 在 `paint()` 阶段设置裁剪区域：

```rust
// 内部原理（简化）
window.with_content_mask(Some(content_mask), |window| {
    // 在此区域内绘制子元素
    child.paint(window, window);
});
```

`ContentMask` 是一个 `Bounds<Pixels>`，定义了允许绘制的矩形区域。超出此区域的绘制操作被丢弃。

```rust
// 分轴控制溢出
div()
    .overflow_x_hidden()  // 水平裁剪
    // overflow_y 未设置，允许溢出但不裁剪（可滚动）
    .child(div().w(px(500.)).child("超出水平边界的内容会被裁剪"))

div()
    .overflow_y_hidden()  // 垂直裁剪
    // overflow_x 未设置
    .child(div().w(px(500.)).child("水平方向可以滚动"))
```

## 20.6 几何类型速查

### 20.6.1 `Point<T>`、`Size<T>`、`Bounds<T>`

```rust
use gpui::{Point, Size, Bounds, point, size, bounds};

// 点
let p = point(px(10.), px(20.));      // Point<Pixels>
let p = Point::new(px(10.), px(20.));

// 尺寸
let s = size(px(100.), px(200.));     // Size<Pixels>
let s = Size::new(px(100.), px(200.));

// 边界矩形
let b = bounds(point(px(0.), px(0.)), size(px(100.), px(200.)));
let b = Bounds::new(origin, size);
```

### 20.6.2 `Edges<T>` 与 `Corners<T>`

```rust
use gpui::{Edges, Corners};

// Edges —— 四边值（用于 padding、margin、border）
let edges = Edges {
    top: px(8.),
    right: px(16.),
    bottom: px(8.),
    left: px(16.),
};

// Corners —— 四角值（用于圆角）
let corners = Corners {
    top_left: px(4.),
    top_right: px(4.),
    bottom_right: px(4.),
    bottom_left: px(4.),
};
```

### 20.6.3 `Anchor` 枚举

```rust
use gpui::Anchor;

// 八个锚点位置
Anchor::TopLeft      // 左上
Anchor::TopRight     // 右上
Anchor::TopCenter    // 上中
Anchor::BottomLeft   // 左下
Anchor::BottomRight  // 右下
Anchor::BottomCenter // 下中
Anchor::LeftCenter   // 左中
Anchor::RightCenter  // 右中
```

注意：`Anchor` 没有 `Center` 变体——中心点不在枚举中。

### 20.6.4 `Radians` 与 `Percentage`

```rust
use gpui::Radians;

let angle = Radians(std::f32::consts::PI / 4.0);  // 45 度

use gpui::Percentage;
let percent = Percentage(0.5);  // 50%
```
