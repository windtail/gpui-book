# 第 49 章：文本塑形

文本塑形是将 Unicode 文本转换为带有精确位置的 `ShapedGlyph` 集合的过程。GPUI 将文本系统分为两层：全局 `TextSystem` 和每窗口的 `WindowTextSystem`。

## 49.1 TextSystem 架构

### 全局 TextSystem

`TextSystem` 是应用级的，负责：
- 字体加载和 `FontId` 分配
- 字体度量查询
- Glyph 栅格化（将矢量轮廓转为位图）
- `LineWrapper` 对象池

```rust
pub struct TextSystem {
    platform_text_system: Arc<dyn PlatformTextSystem>,
    font_ids_by_font: RwLock<FxHashMap<Font, Result<FontId>>>,
    font_metrics: RwLock<FxHashMap<FontId, FontMetrics>>,
    raster_bounds: RwLock<FxHashMap<RenderGlyphParams, Bounds<DevicePixels>>>,
    wrapper_pool: Mutex<FxHashMap<FontIdWithSize, Vec<LineWrapper>>>,
    font_runs_pool: Mutex<Vec<Vec<FontRun>>>,
    fallback_font_stack: SmallVec<[Font; 2]>,
}
```

创建方式：

```rust
use gpui::{TextSystem, PlatformTextSystem};
use std::sync::Arc;

let platform_text: Arc<dyn PlatformTextSystem> = /* 平台实现 */;
let text_system = TextSystem::new(platform_text);
```

### WindowTextSystem 每窗口实例

`WindowTextSystem` 在每个窗口创建时建立，持有行布局缓存：

```rust
pub struct WindowTextSystem {
    line_layout_cache: LineLayoutCache,
    text_system: Arc<TextSystem>,
}
```

```rust
// 在窗口初始化时创建
let window_text_system = WindowTextSystem::new(cx.text_system().clone());

// 塑形单行文本
let shaped = window_text_system.shape_line(
    "Hello, GPUI!".into(),
    gpui::px(16.0),
    &[TextRun {
        len: 14,
        font: font("Helvetica"),
        color: gpui::white(),
        background_color: None,
        underline: None,
        strikethrough: None,
    }],
    None, // force_width
);

// 塑形多行文本
let lines = window_text_system.shape_text(
    "line 1\nline 2\nline 3".into(),
    gpui::px(16.0),
    &[/* TextRun 数组 */],
    Some(gpui::px(300.0)), // wrap_width
    None,                  // line_clamp
)?;
```

每帧结束时清理缓存：

```rust
window_text_system.finish_frame();
// 将 current_frame 和 previous_frame 交换
// previous_frame 的缓存可以在下一帧被复用
```

## 49.2 ShapedGlyph 与 ShapedRun

塑形结果由 `LineLayout` 承载：

```rust
pub struct LineLayout {
    pub font_size: Pixels,
    pub width: Pixels,
    pub ascent: Pixels,
    pub descent: Pixels,
    pub runs: Vec<ShapedRun>,
    pub len: usize, // UTF-8 字节长度
}

pub struct ShapedRun {
    pub font_id: FontId,
    pub glyphs: Vec<ShapedGlyph>,
}

pub struct ShapedGlyph {
    pub id: GlyphId,
    pub position: Point<Pixels>,
    pub index: usize,   // 原文本中的 UTF-8 字节索引
    pub is_emoji: bool,
}
```

使用示例：

```rust
let layout = window_text_system.layout_line(
    "GPUI",
    gpui::px(16.0),
    &[TextRun {
        len: 4,
        font: font("Helvetica"),
        color: gpui::white(),
        background_color: None,
        underline: None,
        strikethrough: None,
    }],
    None,
);

for run in &layout.runs {
    for glyph in &run.glyphs {
        println!(
            "Glyph {:?} at ({}, {}), byte index {}",
            glyph.id, glyph.position.x, glyph.position.y, glyph.index
        );
    }
}
```

### 子像素渲染变体

GPUI 为亚像素抗锯齿预计算 glyph 变体：

```rust
/// X 轴子像素变体数量
pub const SUBPIXEL_VARIANTS_X: u8 = 4;
/// Y 轴子像素变体数量
pub const SUBPIXEL_VARIANTS_Y: u8 = 1;
```

每个 glyph 根据其在屏幕上的亚像素位置选择 4 个预光栅化变体之一。这比实时亚像素采样质量更高。

## 49.3 Glyph 栅格化

### RenderGlyphParams

```rust
pub struct RenderGlyphParams {
    pub font_id: FontId,
    pub glyph_id: GlyphId,
    pub font_size: Pixels,
    pub subpixel_variant: Point<u8>,    // (0..4, 0)
    pub scale_factor: f32,
    pub is_emoji: bool,
    pub subpixel_rendering: bool,
}
```

### Atlas 纹理

Glyph 位图被分配到 Atlas 纹理中。GPUI 使用纹理池管理：

```rust
// 栅格化参数用作缓存键
let params = RenderGlyphParams {
    font_id,
    glyph_id: GlyphId(42),
    font_size: gpui::px(16.0),
    subpixel_variant: gpui::point(2u8, 0u8),
    scale_factor: 1.0,
    is_emoji: false,
    subpixel_rendering: true,
};

// 查询缓存的栅格边界
let bounds = text_system.raster_bounds(&params)?;

// 栅格化 glyph（缓存未命中时调用平台实现）
let (size, bitmap) = text_system.rasterize_glyph(&params)?;
```

Atlas 纹理由 GPU 后端的 `GpuJustification` 系统管理。纹理大小动态增长，满了后创建新的 Atlas。

### 渲染模式选择

平台推荐不同的渲染模式：

```rust
pub fn recommended_rendering_mode(
    &self,
    font_id: FontId,
    font_size: Pixels,
) -> TextRenderingMode {
    self.platform_text_system
        .recommended_rendering_mode(font_id, font_size)
}
```

macOS 对较大字号使用亚像素渲染，对较小字号关闭。Linux (wgpu) 通常不使用亚像素渲染。

## 49.4 ShapedLine 绘制

`ShapedLine` 是可以直接绘制的结果：

```rust
let shaped_line = window_text_system.shape_line(
    "Hello".into(),
    gpui::px(16.0),
    &[TextRun {
        len: 5,
        font: font("Helvetica"),
        color: gpui::white(),
        ..Default::default()
    }],
    None,
);

// 在 paint 阶段绘制
shaped_line.paint(
    gpui::point(gpui::px(10.0), gpui::px(20.0)), // origin
    gpui::px(24.0),                               // line_height
    TextAlign::Left,
    None,                                         // align_width
    window,
    cx,
)?;
```

### 预计算光栅数据

在 prepaint 阶段可以预计算 glyph 光栅数据，避免 paint 阶段查找：

```rust
use gpui::text::GlyphRasterData;

// prepaint 中预计算
let raster_data = shaped_line.compute_glyph_raster_data(
    origin,
    line_height,
    TextAlign::Left,
    None,
    window,
    cx,
)?;

// paint 中使用预计算数据
shaped_line.paint_with_raster_data(
    &raster_data,
    window,
    cx,
)?;
```

这避免了 paint 阶段对每个 glyph 做哈希查找，对长文本段落有明显性能提升。

## 49.5 TextRun 多字体混合

`TextRun` 允许在同一段文本中混合不同样式：

```rust
let runs = vec![
    TextRun {
        len: 6,                              // "Hello " 的 UTF-8 字节长度
        font: font("Helvetica"),
        color: gpui::white(),
        background_color: None,
        underline: None,
        strikethrough: None,
    },
    TextRun {
        len: 4,                              // "GPUI"
        font: font("Helvetica").bold(),
        color: gpui::rgb(0x3B, 0x82, 0xF6),  // 蓝色
        background_color: None,
        underline: None,
        strikethrough: None,
    },
];

let shaped = window_text_system.shape_line(
    "Hello GPUI".into(),
    gpui::px(16.0),
    &runs,
    None,
);
```

`WindowTextSystem` 内部会将 `TextRun` 合并为更少的 `FontRun`——相邻且字体相同的 run 会合并。

## 49.6 文本测量性能

`TextSystem` 做了多层缓存优化：

1. **FontId 缓存**：相同 `Font` 描述复用 `FontId`
2. **FontMetrics 缓存**：每个 `FontId` 的度量值只查询一次
3. **行布局缓存**：`LineLayoutCache` 使用 current/previous 帧双缓存
4. **LineWrapper 对象池**：避免频繁分配
5. **FontRun 数组池**：`font_runs_pool` 复用 `Vec`
6. **Glyph 光栅缓存**：相同 `RenderGlyphParams` 复用光栅结果

测量单字符宽度时也有快捷路径：

```rust
// LineWrapper 内部缓存
cached_ascii_char_widths: [Option<Pixels>; 128],  // ASCII 直接索引
cached_other_char_widths: HashMap<char, Pixels>,  // 其他字符哈希表

#[inline(always)]
fn width_for_char(&mut self, c: char) -> Pixels {
    if (c as u32) < 128 {
        // ASCII 路径：直接数组查找
        if let Some(width) = self.cached_ascii_char_widths[c as usize] {
            return width;
        }
        let width = self.text_system.layout_width(self.font_id, self.font_size, c);
        self.cached_ascii_char_widths[c as usize] = Some(width);
        width
    } else {
        // 非 ASCII：HashMap 查找
        if let Some(width) = self.cached_other_char_widths.get(&c) {
            return *width;
        }
        let width = self.text_system.layout_width(self.font_id, self.font_size, c);
        self.cached_other_char_widths.insert(c, width);
        width
    }
}
```
