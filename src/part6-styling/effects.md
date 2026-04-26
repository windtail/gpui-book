# 第24章：边框、阴影与变换

## 24.1 边框

### 24.1.1 `border_1()` 到 `border_4()` 快捷方法

GPUI 提供边框宽度的快捷方法：

```rust
div()
    .border_0()  // 无边框
    .border_1()  // 1px
    .border_2()  // 2px
    .border_3()  // 3px
    .border_4()  // 4px
    .child("带边框的元素")
```

使用宏生成的 `border_width()` 设置任意宽度：

```rust
div()
    .border_width(px(5.))  // 自定义宽度
    .child("5px 边框")
```

### 24.1.2 边框颜色

```rust
div()
    .border_1()
    .border_color(red)           // 命名颜色
    .border_color(rgb(0xcc0000)) // RGB
    .border_color(cx.theme().border)  // 主题颜色
    .child("红色边框")
```

### 24.1.3 边框样式

```rust
// 实线边框（默认）
div().border_1().border_solid().child("实线")

// 虚线边框
div().border_1().border_dashed().child("虚线")

// 点线边框
div().border_1().border_dotted().child("点线")
```

### 24.1.4 单边边框

为各边单独设置边框：

```rust
// 仅底部边框（常用于分割线效果）
div()
    .border_b_1()
    .border_color(cx.theme().border)
    .p_4()
    .child("底部有边框的元素")

// 仅左侧边框（常用于选中指示器）
div()
    .border_l_2()
    .border_color(red)
    .pl_2()
    .child("左侧红色指示线")

// 组合使用
div()
    .border_t_1()    // 上边框
    .border_b_1()    // 下边框
    .child("只有上下边框")
```

## 24.2 圆角

### 24.2.1 `rounded_*` 预设值

```rust
div()
    .rounded_none()   // 无圆角
    .rounded_sm()     // 小圆角 2px
    .rounded_md()     // 中等圆角 4px
    .rounded_lg()     // 大圆角 8px
    .rounded_xl()     // 超大圆角 12px
    .rounded_2xl()    // 24px
    .rounded_3xl()    // 32px
    .rounded_full()   // 完全圆形（等于元素最小尺寸的一半）
    .child("圆角元素")
```

### 24.2.2 `Corners<T>` 自定义圆角

使用 `rounded_*` 快捷方法无法满足需求时，直接设置 `Corners`：

```rust
use gpui::Corners;

// 只设置左上角
div()
    .rounded(Corners {
        top_left: px(8.),
        top_right: px(0.),
        bottom_left: px(0.),
        bottom_right: px(0.),
    })
    .child("仅左上角圆角")

// 上方圆角，下方直角（卡片风格）
div()
    .rounded(Corners {
        top_left: px(8.),
        top_right: px(8.),
        bottom_left: px(0.),
        bottom_right: px(0.),
    })
    .child("上圆下方")
```

单边圆角快捷方法：

```rust
div()
    .rounded_t_lg()  // 顶部两角圆角
    .rounded_b_lg()  // 底部两角圆角
    .rounded_l_lg()  // 左侧两角圆角
    .rounded_r_lg()  // 右侧两角圆角
    .rounded_tl_lg() // 仅左上角
    .rounded_tr_lg() // 仅右上角
    .rounded_bl_lg() // 仅左下角
    .rounded_br_lg() // 仅右下角
```

## 24.3 阴影

### 24.3.1 `shadow_*` 预设

```rust
div()
    .shadow_sm()   // 小阴影
    .shadow_md()   // 中等阴影
    .shadow_lg()   // 大阴影
    .shadow_xl()   // 超大阴影
    .child("带阴影的元素")
```

各预设的差异在于偏移量、扩散范围和颜色深度。

### 24.3.2 自定义阴影

使用 `shadow()` 方法自定义阴影参数：

```rust
use gpui::{BoxShadow, point};

div()
    .shadow(vec![BoxShadow {
        blur_radius: px(10.),
        spread_radius: px(0.),
        offset: point(px(0.), px(4.)),
        color: rgba(0x00000020),
    }])
    .child("自定义阴影")
```

多阴影叠加：

```rust
div()
    .shadow(vec![
        BoxShadow {
            blur_radius: px(4.),
            spread_radius: px(0.),
            offset: point(px(0.), px(1.)),
            color: rgba(0x00000010),
        },
        BoxShadow {
            blur_radius: px(20.),
            spread_radius: px(0.),
            offset: point(px(0.), px(8.)),
            color: rgba(0x00000008),
        },
    ])
    .child("双层阴影效果")
```

## 24.4 透明度

### 24.4.1 `opacity()` 方法

`opacity()` 设置元素及其子元素的不透明度，值范围 `0.0`（完全透明）到 `1.0`（完全不透明）：

```rust
div()
    .opacity(0.5)  // 半透明
    .child("半透明内容")

// 禁用状态常用模式
div()
    .when(disabled, |this| {
        this.opacity(0.4)
    })
    .child("禁用时变淡")
```

### 24.4.2 性能影响

`opacity()` 创建一个新的合成层，在 GPU 上混合。大量使用 `opacity` 的元素会增加合成开销。

替代方案：在颜色中设置 alpha 值：

```rust
// 使用 opacity（创建合成层）
div().opacity(0.5).bg(red)

// 使用带 alpha 的颜色（不创建合成层，性能更好）
div().bg(rgba(0xff000080))
```

对于静态元素，优先使用带 alpha 的颜色。对于需要动画的元素（如淡入淡出），使用 `opacity()`。

## 24.5 变换

### 24.5.1 旋转

使用 `rotate()` 设置旋转变换：

```rust
use gpui::Radians;

div()
    .rotate(Radians(std::f32::consts::PI / 4.))  // 45 度
    .child("旋转45度")

// 常用角度
div().rotate(Radians(0.))           // 0 度（无旋转）
div().rotate(Radians(std::f32::consts::PI / 2.))  // 90 度
div().rotate(Radians(std::f32::consts::PI))       // 180 度
```

### 24.5.2 缩放

使用 `TransformationMatrix` 进行缩放：

```rust
use gpui::TransformationMatrix;

div()
    .transform(TransformationMatrix::scale(1.5, 1.5))  // 放大 1.5 倍
    .child("放大")

div()
    .transform(TransformationMatrix::scale(0.8, 0.8))  // 缩小到 80%
    .child("缩小")
```

### 24.5.3 平移

使用 `TransformationMatrix` 进行平移：

```rust
use gpui::{TransformationMatrix, point};

div()
    .transform(TransformationMatrix::translate(point(px(10.), px(20.))))
    .child("向右10px，向下20px")
```

### 24.5.4 `TransformationMatrix` 组合变换

`TransformationMatrix` 支持组合多种变换：

```rust
use gpui::{TransformationMatrix, point, Radians};

// 组合变换：先平移，再旋转，最后缩放
let transform = TransformationMatrix::unit()
    .then_translate(point(px(50.), px(50.)))
    .then_rotate(Radians(std::f32::consts::PI / 6.))  // 30度
    .then_scale(1.2, 1.2);

div()
    .transform(transform)
    .child("组合变换")
```

变换方法：

```rust
TransformationMatrix::unit()                        // 单位矩阵
TransformationMatrix::translate(point)               // 平移
TransformationMatrix::rotate(radians)                // 旋转
TransformationMatrix::scale(sx, sy)                  // 缩放

// 链式组合
matrix.then_translate(point)
matrix.then_rotate(radians)
matrix.then_scale(sx, sy)
```

### 24.5.5 变换与动画

变换可以与动画系统集成，实现平滑的过渡效果：

```rust
use gpui::{Animation, AnimationExt, ease_in_out};

div()
    .with_animation(
        "rotate",
        Animation::new().duration(std::time::Duration::from_secs(2)),
        |this, progress| {
            let angle = Radians(progress * std::f32::consts::PI * 2.0);
            this.transform(TransformationMatrix::rotate(angle))
        },
    )
    .child("旋转动画")
```
