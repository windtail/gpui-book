# 第28章：手势事件

## 28.1 `PinchEvent` 捏合缩放

`PinchEvent` 在触控板上双指捏合时触发：

```rust
use gpui::{
    div, PinchEvent, TouchPhase,
    InteractiveElement, ParentElement, IntoElement, Context, Window,
    px,
};

struct ZoomableImage {
    scale: f32,
}

impl Render for ZoomableImage {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("zoomable")
            .size_full()
            .on_resize(cx.listener(|this, event: &gpui::ResizeEvent, _, cx| {
                // 窗口 resize 时调整
                cx.notify();
            }))
            .on_pinch(cx.listener(|this, event: &PinchEvent, _, cx| {
                // event.position: Point<Pixels> — 捏合中心位置
                // event.delta: f32 — 缩放增量，正值放大，负值缩小
                // event.modifiers: Modifiers
                // event.phase: TouchPhase

                match event.phase {
                    TouchPhase::Started => {
                        // 捏合开始
                    }
                    TouchPhase::Moved => {
                        // 持续缩放
                        this.scale += event.delta;
                        this.scale = this.scale.max(0.1).min(10.0);
                        cx.notify();
                    }
                    TouchPhase::Ended => {
                        // 捏合结束
                    }
                }
            }))
            .child(format!("zoom: {:.1}x", this.scale))
    }
}
```

`PinchEvent` 结构：

```rust
pub struct PinchEvent {
    pub position: Point<Pixels>,
    pub delta: f32,        // 正值 = 放大，负值 = 缩小
    pub modifiers: Modifiers,
    pub phase: TouchPhase,
}
```

`delta` 是相对值：`0.1` 表示 10% 的缩放增量。

## 28.2 Touch 阶段追踪

`TouchPhase` 枚举：

```rust
pub enum TouchPhase {
    Started,   // 手势开始
    Moved,     // 手势进行中
    Ended,     // 手势结束
}
```

完整处理触摸阶段：

```rust
struct PinchToZoom {
    scale: f32,
    base_scale: f32,
}

impl Render for PinchToZoom {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("pinch-zoom")
            .size_full()
            .on_pinch(cx.listener(|this, event: &PinchEvent, _, cx| {
                match event.phase {
                    TouchPhase::Started => {
                        this.base_scale = this.scale;
                    }
                    TouchPhase::Moved => {
                        this.scale = this.base_scale * (1.0 + event.delta);
                        this.scale = this.scale.clamp(0.1, 10.0);
                        cx.notify();
                    }
                    TouchPhase::Ended => {
                        // 可以加弹性动画回弹到边界
                    }
                }
            }))
            .child(format!("scale: {:.2}", this.scale))
    }
}
```

`TouchPhase` 也用于 `ScrollWheelEvent`：

```rust
div()
    .id("scroll-view")
    .on_scroll_wheel(cx.listener(|this, event: &ScrollWheelEvent, _, cx| {
        match event.touch_phase {
            TouchPhase::Started => this.scroll_momentum = 0.0,
            TouchPhase::Moved => this.apply_scroll(event),
            TouchPhase::Ended => this.apply_momentum(),
        }
        cx.notify();
    }))
```

## 28.3 平台特定手势

捏合手势主要在 macOS 触控板上可用。在 Linux 和部分 Windows 触控设备上行为取决于平台后端实现。

```rust
// 检测平台
let supports_pinch = cfg!(target_os = "macos");

// 在 Linux 上提供替代方案（如 Ctrl+滚轮缩放）
div()
    .id("zoomable")
    .on_pinch(cx.listener(|this, event: &PinchEvent, _, cx| {
        this.apply_zoom(event.delta, cx);
    }))
    .on_scroll_wheel(cx.listener(|this, event: &ScrollWheelEvent, _, cx| {
        // Ctrl+滚轮作为捏合的替代
        if event.modifiers.control {
            let delta = -event.delta.pixel_delta(this.line_height).y.0 * 0.01;
            this.apply_zoom(delta, cx);
        }
    }))
```

`PinchEvent` 实现了 `Deref<Target = Modifiers>`：

```rust
// 直接访问修饰符
div()
    .id("pinch")
    .on_pinch(cx.listener(|this, event: &PinchEvent, _, cx| {
        if event.shift {
            // Shift+Pinch: 非等比缩放
        } else {
            // 普通捏合: 等比缩放
        }
    }))
```

## 28.4 手势与缩放的组合

结合 `PinchEvent` 和平移操作实现完整的缩放/平移交互：

```rust
struct ImageViewer {
    scale: f32,
    offset: Point<Pixels>,
    last_pinch_position: Option<Point<Pixels>>,
    panning: bool,
    pan_start: Option<Point<Pixels>>,
}

impl Render for ImageViewer {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("image-viewer")
            .size_full()
            .overflow_hidden()
            // 捏合缩放
            .on_pinch(cx.listener(|this, event: &PinchEvent, _, cx| {
                match event.phase {
                    TouchPhase::Started => {
                        this.last_pinch_position = Some(event.position);
                    }
                    TouchPhase::Moved => {
                        // 以捏合中心为基准缩放
                        if let Some(pinch_pos) = this.last_pinch_position {
                            let old_scale = this.scale;
                            this.scale += event.delta;
                            this.scale = this.scale.clamp(0.1, 10.0);

                            // 调整偏移使捏合中心点不动
                            let scale_factor = this.scale / old_scale;
                            this.offset = event.position
                                - (pinch_pos - this.offset) * scale_factor;

                            this.last_pinch_position = Some(event.position);
                            cx.notify();
                        }
                    }
                    TouchPhase::Ended => {
                        this.last_pinch_position = None;
                    }
                }
            }))
            // 拖拽平移
            .on_mouse_down(MouseButton::Left, cx.listener(|this, event: &MouseDownEvent, _, cx| {
                this.panning = true;
                this.pan_start = Some(event.position);
            }))
            .on_mouse_move(cx.listener(|this, event: &MouseMoveEvent, _, cx| {
                if this.panning {
                    if let Some(start) = this.pan_start {
                        this.offset += event.position - start;
                        this.pan_start = Some(event.position);
                        cx.notify();
                    }
                }
            }))
            .on_mouse_up(MouseButton::Left, cx.listener(|this, _, _, cx| {
                this.panning = false;
                this.pan_start = None;
                cx.notify();
            }))
            // 滚轮缩放（以鼠标位置为中心）
            .on_scroll_wheel(cx.listener(|this, event: &ScrollWheelEvent, _, cx| {
                if event.modifiers.control {
                    let delta = -event.delta.pixel_delta(px(20.0)).y.0 * 0.005;
                    let old_scale = this.scale;
                    this.scale = (this.scale + delta).clamp(0.1, 10.0);
                    let factor = this.scale / old_scale;
                    this.offset = event.position - (event.position - this.offset) * factor;
                    cx.notify();
                }
            }))
    }
}
```

这个示例展示了：
- 捏合缩放以捏合中心为基准
- 拖拽平移
- Ctrl+滚轮作为替代缩放方案
- 三种交互方式的协调
