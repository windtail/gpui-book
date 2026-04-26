# 第 52 章：GPU 渲染管线

GPUI 将渲染从 Scene 到屏幕的过程交给平台特定的 GPU 后端。macOS 使用 Metal，Linux 使用 wgpu，Windows 使用 DirectX 11。

## 52.1 从 Scene 到屏幕

渲染流程概览：

```
Scene（绘制操作集合）
  │
  ▼
batches() ──→ PrimitiveBatch（批次分组）
  │
  ▼
GPU Backend（Metal / wgpu / DirectX）
  │
  ├─ Atlas 纹理上传（glyph / image）
  ├─ Quad 渲染（矩形、圆角、边框）
  ├─ Sprite 渲染（glyph、emoji）
  ├─ Path 渲染（贝塞尔曲线）
  └─ Surface 合成（视频等外部纹理）
  │
  ▼
Frame 提交到显示服务器
```

### Window 中的渲染入口

```rust
// window.rs 中
fn draw(&mut self) -> Result<Scene> {
    let mut scene = Scene::default();

    // 遍历元素树，收集 paint 操作
    self.root.paint(&mut scene, self, cx)?;

    scene.finish(); // 排序
    Ok(scene)
}
```

平台窗口接收 scene 并提交给 GPU：

```rust
// platform/mac.rs 或 platform/linux.rs
trait PlatformWindow {
    fn draw(&self, scene: &Scene, fps: Option<f32>) -> Result<()>;
}
```

### 着色器

GPUI 为不同类型的图元使用不同的着色器：

```wgsl
// Quad 着色器 (Metal 示意)
// 顶点着色器：将 bounds 转为两个三角形
// 片元着色器：
//   - 检查圆角裁剪
//   - 检查边框宽度
//   - 输出背景色或边框色
//
// Sprite 着色器：
//   - 从 Atlas 纹理采样
//   - Monochrome: alpha * color
//   - Subpixel: 3-channel RGB 亚像素混合
//   - Polychrome: 直接 RGBA 采样
//
// Path 着色器：
//   - 用纹理坐标做贝塞尔曲线插值
//   - 边缘抗锯齿
```

Metal 着色器在 `crates/gpui/src/platform/mac/shaders.metal` 中。wgpu 使用 WGSL 着色器。

## 52.2 Atlas 纹理

### Glyph Atlas

文本 glyph 被光栅化到共享 Atlas 纹理中：

```rust
// Atlas 纹理由 GpuAtlas 管理
struct GpuAtlas {
    textures: Vec<Arc<GpuTexture>>,
    // ...
}
```

每个 Atlas tile 记录：

```rust
pub struct AtlasTile {
    pub texture_id: AtlasTextureId,
    pub tile_id: TileId,
    // 纹理坐标
    // ...
}
```

### Atlas 分配流程

```
1. glyph 光栅化请求
   ↓
2. 检查缓存：RenderGlyphParams 已存在？
   ↓ 否
3. PlatformTextSystem::rasterize_glyph()
   ↓
4. 分配 Atlas tile
   ↓ 纹理满？
5. 创建新 Atlas 纹理
   ↓
6. 上传 bitmap 到 GPU
   ↓
7. 返回 AtlasTile（纹理 ID + 坐标）
```

### Atlas 生命周期管理

Atlas 纹理不会永久保留。GPUI 使用引用计数：

```rust
// 当没有 sprite 引用某个 Atlas 纹理时
// 该纹理可以被回收
```

### Image Atlas

图片也会进入 Atlas（小型图片）：

```rust
// 小图片分配到 Atlas
// 大图片使用独立纹理
```

## 52.3 Sprite 渲染

### MonochromeSprite

用于文本 glyph 和单色图标：

```rust
// GPU 顶点数据
struct SpriteVertex {
    position: [f32; 2],     // 屏幕坐标
    tex_coord: [f32; 2],    // Atlas 纹理坐标
    color: [f32; 4],        // HSLA → RGBA
}

// 片元着色器
// color.a = alpha_from_atlas * vertex_color.a
// color.rgb = vertex_color.rgb
```

绘制时绑定：
1. Atlas 纹理（sampler）
2. Sprite 顶点缓冲区
3. 统一的投影矩阵

### SubpixelSprite

亚像素抗锯齿使用 3 通道采样：

```rust
// 每个 glyph 有 4 个子像素变体 (SUBPIXEL_VARIANTS_X = 4)
// subpixel_variant: Point<u8> = (0..4, 0)

// GPU 着色器：
// 1. 根据 glyph 的 subpixel 位置选择正确的变体
// 2. RGB 三个通道独立采样
// 3. 与背景颜色混合

// 这利用了 LCD 子像素的物理排列
// 水平方向的有效分辨率提高 3 倍
```

macOS 推荐亚像素渲染，但现代高分屏上效果不明显。Linux (wgpu) 通常关闭。

### PolychromeSprite

用于 emoji 和完整颜色图片：

```rust
pub struct PolychromeSprite {
    pub order: DrawOrder,
    pub grayscale: bool,       // 灰度模式
    pub opacity: f32,          // 整体透明度
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub corner_radii: Corners<ScaledPixels>,
    pub tile: AtlasTile,
}
```

圆角在着色器中通过检查距离实现：

```glsl
// 片元着色器伪代码
float dist = distance_to_rounded_rect(uv, corner_radii);
if (dist > 0.0) {
    alpha = 1.0 - smoothstep(0.0, 1.0, dist);
}
```

## 52.4 Surface 合成

### macOS CoreVideo

`PaintSurface` 将外部 `CVPixelBuffer` 合成到 GPUI 场景：

```rust
#[cfg(target_os = "macos")]
pub struct PaintSurface {
    pub order: DrawOrder,
    pub bounds: Bounds<ScaledPixels>,
    pub content_mask: ContentMask<ScaledPixels>,
    pub image_buffer: core_video::pixel_buffer::CVPixelBuffer,
}
```

用于：
- 视频播放
- 屏幕共享
- 外部窗口嵌入

Metal 渲染流程：

```rust
// 1. 从 CVPixelBuffer 创建 MTLLibrary-compatible texture
// 2. 在 render pass 中作为独立纹理采样
// 3. 应用 content_mask 裁剪
```

### wgpu/Metal/DirectX 合成

两种后端都遵循相同模式：

1. Scene 被转换为批次
2. 每个批次绑定对应的 pipeline state
3. 执行 draw call
4. 所有绘制完成后 present

```rust
// 渲染一个完整帧
fn render_frame(&mut self, scene: &Scene) -> Result<()> {
    let mut encoder = self.device.new_command_encoder();

    // 开始 render pass
    let mut pass = encoder.begin_render_pass(&RenderPassDescriptor {
        color_attachments: &[self.current_drawable.texture()],
        depth_attachment: None,
    });

    for batch in scene.batches() {
        match batch {
            PrimitiveBatch::Quads(range) => {
                pass.set_pipeline(&self.quad_pipeline);
                pass.set_vertex_buffer(&scene.quads[range]);
                pass.draw(0..4, 0..count);
            }
            PrimitiveBatch::MonochromeSprites { texture_id, range } => {
                pass.set_pipeline(&self.sprite_pipeline);
                pass.bind_texture(&self.atlas.get(texture_id));
                pass.set_vertex_buffer(&scene.monochrome_sprites[range]);
                pass.draw(0..4, 0..count);
            }
            // ...
        }
    }

    pass.end();
    encoder.commit();
    self.present()?;
    Ok(())
}
```

### Quad 渲染优化

Quad 用两个三角形绘制。圆角在片元着色器中做裁剪检查：

```metal
// 片元着色器
fragment float4 quad_fragment(
    vertex_out in [[stage_in]],
    constant QuadUniforms &uniforms [[buffer(0)]]
) {
    float2 corner_dist = distance_to_corners(in.uv, uniforms.corner_radii);
    float alpha = smooth_alpha(corner_dist, 1.0);

    // 边框
    if (is_border_pixel(in.uv, uniforms.border_widths)) {
        return uniforms.border_color * alpha;
    }

    return uniforms.background_color * alpha;
}
```

### Dashed 边框

虚线边框在 GPU 中用纹理实现：

```
1. 计算像素在边框上的位置
2. 用虚线图案纹理做 alpha 测试
3. 偶数段绘制，奇数段跳过
```
