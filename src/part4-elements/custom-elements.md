# 第17章：自定义元素

## 17.1 什么时候需要自定义 Element

### 17.1.1 div 无法满足的场景

```rust
// div 支持的能力有限：
// - Flexbox/Grid 布局
// - 基本样式（颜色、边框、阴影）
// - 鼠标/键盘事件

// 以下场景需要自定义 Element：
// - 代码编辑器（自定义光标、选区、文本测量）
// - 虚拟列表（自定义布局算法）
// - 数据可视化（直接绘制路径、点、线）
// - 自定义滚动条（精确控制绘制）
```

### 17.1.2 性能关键路径

```rust
// 场景：渲染 10000 个列表项

// 方案 A：10000 个 div
// 每个 div 包含：Interactivity（~200字节）+ StyleRefinement + 子元素
// 总计约 2-3 MB 内存分配

// 方案 B：自定义 Element
// 只包含必要数据：位置、尺寸、文本
// 总计约几百 KB
```

### 17.1.3 特殊布局需求

Taffy 只支持 Flexbox 和 Grid。如果你需要：
- 瀑布流布局
- 自定义对齐算法
- 基于内容的动态缩放

需要实现自定义 Element 的 `request_layout`。

## 17.2 实现 Element trait

### 17.2.1 定义 struct 与关联类型

```rust
use gpui::{
    Element, GlobalElementId, InspectorElementId, IntoElement, LayoutId,
    Window, App, Bounds, Pixels, Style, Point, Size, Length,
};

struct ProgressBar {
    progress: f32,        // 0.0 - 1.0
    width: Pixels,
    height: Pixels,
    fill_color: gpui::Hsla,
    bg_color: gpui::Hsla,
}

impl ProgressBar {
    type RequestLayoutState = LayoutId;
    type PrepaintState = ();
}
```

### 17.2.2 实现 `request_layout()`

```rust
impl Element for ProgressBar {
    type RequestLayoutState = LayoutId;
    type PrepaintState = ();

    fn id(&self) -> Option<gpui::ElementId> {
        None
    }

    fn source_location(&self) -> Option<&'static std::panic::Location<'static>> {
        None
    }

    fn request_layout(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        window: &mut Window,
        cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState) {
        let style = Style {
            size: Size {
                width: Length::Definite(self.width.into()),
                height: Length::Definite(self.height.into()),
            },
            ..Default::default()
        };

        let layout_id = window.request_layout(style, [], cx);
        (layout_id, layout_id)
    }
}
```

关键点：
- 设置 `Style` 的 `size` 告知 Taffy 你的尺寸
- `[]` 表示没有子元素
- 返回 `LayoutId` 作为 `RequestLayoutState`（后续阶段可能用到）

### 17.2.3 实现 `prepaint()`

```rust
    fn prepaint(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        bounds: Bounds<Pixels>,
        _request_layout: &mut Self::RequestLayoutState,
        window: &mut Window,
        cx: &mut App,
    ) -> Self::PrepaintState {
        // 注册 hitbox 以支持鼠标事件
        window.insert_hitbox(bounds, gpui::HitboxBehavior::Normal);
        ()
    }
```

如果你的元素需要响应鼠标事件，必须在 prepaint 阶段注册 hitbox。

### 17.2.4 实现 `paint()`

```rust
    fn paint(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        bounds: Bounds<Pixels>,
        _request_layout: &mut Self::RequestLayoutState,
        _prepaint: &mut Self::PrepaintState,
        window: &mut Window,
        cx: &mut App,
    ) {
        // 绘制背景
        window.paint_quad(gpui::fill(bounds, self.bg_color));

        // 绘制进度填充
        let fill_width = bounds.size.width * self.progress;
        let fill_bounds = Bounds::new(
            bounds.origin,
            Size::new(fill_width, bounds.size.height),
        );
        window.paint_quad(gpui::fill(fill_bounds, self.fill_color));
    }
```

### 17.2.5 完整示例：自定义滚动条

```rust
use gpui::{
    Element, GlobalElementId, InspectorElementId, IntoElement, LayoutId,
    Window, App, Bounds, Pixels, Style, Point, Size, Length, rgb, rgba,
    Hitbox, ContentMask,
};

struct Scrollbar {
    scroll_offset: f32,    // 当前滚动位置（0.0-1.0）
    thumb_ratio: f32,      // 滑块比例（0.0-1.0）
    vertical: bool,        // 垂直还是水平
    track_color: gpui::Hsla,
    thumb_color: gpui::Hsla,
    corner_radius: Pixels,
}

impl Scrollbar {
    fn new() -> Self {
        Self {
            scroll_offset: 0.0,
            thumb_ratio: 1.0,
            vertical: true,
            track_color: rgba(0x00000000),
            thumb_color: rgba(0x00000040),
            corner_radius: Pixels(4.0),
        }
    }
}

impl Element for Scrollbar {
    type RequestLayoutState = LayoutId;
    type PrepaintState = Hitbox;

    fn id(&self) -> Option<gpui::ElementId> {
        None
    }

    fn source_location(&self) -> Option<&'static std::panic::Location<'static>> {
        None
    }

    fn request_layout(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        window: &mut Window,
        cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState) {
        let track_width = Pixels(8.0);
        let style = Style {
            size: if self.vertical {
                Size {
                    width: Length::Definite(track_width.into()),
                    height: Length::Auto,
                }
            } else {
                Size {
                    width: Length::Auto,
                    height: Length::Definite(track_width.into()),
                }
            },
            ..Default::default()
        };

        let layout_id = window.request_layout(style, [], cx);
        (layout_id, layout_id)
    }

    fn prepaint(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        bounds: Bounds<Pixels>,
        _request_layout: &mut Self::RequestLayoutState,
        window: &mut Window,
        cx: &mut App,
    ) -> Self::PrepaintState {
        // 注册 hitbox 用于拖拽滑块
        window.insert_hitbox(bounds, gpui::HitboxBehavior::Normal)
    }

    fn paint(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        bounds: Bounds<Pixels>,
        _request_layout: &mut Self::RequestLayoutState,
        _prepaint: &mut Self::PrepaintState,
        window: &mut Window,
        cx: &mut App,
    ) {
        // 绘制轨道
        window.paint_quad(gpui::fill(bounds, self.track_color));

        // 计算滑块位置和大小
        if self.vertical {
            let thumb_height = bounds.size.height * self.thumb_ratio;
            let track_height = bounds.size.height - thumb_height;
            let thumb_y = bounds.origin.y + track_height * self.scroll_offset;

            let thumb_bounds = Bounds::new(
                Point::new(bounds.origin.x, px(thumb_y)),
                Size::new(bounds.size.width, thumb_height),
            );

            window.paint_quad(gpui::fill(thumb_bounds, self.thumb_color));
        } else {
            let thumb_width = bounds.size.width * self.thumb_ratio;
            let track_width = bounds.size.width - thumb_width;
            let thumb_x = bounds.origin.x + track_width * self.scroll_offset;

            let thumb_bounds = Bounds::new(
                Point::new(px(thumb_x), bounds.origin.y),
                Size::new(thumb_width, bounds.size.height),
            );

            window.paint_quad(gpui::fill(thumb_bounds, self.thumb_color));
        }
    }
}
```

实现 `IntoElement` 使其可以作为子元素使用：

```rust
impl IntoElement for Scrollbar {
    type Element = Self;

    fn into_element(self) -> Self::Element {
        self
    }
}
```

## 17.3 自定义布局

### 17.3.1 `AvailableSpace` 约束

```rust
use gpui::AvailableSpace;

fn request_layout(
    &mut self,
    _id: Option<&GlobalElementId>,
    _inspector_id: Option<&InspectorElementId>,
    window: &mut Window,
    cx: &mut App,
) -> (LayoutId, Self::RequestLayoutState) {
    // AvailableSpace 有三种模式：
    // - Definite(px(100.0)): 固定尺寸
    // - MinContent: 最小内容尺寸
    // - MaxContent: 最大内容尺寸

    let style = Style {
        size: Size {
            width: Length::Definite(self.width.into()),
            height: Length::Definite(self.height.into()),
        },
        ..Default::default()
    };

    let layout_id = window.request_layout(style, [], cx);
    (layout_id, layout_id)
}
```

### 17.3.2 子元素布局委托

如果自定义 Element 有子元素，需要委托布局：

```rust
struct CustomContainer {
    children: Vec<AnyElement>,
}

impl Element for CustomContainer {
    type RequestLayoutState = (LayoutId, Vec<LayoutId>);
    type PrepaintState = ();

    fn request_layout(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        window: &mut Window,
        cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState) {
        // 先请求子元素布局
        let child_layout_ids: Vec<_> = self.children.iter_mut()
            .map(|child| child.request_layout(window, cx))
            .collect();

        // 再请求容器布局，传入子元素 ID
        let style = Style::default();
        let layout_id = window.request_layout(
            style,
            child_layout_ids.iter().copied(),
            cx,
        );

        (layout_id, (layout_id, child_layout_ids))
    }

    fn prepaint(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        bounds: Bounds<Pixels>,
        state: &mut Self::RequestLayoutState,
        window: &mut Window,
        cx: &mut App,
    ) -> Self::PrepaintState {
        // prepaint 子元素
        for child in &mut self.children {
            child.prepaint(window, cx);
        }
        ()
    }

    fn paint(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        bounds: Bounds<Pixels>,
        state: &mut Self::RequestLayoutState,
        prepaint: &mut Self::PrepaintState,
        window: &mut Window,
        cx: &mut App,
    ) {
        // paint 子元素
        for child in &mut self.children {
            child.paint(window, cx);
        }
    }
}
```

### 17.3.3 内容裁剪与 `with_content_mask()`

```rust
fn paint(
    &mut self,
    _id: Option<&GlobalElementId>,
    _inspector_id: Option<&InspectorElementId>,
    bounds: Bounds<Pixels>,
    _request_layout: &mut Self::RequestLayoutState,
    _prepaint: &mut Self::PrepaintState,
    window: &mut Window,
    cx: &mut App,
) {
    // 裁剪子元素的绘制区域
    window.with_content_mask(
        Some(ContentMask { bounds }),
        |window| {
            // 在这个闭包内，所有绘制都被 bounds 裁剪
            self.paint_children(window, cx);
        },
    );
}
```

`with_content_mask()` 设置内容裁剪区域，子元素超出边界的部分会被裁掉。等同于 CSS 的 `overflow: hidden`。

## 17.4 交互支持

### 17.4.1 注册事件处理器

自定义 Element 可以通过 `Interactivity` 结构体获得事件支持：

```rust
struct InteractiveElement {
    interactivity: gpui::Interactivity,
    data: String,
}

impl Element for InteractiveElement {
    fn request_layout(
        &mut self,
        global_id: Option<&GlobalElementId>,
        inspector_id: Option<&InspectorElementId>,
        window: &mut Window,
        cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState) {
        // 委托给 Interactivity 处理
        self.interactivity.request_layout(
            global_id,
            inspector_id,
            window,
            cx,
            |style, window, cx| {
                window.request_layout(style, [], cx)
            },
        )
    }

    fn prepaint(
        &mut self,
        global_id: Option<&GlobalElementId>,
        inspector_id: Option<&InspectorElementId>,
        bounds: Bounds<Pixels>,
        state: &mut Self::RequestLayoutState,
        window: &mut Window,
        cx: &mut App,
    ) -> Self::PrepaintState {
        self.interactivity.prepaint(
            global_id,
            inspector_id,
            bounds,
            state,
            window,
            cx,
            |_, _, hitbox, window, cx| {
                hitbox
            },
        )
    }
}
```

### 17.4.2 Hitbox 与命中检测

```rust
fn prepaint(
    &mut self,
    ...
    bounds: Bounds<Pixels>,
    ...
) -> Self::PrepaintState {
    // 注册 hitbox 是事件接收的前提
    // 没有 hitbox 的元素不会收到鼠标事件

    // 基本 hitbox
    let hitbox = window.insert_hitbox(
        bounds,
        gpui::HitboxBehavior::Normal,
    );

    hitbox
}
```

Hitbox 注册后，GPUI 在事件分发时通过空间索引查找命中的元素。

## 17.5 性能优化

### 17.5.1 缓存布局计算结果

```rust
struct CachedLayoutElement {
    data: Vec<f32>,
    cached_layout: Option<LayoutData>,
}

struct LayoutData {
    positions: Vec<Point<Pixels>>,
    sizes: Vec<Size<Pixels>>,
}

impl Element for CachedLayoutElement {
    fn request_layout(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        window: &mut Window,
        cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState) {
        // 只有数据变化时才重新计算
        let layout = self.cached_layout.get_or_insert_with(|| {
            self.compute_expensive_layout()
        });

        // 使用缓存的布局数据
        ...
    }
}
```

### 17.5.2 减少状态传递

```rust
// 低效：在三个阶段之间传递大量数据
struct InefficientElement {
    type RequestLayoutState = ComplexData;  // 100+ 字节
    type PrepaintState = ComplexData2;      // 另一个 100+ 字节
}

// 高效：只传递必要引用
struct EfficientElement {
    type RequestLayoutState = LayoutId;  // 只传递句柄
    type PrepaintState = ();             // 不需要中间状态
}
```

### 17.5.3 惰性绘制

```rust
struct LazyElement {
    items: Vec<ItemData>,
    visible_range: std::ops::Range<usize>,  // 只绘制可见范围内的项
}

impl Element for LazyElement {
    fn paint(
        &mut self,
        _id: Option<&GlobalElementId>,
        _inspector_id: Option<&InspectorElementId>,
        bounds: Bounds<Pixels>,
        _request_layout: &mut Self::RequestLayoutState,
        _prepaint: &mut Self::PrepaintState,
        window: &mut Window,
        cx: &mut App,
    ) {
        // 只绘制可见范围内的项目
        for i in self.visible_range.clone() {
            if let Some(item) = self.items.get(i) {
                let item_bounds = self.compute_item_bounds(i);
                window.paint_quad(gpui::fill(item_bounds, item.color));
                // 文本通过 TextRun + window.paint_glyph() 绘制
                // 或使用 div().child(text) 在更高层级渲染
            }
        }
        // 不可见的项不产生任何绘制指令
    }
}
```

这是虚拟列表的核心思想：只对可见区域进行绘制操作，大幅减少 GPU 工作量。
