# 第15章：SVG 元素

## 15.1 `svg()` 基础

### 15.1.1 从内联路径加载

```rust
use gpui::{svg, div, IntoElement, Context, Window, Pixels};

struct IconView {}

impl Render for IconView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .gap_2()
            .child(
                svg()
                    .path("icons/search.svg")  // 嵌入资源
                    .size(Pixels(16.0))
            )
            .child(
                svg()
                    .path("icons/close.svg")
                    .size(Pixels(16.0))
            )
    }
}
```

`svg()` 创建空的 SVG 元素，`.path()` 指定内联 SVG 文件路径。

### 15.1.2 从外部路径加载

```rust
svg()
    .external_path("/home/user/assets/icon.svg")  // 文件系统路径
    .size(Pixels(32.0))
```

`.external_path()` 用于加载文件系统中的 SVG 文件，`.path()` 用于嵌入资源。

```rust
// 可以链式设置样式
svg()
    .path("icons/star.svg")
    .size(Pixels(24.0))
    .text_yellow_500()  // 通过 currentColor 控制 SVG 填充色
```

SVG 元素支持 `InteractiveElement`，可以注册事件：

```rust
svg()
    .id("interactive-icon")
    .path("icons/settings.svg")
    .size(Pixels(20.0))
    .on_click(|_, window, cx| {
        println!("Settings icon clicked");
    })
```

## 15.2 SVG 变换

### 15.2.1 `Transformation` struct

```rust
pub struct Transformation {
    scale: Size<f32>,       // 缩放
    translate: Point<Pixels>,  // 平移
    rotate: Radians,        // 旋转
}
```

### 15.2.2 缩放：`scale()`

```rust
svg()
    .path("icons/zoom.svg")
    .with_transformation(Transformation::scale(size(2.0, 2.0)))  // 放大2倍
```

非均匀缩放：

```rust
// 水平拉伸2倍，垂直不变
svg()
    .path("icons/wide.svg")
    .with_transformation(Transformation::scale(size(2.0, 1.0)))
```

### 15.2.3 平移：`translate()`

```rust
svg()
    .path("icons/moved.svg")
    .with_transformation(Transformation::translate(point(px(10.0), px(20.0))))
```

### 15.2.4 旋转：`rotate()`

```rust
use gpui::radians;

svg()
    .path("icons/rotate.svg")
    .with_transformation(Transformation {
        rotate: radians(std::f32::consts::PI / 4.0),  // 45度
        ..Default::default()
    })
```

### 15.2.5 链式变换

组合多个变换：

```rust
// 先缩放，再平移，再旋转
svg()
    .path("icons/transformed.svg")
    .with_transformation(Transformation {
        scale: size(1.5, 1.5),
        translate: point(px(5.0), px(10.0)),
        rotate: radians(0.785),  // ~45度
    })
```

**注意**：变换只影响渲染，不影响布局尺寸和 hitbox 区域。

## 15.3 SVG 与 img 的对比

### 15.3.1 何时用 svg，何时用 img

| 场景 | 用 `svg()` | 用 `img()` |
|------|-----------|-----------|
| 图标（单一颜色可变） | 是 | 否 |
| 复杂插图 | 否 | 是 |
| 需要 `currentColor` 换肤 | 是 | 否 |
| 需要变换动画 | 是 | 否 |
| 光栅图片 | 否 | 是 |

```rust
// 图标：用 svg，可以通过 text_color 换色
svg()
    .path("icons/menu.svg")
    .size(Pixels(20.0))
    .text_color(Colors::get(cx).foreground)  // 跟随主题色

// 照片：用 img
img("photos/profile.jpg")
    .size(Pixels(100.0))
    .object_fit(ObjectFit::Cover)
```

### 15.3.2 性能考量

SVG 渲染比 PNG/JPG 更耗性能：
- SVG 需要解析 XML、构建路径、执行填充
- 光栅图片直接上传到 GPU 纹理

**图标数量多时**：考虑将 SVG 预渲染为位图或使用图标字体。
**图标频繁变色时**：SVG 的 `currentColor` 支持使其更适合主题切换。

### 15.3.3 交互能力差异

```rust
// SVG 支持交互
svg()
    .id("hoverable")
    .path("icons/heart.svg")
    .on_hover(|_, window, cx| { ... })
    .on_click(|_, window, cx| { ... })

// img 也支持交互，但 SVG 更适合做按钮图标
```

两者都支持事件注册。SVG 的优势在于：
- 内联 SVG 可以通过 CSS 控制颜色
- 矢量缩放不失真
- 文件通常更小
