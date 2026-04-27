# 第16章：Canvas 元素

## 16.1 `canvas()` 函数

### 16.1.1 prepaint 闭包：布局准备

```rust
use gpui::{canvas, div, IntoElement, Context, Window, App, Bounds, Pixels};

struct ChartView {
    data: Vec<f32>,
}

impl Render for ChartView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let data = self.data.clone();

        div()
            .size_full()
            .child(
                canvas(
                    move |bounds, window, cx| {
                        // prepaint: 计算绘制所需的数据
                        let max_val = data.iter().cloned().fold(0.0f32, f32::max);
                        let bar_width = bounds.size.width / data.len() as f32;
                        ChartPrepaint {
                            max_val,
                            bar_width,
                            bounds,
                            data,
                        }
                    },
                    move |prepaint, bounds, window, cx| {
                        // paint: 执行实际绘制
                        // 使用 prepaint 传递的数据
                    },
                )
            )
    }
}
```

`canvas()` 接受两个闭包：
- **prepaint**：`FnOnce(Bounds<Pixels>, &mut Window, &mut App) -> T`
- **paint**：`FnOnce(Bounds<Pixels>, T, &mut Window, &mut App)`

### 16.1.2 paint 闭包：自定义绘制

```rust
canvas(
    |bounds, window, cx| {
        // prepaint 阶段：计算
        let bar_count = 10;
        let bar_width = bounds.size.width / bar_count as f32;
        bar_width
    },
    |bar_width, bounds, window, cx| {
        // paint 阶段：绘制
        for i in 0..10 {
            let x = bounds.origin.x + i as f32 * bar_width;
            let height = bounds.size.height * 0.5;
            let y = bounds.origin.y + bounds.size.height - height;

            let rect = Bounds::new(
                point(px(x), px(y)),
                size(px(bar_width - 2.0), px(height)),
            );
            window.paint_quad(gpui::fill(rect, rgb(0x3b82f6)));
        }
    },
)
```

paint 闭包接收 prepaint 的返回值作为第二个参数，实现两阶段之间的数据传递。

## 16.2 绘制操作

### 16.2.1 绘制 Quad（矩形）

```rust
canvas(
    |bounds, window, cx| (),
    |_, bounds, window, cx| {
        // 实心矩形
        window.paint_quad(gpui::fill(bounds, rgb(0x3b82f6)));

        // 带边框的矩形
        window.paint_quad(gpui::quad(
            bounds,
            gpui::Corners::all(Pixels(8.0)),
            gpui::Background::Transparent,
            gpui::Edges::all(Pixels(2.0)),
            gpui::rgb(0x1e40af),
            gpui::BorderStyle::Solid,
        ));

        // 半透明覆盖层
        window.paint_quad(gpui::fill(
            bounds,
            gpui::rgba(0x00000080),  // 50% 透明黑色
        ));
    },
)
```

### 16.2.2 绘制 Path（路径）

```rust
use gpui::{Path, point, px};

canvas(
    |bounds, window, cx| {
        let mut path = Path::new(point(px(10.0), px(10.0)));
        path.line_to(point(px(100.0), px(50.0)));
        path.line_to(point(px(50.0), px(100.0)));
        path
    },
    |path, bounds, window, cx| {
        window.paint_path(path, gpui::rgb(0xff0000));
    },
)
```

### 16.2.3 绘制图片

在 canvas 中绘制图片需要使用 `img()` 元素而非 `paint_sprite`：

```rust
canvas(
    |bounds, window, cx| {
        bounds
    },
    |bounds, window, cx| {
        // 在 canvas 中使用 img 元素渲染图片
        // 注意：canvas 内部不能直接渲染元素，
        // 图片加载应通过 Asset 系统或 img() 元素完成
    },
)
```

对于需要在自定义位置渲染图片的场景，推荐使用 `div()` + `img()` 组合：

```rust
div()
    .relative()
    .child(
        img("icon.png")
            .absolute()
            .top(px(10.))
            .left(px(10.))
            .size(px(32.))
    )
```

## 16.3 使用场景

### 16.3.1 图表绘制

```rust
struct LineChart {
    points: Vec<(f32, f32)>,  // (x%, y%)
    line_color: gpui::Hsla,
}

impl Render for LineChart {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let points = self.points.clone();
        let color = self.line_color;

        canvas(
            move |bounds, window, cx| {
                // 将百分比坐标转换为像素坐标
                let pixel_points: Vec<_> = points.iter().map(|(x, y)| {
                    point(
                        bounds.origin.x + bounds.size.width * x / 100.0,
                        bounds.origin.y + bounds.size.height * (1.0 - y / 100.0),
                    )
                }).collect();
                pixel_points
            },
            move |pixel_points, bounds, window, cx| {
                // 绘制连线
                for i in 1..pixel_points.len() {
                    let start = pixel_points[i - 1];
                    let end = pixel_points[i];

                    let mut path = gpui::Path::new();
                    path.move_to(start);
                    path.line_to(end);
                    window.paint_path(path, color);
                }

                // 绘制数据点
                for pt in &pixel_points {
                    let dot = gpui::Bounds::new(
                        point(pt.x - px(2.0), pt.y - px(2.0)),
                        size(px(4.0), px(4.0)),
                    );
                    window.paint_quad(gpui::fill(dot, color));
                }
            },
        )
        .size_full()
    }
}
```

### 16.3.2 自定义进度条

```rust
struct ProgressBar {
    progress: f32,  // 0.0 - 1.0
}

impl Render for ProgressBar {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let progress = self.progress;

        div()
            .h(px(8.0))
            .rounded_full()
            .bg(gray_200())
            .child(
                canvas(
                    move |bounds, window, cx| {
                        let fill_width = bounds.size.width * progress;
                        gpui::Bounds::new(
                            bounds.origin,
                            gpui::Size::new(fill_width, bounds.size.height),
                        )
                    },
                    move |fill_bounds, bounds, window, cx| {
                        window.paint_quad(gpui::fill(fill_bounds, rgb(0x3b82f6)));
                    },
                )
                .absolute()
                .inset_0()
            )
    }
}
```

### 16.3.3 粒子效果

```rust
struct ParticleEffect {
    particles: Vec<Particle>,
}

struct Particle {
    x: f32,
    y: f32,
    vx: f32,
    vy: f32,
    life: f32,
}

impl Render for ParticleEffect {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let particles = self.particles.clone();

        // 请求动画帧以持续更新
        window.request_animation_frame();

        canvas(
            move |bounds, window, cx| particles,
            move |particles, bounds, window, cx| {
                for p in &particles {
                    if p.life > 0.0 {
                        let dot = gpui::Bounds::new(
                            point(px(p.x), px(p.y)),
                            size(px(3.0), px(3.0)),
                        );
                        let alpha = (p.life * 255.0) as u8;
                        window.paint_quad(gpui::fill(
                            dot,
                            gpui::rgba(0x3b82f6_u32 | ((alpha as u32) << 24)),
                        ));
                    }
                }
            },
        )
        .size_full()
    }
}
```

## 16.4 Canvas vs 自定义 Element

```rust
// Canvas：轻量，适合一次性绘制
canvas(
    |bounds, window, cx| { /* 计算 */ },
    |data, bounds, window, cx| { /* 绘制 */ },
)
// 优点：简单直接，无需定义新类型
// 缺点：每帧重建闭包，不支持子元素，无 hitbox

// 自定义 Element：重量，适合复杂场景
struct MyElement { /* 数据 */ }
impl Element for MyElement { ... }
// 优点：完整控制三阶段生命周期，支持子元素和 hitbox
// 缺点：代码量大，需要手动实现所有方法
```

选择原则：
- **Canvas**：一次性装饰、图表、简单自定义绘制
- **自定义 Element**：需要交互、子元素、复杂布局的元素
