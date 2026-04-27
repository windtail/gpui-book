# 第21章：Taffy 引擎内部

## 21.1 Taffy 简介

### 21.1.1 什么是 Taffy

Taffy 是一个用 Rust 编写的 Flexbox 和 Grid 布局引擎。它是独立的开源项目，被多个 Rust GUI 框架使用。GPUI 将 Taffy 作为底层布局引擎，所有 `div` 的 flex 和 grid 布局最终都交给 Taffy 计算。

Taffy 的职责：
- 解析样式（flex direction、gap、grid template 等）
- 递归计算每个节点的尺寸和位置
- 处理约束传播和空间分配

### 21.1.2 GPUI 如何集成 Taffy

GPUI 通过 `TaffyLayoutEngine` 封装 Taffy：

```rust
// gpui/src/taffy.rs 核心结构
pub struct TaffyLayoutEngine {
    taffy: TaffyTree<NodeContext>,
    absolute_layout_bounds: FxHashMap<LayoutId, Bounds<Pixels>>,
    absolute_outer_origins: FxHashMap<LayoutId, Point<f32>>,
    computed_layouts: FxHashSet<LayoutId>,
    layout_bounds_scratch_space: Vec<LayoutId>,
}
```

每帧渲染时，GPUI 执行以下步骤：

```rust
// 1. 请求布局节点
let layout_id = taffy.request_layout(style, rem_size, scale_factor, &children);

// 2. 计算布局
taffy.compute_layout(layout_id, available_space, window, cx);

// 3. 获取边界
let bounds = taffy.layout_bounds(layout_id, scale_factor);
```

样式转换通过 `ToTaffy` trait 完成：

```rust
// Style -> taffy::style::Style
impl ToTaffy<taffy::style::Style> for Style {
    fn to_taffy(&self, rem_size: Pixels, scale_factor: f32) -> taffy::style::Style {
        taffy::style::Style {
            display: self.display.into(),
            flex_direction: self.flex_direction.into(),
            flex_wrap: self.flex_wrap.into(),
            flex_grow: self.flex_grow,
            flex_shrink: self.flex_shrink,
            // ... 更多字段
        }
    }
}
```

## 21.2 `LayoutId`

### 21.2.1 不透明布局句柄

`LayoutId` 是 GPUI 对 Taffy `NodeId` 的封装，表示布局树中的一个节点：

```rust
#[repr(transparent)]
pub struct LayoutId(NodeId);
```

`repr(transparent)` 保证 `LayoutId` 和 `NodeId` 在内存中布局相同，可以零成本转换。

```rust
// 创建 LayoutId 的方式
let id = taffy.request_layout(style, rem_size, scale_factor, &[]);
// 或
let id = taffy.request_measured_layout(style, rem_size, scale_factor, measure_fn);
```

### 21.2.2 布局请求传播

布局请求从根节点向下传播，子元素收集父节点传递的 `LayoutId`：

```rust
// 在 Element::request_layout 中
fn request_layout(
    &mut self,
    window: &mut Window,
    cx: &mut App,
    global_id: Option<&GlobalElementId>,
) -> LayoutId {
    // 1. 收集子元素的 LayoutId
    let child_layout_ids: Vec<LayoutId> = self
        .children
        .iter_mut()
        .map(|child| child.request_layout(window, cx))
        .collect();

    // 2. 将自身样式 + 子 LayoutId 提交给 Taffy
    window.request_layout(self, &child_layout_ids, cx)
}
```

这个两阶段过程确保：
1. 子元素先请求自己的布局（递归到底）
2. 父元素拿到所有子的 `LayoutId` 后，一次性提交给 Taffy
3. Taffy 拥有完整的树信息，可以一次性计算全局最优布局

## 21.3 `AvailableSpace`

### 21.3.1 `Definite` vs `MinContent` vs `MaxContent`

`AvailableSpace` 描述给元素分配的空间约束：

```rust
pub enum AvailableSpace {
    /// 固定像素尺寸
    Definite(Pixels),
    /// 最小内容约束：元素应缩小到能容纳内容的最小尺寸
    MinContent,
    /// 最大内容约束：元素应扩展到能容纳内容的最大尺寸
    MaxContent,
}
```

```rust
// 构造函数
AvailableSpace::min_size()  // Size { width: MinContent, height: MinContent }
AvailableSpace::Definite(px(300.))
px(300.).into()             // 通过 From 转换
```

### 21.3.2 约束传播机制

布局计算时，约束从父节点向子节点传播：

```
窗口可用空间 (Definite(800px), Definite(600px))
  -> Flex 容器 (Definite(800px), Definite(600px))
    -> Sidebar (Definite(200px), Definite(600px))   // w(px(200.))
    -> Main Content (Definite(584px), Definite(600px))  // flex_1, 扣除 sidebar + gap
      -> Card (MinContent, Definite(...))           // 内容决定宽度
```

关键规则：
- `Definite` 约束传播到具有确定尺寸的直接子元素
- `MinContent` 传播到 `flex_shrink` 的子元素
- `MaxContent` 传播到 `flex_grow` 的子元素
- `flex_1()` 的子元素收到扣除其他子元素后的剩余空间

```rust
// 可用的快捷方法
AvailableSpace::Definite(px(300.))  // 确定 300px
AvailableSpace::MinContent           // 最小内容约束
AvailableSpace::MaxContent           // 最大内容约束
```

## 21.4 自定义布局集成

### 21.4.1 `request_measured_layout()` 接入外部测量

对于内容尺寸无法提前确定的元素（如文本、自定义图形），使用 `request_measured_layout()`：

```rust
pub fn request_measured_layout(
    &mut self,
    style: Style,
    measure: impl FnMut(
        Size<Option<Pixels>>,    // 已知维度（None = 需要测量）
        Size<AvailableSpace>,    // 可用空间约束
        &mut Window,
        &mut App,
    ) -> Size<Pixels> + 'static,
) -> LayoutId
```

自定义 Element 的使用方式：

```rust
struct TextElement {
    text: String,
    font_size: Pixels,
}

impl Element for TextElement {
    type RequestLayoutState = LayoutId;
    type PrepaintState = ();

    fn request_layout(
        &mut self,
        global_id: Option<&GlobalElementId>,
        window: &mut Window,
        cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState) {
        let text = self.text.clone();
        let font_size = self.font_size;

        let layout_id = window.request_measured_layout(
            Style::default(),
            move |known_dimensions, available_space, window, cx| {
                // 如果宽度已知，文本可以换行；否则不换行
                let width = known_dimensions.width;
                let text_bounds = match width {
                    Some(w) => Bounds::new(point(px(0.), px(0.)), size(w, available_space.height.into())),
                    None => Bounds::new(point(px(0.), px(0.)), size(available_space.width.into(), available_space.height.into())),
                };

                // 测量文本尺寸
                let text_system = window.text_system();
                let run = TextRun {
                    len: text.len(),
                    font: Font { size: font_size, ..Default::default() },
                    ..Default::default()
                };

                text_system
                    .shape_text(text, font_size, &[run], text_bounds.size, None)
                    .map(|lines| {
                        // 返回实际测量尺寸
                        let total_height = lines.iter().map(|l| l.size.height).sum::<Pixels>();
                        let max_width = lines.iter().map(|l| l.size.width).fold(px(0.), |a, b| a.max(b));
                        size(max_width, total_height)
                    })
                    .unwrap_or(size(px(0.), px(0.)))
            },
        );

        (layout_id, layout_id)
    }

    fn prepaint(&mut self, _: LayoutId, _: &mut Window, _: &mut App) {}

    fn paint(&mut self, _: LayoutId, prepaint: (), window: &mut Window, cx: &mut App) {}
}
```

### 21.4.2 自定义布局算法

如果需要完全自定义布局逻辑（不走 Taffy），可以直接在 `request_layout()` 中计算：

```rust
struct CircleLayout {
    items: Vec<AnyElement>,
    radius: Pixels,
}

impl Element for CircleLayout {
    type RequestLayoutState = Vec<LayoutId>;
    type PrepaintState = ();

    fn request_layout(
        &mut self,
        _: Option<&GlobalElementId>,
        window: &mut Window,
        cx: &mut App,
    ) -> (LayoutId, Self::RequestLayoutState) {
        // 先让子元素各自请求布局
        let child_ids: Vec<LayoutId> = self
            .items
            .iter_mut()
            .map(|item| item.request_layout(window, cx))
            .collect();

        // 仍然通过 Taffy 注册一个节点，但使用自定义尺寸
        let style = Style {
            size: size(self.radius * 2., self.radius * 2.).into(),
            ..Default::default()
        };

        let layout_id = window.request_layout(style, &child_ids, cx);
        (layout_id, child_ids)
    }

    fn prepaint(&mut self, child_ids: Vec<LayoutId>, window: &mut Window, cx: &mut App) {
        // Taffy 已自动完成子元素布局，prepaint 阶段可用于
        // 注册 hitbox 或其他准备工作
    }
}
```

## 21.5 布局调试

### 21.5.1 使用 `debug()` 查看布局

在 `debug_assertions` 启用时，GPUI 提供了 `debug()` 方法为元素添加可视化边框：

```rust
div()
    .flex()
    .child("内容")
    .debug()  // 添加随机颜色边框以调试布局
```

### 21.5.2 打印布局树

Taffy 内部支持 `Debug` trait，可以通过设置环境变量 `TAFFY_DEBUG=1` 启用布局调试输出。

在开发中，使用 `.debug()` 方法更直观：

```rust
// 为元素添加调试边框
div()
    .debug()           // 红色边框显示边界
    .child(div().debug().child("嵌套调试"))
```

### 21.5.2 性能分析

影响 Taffy 布局性能的因素：

```rust
// 布局复杂度度量
let child_count = taffy.count_all_children(root_id);
let max_depth = taffy.max_depth(0, root_id);

// 经验规则：
// - 布局树深度 < 20 通常没问题
// - 总节点数 < 1000 通常没问题
// - 超过这些阈值时考虑拆分组件或使用缓存
```

优化策略：
- 减少不必要的嵌套 `div`
- 使用 `grid` 替代多层 `flex` 嵌套
- 对不经常变化的子树使用视图缓存
- 虚拟列表（`List`、`UniformList`）避免大量 DOM 节点
