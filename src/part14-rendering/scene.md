# 第 51 章：场景图

场景图（Scene）是 GPUI 渲染管线的核心数据结构。所有元素的 paint 操作最终被收集为 `Scene` 中的一组原始图元。

## 51.1 Scene struct

`Scene` 收集每帧的所有绘制操作：

```rust
pub struct Scene {
    pub(crate) paint_operations: Vec<PaintOperation>,
    primitive_bounds: BoundsTree<ScaledPixels>,
    layer_stack: Vec<DrawOrder>,
    pub shadows: Vec<Shadow>,
    pub quads: Vec<Quad>,
    pub paths: Vec<Path<ScaledPixels>>,
    pub underlines: Vec<Underline>,
    pub monochrome_sprites: Vec<MonochromeSprite>,
    pub subpixel_sprites: Vec<SubpixelSprite>,
    pub polychrome_sprites: Vec<PolychromeSprite>,
    pub surfaces: Vec<PaintSurface>,
}
```

两层存储：
- `paint_operations`：保持插入顺序，用于 `replay` 操作
- 类型分离的 `Vec`：用于 GPU 批量提交

```rust
// 清空的场景
scene.clear();

// 插入原始图元
scene.insert_primitive(Quad {
    order: 0,
    border_style: BorderStyle::Solid,
    bounds: Bounds::new(point(px(0.), px(0.)), size(px(100.), px(50.))),
    content_mask: ContentMask { bounds: Bounds::max_sized() },
    background: Background::Solid(gpui::rgb(0xFF, 0x00, 0x00)),
    border_color: gpui::black(),
    corner_radii: Corners::all(px(8.)),
    border_widths: Edges::all(px(1.)),
});

// 图层管理
scene.push_layer(Bounds::new(point(px(0.), px(0.)), size(px(200.), px(200.))));
scene.insert_primitive(/* 图元 */);
scene.pop_layer();

// 完成收集后排序
scene.finish();
```

### PaintOperation

```rust
pub enum PaintOperation {
    Primitive(Primitive),
    StartLayer(Bounds<ScaledPixels>),
    EndLayer,
}
```

每个 `insert_primitive` 会向 `paint_operations` 追加一条记录。这使得场景可以回放：

```rust
// 从旧场景重播部分操作
scene.replay(0..prev_scene.len(), &prev_scene);
```

### DrawOrder 排序

`DrawOrder` 是一个 `u32` 类型：

```rust
pub type DrawOrder = u32;
```

通过 `BoundsTree` 空间索引分配：

```rust
// 插入图元时自动分配 order
let order = self.primitive_bounds.insert(clipped_bounds);

// 根据 layer 栈确定 order
let order = self.layer_stack.last().copied()
    .unwrap_or_else(|| self.primitive_bounds.insert(clipped_bounds));
```

`finish()` 按 order 排序所有图元类型：

```rust
pub fn finish(&mut self) {
    self.shadows.sort_by_key(|s| s.order);
    self.quads.sort_by_key(|q| q.order);
    self.paths.sort_by_key(|p| p.order);
    self.underlines.sort_by_key(|u| u.order);
    self.monochrome_sprites.sort_by_key(|s| (s.order, s.tile.tile_id));
    self.subpixel_sprites.sort_by_key(|s| (s.order, s.tile.tile_id));
    self.polychrome_sprites.sort_by_key(|s| (s.order, s.tile.tile_id));
    self.surfaces.sort_by_key(|s| s.order);
}
```

Sprite 排序时还会按 `tile_id` 子排序——这使相同 Atlas 纹理的 sprite 能批量提交。

## 51.2 Primitive 枚举

```rust
pub enum Primitive {
    Shadow(Shadow),
    Quad(Quad),
    Path<Path<ScaledPixels>>,
    Underline(Underline),
    MonochromeSprite(MonochromeSprite),
    SubpixelSprite(SubpixelSprite),
    PolychromeSprite(PolychromeSprite),
    Surface(PaintSurface),
}
```

### Quad——矩形

```rust
#[repr(C)]
pub struct Quad {
    pub order: DrawOrder,
    pub border_style: BorderStyle,
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub background: Background,
    pub border_color: Hsla,
    pub corner_radii: Corners<ScaledPixels>,
    pub border_widths: Edges<ScaledPixels>,
}
```

圆角矩形、带边框的矩形都用 `Quad` 表示。`Borderstyle` 枚举：

```rust
pub enum BorderStyle {
    Solid = 0,
    Dashed = 1,
}
```

### Shadow——阴影

```rust
#[repr(C)]
pub struct Shadow {
    pub order: DrawOrder,
    pub blur_radius: ScaledPixels,
    pub bounds: Bounds<ScaledPixels>,
    pub corner_radii: Corners<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub color: Hsla,
}
```

阴影在 GPU 着色器中通过扩展边界 + 高斯模糊实现。

### Path——路径

```rust
pub struct Path<P: Clone + Debug + Default + PartialEq> {
    pub id: PathId,
    pub order: DrawOrder,
    pub bounds: Bounds<P>,
    pub content_mask: ContentMask<P>,
    pub vertices: Vec<PathVertex<P>>,
    pub color: Background,
    start: Point<P>,
    current: Point<P>,
    contour_count: usize,
}
```

路径构建 API：

```rust
use gpui::{Path, Pixels, point, px};

let mut path = Path::new(point(px(0.), px(0.)));
path.move_to(point(px(50.), px(0.)));
path.line_to(point(px(50.), px(50.)));
path.curve_to(
    point(px(100.), px(100.)),  // to
    point(px(75.), px(50.)),    // control
);

// 缩放为 ScaledPixels
let scaled = path.scale(window.scale_factor());
```

路径使用三角形扇分解。`curve_to` 将二次贝塞尔曲线拆分为两个三角形，GPU 着色器用纹理坐标做曲线插值。

### Underline——下划线

```rust
#[repr(C)]
pub struct Underline {
    pub order: DrawOrder,
    pub pad: u32,
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub color: Hsla,
    pub thickness: ScaledPixels,
    pub wavy: u32,  // 0 = 直线, 1 = 波浪线
}
```

### Sprite 三种类型

```rust
// 单色 sprite：glyph、图标
pub struct MonochromeSprite {
    pub order: DrawOrder,
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub color: Hsla,         // 着色颜色
    pub tile: AtlasTile,     // Atlas 纹理中的位置
    pub transformation: TransformationMatrix,
}

// 子像素 sprite：亚像素抗锯齿的文本 glyph
pub struct SubpixelSprite {
    pub order: DrawOrder,
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub color: Hsla,
    pub tile: AtlasTile,
    pub transformation: TransformationMatrix,
}

// 彩色 sprite：emoji、图片
pub struct PolychromeSprite {
    pub order: DrawOrder,
    pub pad: u32,
    pub grayscale: bool,
    pub opacity: f32,
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub corner_radii: Corners<ScaledPixels>,
    pub tile: AtlasTile,
}
```

### Surface——平台表面

macOS 特有的 `PaintSurface` 用于 `CVPixelBuffer` 合成：

```rust
pub struct PaintSurface {
    pub order: DrawOrder,
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    #[cfg(target_os = "macos")]
    pub image_buffer: core_video::pixel_buffer::CVPixelBuffer,
}
```

用于视频渲染、外部窗口嵌入等场景。

## 51.3 PrimitiveBatch GPU 批处理

`Scene::batches()` 返回一个迭代器，将图元合并为批次：

```rust
pub enum PrimitiveBatch {
    Shadows(Range<usize>),
    Quads(Range<usize>),
    Paths(Range<usize>),
    Underlines(Range<usize>),
    MonochromeSprites { texture_id: AtlasTextureId, range: Range<usize> },
    SubpixelSprites { texture_id: AtlasTextureId, range: Range<usize> },
    PolychromeSprites { texture_id: AtlasTextureId, range: Range<usize> },
    Surfaces(Range<usize>),
}
```

批次合并规则：
- 相同 `PrimitiveKind` 的连续图元
- order 不超过下一类图元的最小 order
- Sprite 批次还要求相同的 `texture_id`

```rust
// 迭代所有批次
for batch in scene.batches() {
    match batch {
        PrimitiveBatch::Quads(range) => {
            let quads = &scene.quads[range];
            // 一次性提交所有 quad 到 GPU
            gpu.draw_quads(quads)?;
        }
        PrimitiveBatch::MonochromeSprites { texture_id, range } => {
            let sprites = &scene.monochrome_sprites[range];
            // 绑定 Atlas 纹理，批量绘制
            gpu.draw_sprites(texture_id, sprites)?;
        }
        _ => {}
    }
}
```

`BatchIterator` 使用多路归并算法：

```rust
// 取所有类型迭代器的 peek，按 (order, kind) 排序
let mut orders_and_kinds = [
    (shadows_iter.peek().map(|s| s.order), PrimitiveKind::Shadow),
    (quads_iter.peek().map(|q| q.order), PrimitiveKind::Quad),
    // ...
];
orders_and_kinds.sort_by_key(|(order, kind)| (order.unwrap_or(u32::MAX), *kind));
```

## 51.4 TransformationMatrix 2D 变换

```rust
#[repr(C)]
pub struct TransformationMatrix {
    pub rotation_scale: [[f32; 2]; 2],  // 2x2 旋转+缩放矩阵
    pub translation: [f32; 2],          // 平移向量
}
```

```rust
use gpui::TransformationMatrix;

// 单位矩阵（无变换）
let identity = TransformationMatrix::unit();

// 链式变换
let transform = TransformationMatrix::unit()
    .translate(point(px(100.), px(50.)))
    .rotate(gpui::radians(std::f32::consts::PI / 4.0))
    .scale(size(2.0, 1.5));

// 应用到点
let point = transform.apply(point(px(10.), px(20.)));
```

矩阵乘法（compose）顺序：先 `other` 后 `self`：

```rust
pub fn compose(self, other: TransformationMatrix) -> TransformationMatrix {
    // self(other(point))
}
```

## 51.5 Bounds tree 空间索引

`BoundsTree` 是场景图的空间索引结构：

```rust
primitive_bounds: BoundsTree<ScaledPixels>,
```

用途：
1. **剔除**：快速判断图元是否在视口内
2. **命中检测**：查找某个点覆盖了哪些图元
3. **DrawOrder 分配**：插入边界时返回排序 key

```rust
// 插入边界，返回 draw order
let order = scene.primitive_bounds.insert(bounds);

// 查询与给定边界相交的条目
for entry in scene.primitive_bounds.query(&query_bounds) {
    // 处理相交的图元
}
```

`BoundsTree` 是四叉树的变体，针对 2D 空间查询优化。在大量图元的场景中，它比线性扫描快几个数量级。

### 空边界裁剪

```rust
pub fn insert_primitive(&mut self, primitive: impl Into<Primitive>) {
    let clipped_bounds = primitive.bounds().intersect(&primitive.content_mask().bounds);
    if clipped_bounds.is_empty() {
        return;  // 完全裁剪，不插入
    }
    // ...
}
```

这避免了为零尺寸图元分配 draw order。
