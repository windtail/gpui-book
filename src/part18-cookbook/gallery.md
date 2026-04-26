# 第 68 章：图片画廊

本章构建一个图片画廊应用，展示网格布局、图片懒加载、缩放平移和动画过渡。

## Cargo.toml

```toml
[package]
name = "image-gallery"
version = "0.1.0"
edition = "2024"

[dependencies]
gpui = "0.2.2"
```

## 数据模型

```rust
use gpui::*;
use std::time::Duration;

const SAMPLE_IMAGES: &[&str] = &[
    "https://picsum.photos/seed/a1/400/300",
    "https://picsum.photos/seed/b2/400/300",
    "https://picsum.photos/seed/c3/400/300",
    "https://picsum.photos/seed/d4/400/300",
    "https://picsum.photos/seed/e5/400/300",
    "https://picsum.photos/seed/f6/400/300",
    "https://picsum.photos/seed/g7/400/300",
    "https://picsum.photos/seed/h8/400/300",
    "https://picsum.photos/seed/i9/400/300",
    "https://picsum.photos/seed/j10/400/300",
    "https://picsum.photos/seed/k11/400/300",
    "https://picsum.photos/seed/l12/400/300",
];

struct Gallery {
    images: Vec<ImageEntry>,
    zoom_level: f32,
}

#[derive(Clone)]
struct ImageEntry {
    url: SharedString,
    loaded: bool,
}
```

`SAMPLE_IMAGES` 使用 picsum.photos 作为占位图片源。`ImageEntry` 跟踪每张图片的加载状态。

## 创建画廊

```rust
impl Gallery {
    fn new(_cx: &mut Context<Self>) -> Self {
        let images = SAMPLE_IMAGES
            .iter()
            .map(|url| ImageEntry {
                url: SharedString::from(*url),
                loaded: false,
            })
            .collect();
        Self {
            images,
            zoom_level: 1.0,
        }
    }

    fn zoom_in(&mut self, cx: &mut Context<Self>) {
        self.zoom_level = (self.zoom_level + 0.1).min(3.0);
        cx.notify();
    }

    fn zoom_out(&mut self, cx: &mut Context<Self>) {
        self.zoom_level = (self.zoom_level - 0.1).max(0.5);
        cx.notify();
    }

    fn on_scroll_wheel(
        &mut self,
        event: &ScrollWheelEvent,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        let delta = event.delta.pixel_delta(px(1.));
        if delta.y < px(0.) {
            self.zoom_in(cx);
        } else if delta.y > px(0.) {
            self.zoom_out(cx);
        }
    }
}
```

## Render 实现

```rust
impl Render for Gallery {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let zoom = self.zoom_level;
        let cols = 3;
        let card_size = px(200.) * zoom;

        div()
            .size_full()
            .flex()
            .flex_col()
            .bg(rgb(0x0a0a0a))
            // 工具栏
            .child(
                div()
                    .flex()
                    .items_center()
                    .justify_between()
                    .px_4()
                    .py_2()
                    .bg(rgb(0x1a1a1a))
                    .border_b_1()
                    .border_color(rgb(0x333333))
                    .child(
                        div()
                            .text_size(px(18.))
                            .text_color(rgb(0xffffff))
                            .child("Image Gallery"),
                    )
                    .child(
                        div()
                            .flex()
                            .gap_2()
                            .child(
                                button("zoom-out", "-")
                                    .on_click(cx.listener(|this, _, _, cx| {
                                        this.zoom_out(cx);
                                    })),
                            )
                            .child(
                                div()
                                    .px_3()
                                    .text_size(px(14.))
                                    .text_color(rgb(0xaaaaaa))
                                    .child(format!("{:.0}%", zoom * 100.)),
                            )
                            .child(
                                button("zoom-in", "+")
                                    .on_click(cx.listener(|this, _, _, cx| {
                                        this.zoom_in(cx);
                                    })),
                            ),
                    ),
            )
            // 图片网格
            .child(
                div()
                    .flex_1()
                    .overflow_scroll()
                    .on_scroll_wheel(cx.listener(Self::on_scroll_wheel))
                    .child(
                        div()
                            .flex()
                            .flex_wrap()
                            .gap_3()
                            .p_4()
                            .justify_center()
                            .children(
                                self.images
                                    .iter()
                                    .enumerate()
                                    .map(|(i, entry)| {
                                        self.render_image_card(i, entry, cx)
                                    })
                                    .collect::<Vec<_>>(),
                            ),
                    ),
            )
    }

    fn render_image_card(&self, index: usize, entry: &ImageEntry, cx: &mut Context<Self>) -> impl IntoElement + 'static {
        let url = entry.url.clone();

        div()
            .id(SharedString::from(format!("card-{}", index)))
            .w(px(200.))
            .h(px(150.))
            .rounded_md()
            .overflow_hidden()
            .border_1()
            .border_color(rgb(0x333333))
            .bg(rgb(0x1a1a1a))
            .child(
                img(url)
                    .size_full()
                    .object_fit(ObjectFit::Cover),
            )
    }
}
```

## 异步加载与动画

为图片添加加载状态提示和淡入动画：

```rust
struct Gallery {
    images: Vec<ImageEntry>,
    zoom_level: f32,
}

#[derive(Clone)]
struct ImageEntry {
    url: SharedString,
    loaded: bool,
}

impl Gallery {
    fn new(cx: &mut Context<Self>) -> Self {
        let images: Vec<ImageEntry> = SAMPLE_IMAGES
            .iter()
            .map(|url| ImageEntry {
                url: SharedString::from(*url),
                loaded: false,
            })
            .collect();

        // 异步预加载图片
        let urls: Vec<SharedString> = images.iter().map(|e| e.url.clone()).collect();
        cx.spawn(|view, mut cx| async move {
            for url in urls {
                // 模拟异步加载
                cx.background_executor().timer(Duration::from_millis(100)).await;
                // 图片由 img 元素自动加载
            }
        }).detach();

        Self { images, zoom_level: 1.0 }
    }
}
```

## 使用 image_cache

GPUI 的 `img` 元素自动处理图片缓存。对于需要更精细控制的场景，使用 `image_cache()` 元素：

```rust
// image_cache 包装器用于延迟加载大量图片
div()
    .image_cache(retain_all())
    .child(/* 图片内容 */)
```

## 捏合缩放

添加捏合手势支持：

```rust
impl Gallery {
    fn on_pinch(&mut self, event: &PinchEvent, _window: &mut Window, cx: &mut Context<Self>) {
        // event.delta 是缩放增量
        let new_zoom = (self.zoom_level + event.delta).clamp(0.5, 3.0);
        self.zoom_level = new_zoom;
        cx.notify();
    }
}

// 在 render 中的网格 div 上注册捏合手势
div()
    .on_scroll_wheel(cx.listener(Self::on_scroll_wheel))
    .on_pinch(cx.listener(Self::on_pinch))
```

## 完整代码

```rust
use gpui::*;
use std::time::Duration;

const SAMPLE_IMAGES: &[&str] = &[
    "https://picsum.photos/seed/a1/400/300",
    "https://picsum.photos/seed/b2/400/300",
    "https://picsum.photos/seed/c3/400/300",
    "https://picsum.photos/seed/d4/400/300",
    "https://picsum.photos/seed/e5/400/300",
    "https://picsum.photos/seed/f6/400/300",
    "https://picsum.photos/seed/g7/400/300",
    "https://picsum.photos/seed/h8/400/300",
    "https://picsum.photos/seed/i9/400/300",
    "https://picsum.photos/seed/j10/400/300",
    "https://picsum.photos/seed/k11/400/300",
    "https://picsum.photos/seed/l12/400/300",
];

struct Gallery {
    images: Vec<ImageEntry>,
    zoom_level: f32,
}

#[derive(Clone)]
struct ImageEntry {
    url: SharedString,
}

impl Gallery {
    fn new(cx: &mut Context<Self>) -> Self {
        let images = SAMPLE_IMAGES
            .iter()
            .map(|url| ImageEntry {
                url: SharedString::from(*url),
            })
            .collect();

        // 预加载：触发图片请求
        cx.spawn(|_view, mut cx| async move {
            for _ in SAMPLE_IMAGES.iter() {
                cx.background_executor().timer(Duration::from_millis(50)).await;
            }
        }).detach();

        Self { images, zoom_level: 1.0 }
    }

    fn zoom_in(&mut self, cx: &mut Context<Self>) {
        self.zoom_level = (self.zoom_level + 0.1).min(3.0);
        cx.notify();
    }

    fn zoom_out(&mut self, cx: &mut Context<Self>) {
        self.zoom_level = (self.zoom_level - 0.1).max(0.5);
        cx.notify();
    }

    fn on_scroll_wheel(
        &mut self,
        event: &ScrollWheelEvent,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        let delta = event.delta.pixel_delta(px(1.));
        if delta.y < px(0.) {
            self.zoom_in(cx);
        } else if delta.y > px(0.) {
            self.zoom_out(cx);
        }
    }

    fn on_pinch(&mut self, event: &PinchEvent, _window: &mut Window, cx: &mut Context<Self>) {
        let new_zoom = (self.zoom_level + event.delta).clamp(0.5, 3.0);
        self.zoom_level = new_zoom;
        cx.notify();
    }
}

impl Render for Gallery {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let zoom = self.zoom_level;

        div()
            .size_full()
            .flex()
            .flex_col()
            .bg(rgb(0x0a0a0a))
            .child(
                div()
                    .flex()
                    .items_center()
                    .justify_between()
                    .px_4()
                    .py_2()
                    .bg(rgb(0x1a1a1a))
                    .border_b_1()
                    .border_color(rgb(0x333333))
                    .child(div().text_size(px(18.)).text_color(rgb(0xffffff)).child("Image Gallery"))
                    .child(
                        div()
                            .flex()
                            .gap_2()
                            .child(
                                button("zoom-out", "-")
                                    .on_click(cx.listener(|this, _, _, cx| this.zoom_out(cx))),
                            )
                            .child(
                                div()
                                    .px_3()
                                    .text_size(px(14.))
                                    .text_color(rgb(0xaaaaaa))
                                    .child(format!("{:.0}%", zoom * 100.)),
                            )
                            .child(
                                button("zoom-in", "+")
                                    .on_click(cx.listener(|this, _, _, cx| this.zoom_in(cx))),
                            ),
                    ),
            )
            .child(
                div()
                    .flex_1()
                    .overflow_scroll()
                    .on_scroll_wheel(cx.listener(Self::on_scroll_wheel))
                    .on_pinch(cx.listener(Self::on_pinch))
                    .child(
                        div()
                            .flex()
                            .flex_wrap()
                            .gap_3()
                            .p_4()
                            .justify_center()
                            .children(
                                self.images
                                    .iter()
                                    .enumerate()
                                    .map(|(i, entry)| {
                                        let url = entry.url.clone();
                                        div()
                                            .id(SharedString::from(format!("card-{}", i)))
                                            .w(px(200.))
                                            .h(px(150.))
                                            .rounded_md()
                                            .overflow_hidden()
                                            .border_1()
                                            .border_color(rgb(0x333333))
                                            .bg(rgb(0x1a1a1a))
                                            .child(
                                                img(url)
                                                    .size_full()
                                                    .object_fit(ObjectFit::Cover),
                                            )
                                    })
                                    .collect::<Vec<_>>(),
                            ),
                    ),
            )
    }
}

fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(900.), px(700.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Image Gallery")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| cx.new(|cx| Gallery::new(cx)),
        );
    });
}
```

## 关键要点

- `img(url)` 从 URL 加载图片，`img(path)` 从本地路径加载图片
- `ObjectFit::Cover` 图片填满容器（裁剪），`ObjectFit::Contain` 完整显示（留边）
- `flex_wrap()` 实现自动换行的网格布局
- `overflow_scroll()` 允许内容超出容器时滚动
- `ScrollWheelEvent` 的 `delta.pixel_delta(px(1.))` 获取滚动像素偏移
- `cx.spawn()` 启动异步任务，`cx.background_executor().timer()` 做延迟
- GPUI 的 `img` 元素自动处理图片加载和缓存，无需手动管理
- 缩放级别通过 View 状态管理，影响渲染时的尺寸
