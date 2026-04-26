# 第 55 章：macOS 后端

GPUI 的 macOS 后端利用原生 Cocoa + Metal API 提供最佳性能和外观。

## 55.1 Metal 渲染

macOS 使用 Metal 作为 GPU 后端：

```rust
// platform/mac/window.rs 中
impl PlatformWindow for MacWindow {
    fn draw(&self, scene: &Scene, fps: Option<f32>) -> Result<()> {
        // 1. 获取 CAMetalLayer 的 drawable texture
        let drawable = self.view metal_layer().next_drawable()?;

        // 2. 创建 Metal command encoder
        let command_buffer = self.queue.new_command_buffer();
        let encoder = command_buffer.new_render_command_encoder(&render_pass_desc);

        // 3. 遍历 Scene 批次，提交 draw call
        for batch in scene.batches() {
            match batch {
                PrimitiveBatch::Quads(range) => {
                    encoder.set_render_pipeline_state(&self.quad_pipeline_state);
                    encoder.set_vertex_bytes(&scene.quads[range]);
                    encoder.draw_primitives(MTLPrimitiveType::Triangle, 0, count);
                }
                PrimitiveBatch::MonochromeSprites { texture_id, range } => {
                    encoder.set_render_pipeline_state(&self.sprite_pipeline_state);
                    encoder.set_fragment_texture(&self.atlas.texture(texture_id));
                    encoder.set_vertex_bytes(&scene.monochrome_sprites[range]);
                    encoder.draw_primitives(MTLPrimitiveType::TriangleStrip, 0, 4);
                }
                // ...
            }
        }

        // 4. 提交并 present
        encoder.end_encoding();
        command_buffer.present_drawable(&drawable);
        command_buffer.commit();
        Ok(())
    }
}
```

### Metal 着色器

着色器写在 `shaders.metal` 文件中：

```metal
#include <metal_stdlib>

// Quad 顶点输出
struct QuadVertexOut {
    float4 position [[position]];
    float2 corner_radii;
    float4 border_widths;
    float4 background_color;
    float4 border_color;
    float2 uv;
};

// 片元着色器
fragment float4 quad_fragment(
    QuadVertexOut in [[stage_in]]
) {
    // 圆角裁剪
    float2 corner_dist = ...;
    float alpha = smooth_alpha(corner_dist);

    return in.background_color * alpha;
}

// Sprite 片元着色器
fragment float4 sprite_fragment(
    float2 uv [[stage_in]],
    texture2d<float> sprite_texture [[texture(0)]],
    sampler sprite_sampler [[sampler(0)]]
) {
    float4 color = sprite_texture.sample(sprite_sampler, uv);
    return color;
}
```

### Atlas 纹理管理

```rust
// Metal 平台管理多个 Atlas 纹理
struct GpuAtlas {
    textures: Vec<Arc<MetalTexture>>,
    // 当前可写入的纹理
    current: usize,
}
```

## 55.2 原生装饰与菜单栏

### 菜单栏

macOS 有全局菜单栏，GPUI 通过 `set_menus` 设置：

```rust
fn set_menus(&self, menus: Vec<Menu>, keymap: &Keymap);
```

```rust
use gpui::{Menu, MenuItem, actions};

actions!(app, [Quit, About, Preferences]);

fn setup_menus(cx: &mut App) {
    cx.set_menus(vec![
        Menu {
            name: "AppName".into(),
            items: vec![
                MenuItem::action("About AppName", Box::new(About)),
                MenuItem::separator(),
                MenuItem::action("Preferences...", Box::new(Preferences)),
                MenuItem::separator(),
                MenuItem::action("Quit", Box::new(Quit)),
            ],
        },
        Menu {
            name: "Edit".into(),
            items: vec![
                MenuItem::os_action("Undo", os_action::Undo),
                MenuItem::os_action("Redo", os_action::Redo),
                MenuItem::separator(),
                MenuItem::os_action("Cut", os_action::Cut),
                MenuItem::os_action("Copy", os_action::Copy),
                MenuItem::os_action("Paste", os_action::Paste),
            ],
        },
    ]);
}
```

`MenuItem::os_action` 绑定 macOS 标准操作（Undo/Copy/Paste），自动使用标准快捷键。

### Dock 菜单

```rust
fn set_dock_menu(&self, menu: Vec<MenuItem>, keymap: &Keymap);
```

Dock 图标右键菜单。

### 窗口控制

```rust
// macOS 特有的窗口行为
window.set_window_background_appearance(WindowBackgroundAppearance::Opaque);
window.set_has_shadow(true);
window.set_resize_increments(Some(Size::new(px(1.), px(1.))));
```

### 标题栏

自定义标题栏：

```rust
WindowOptions {
    window_bounds: None,
    title_bar: Some(TitleBarOptions {
        title: "My App".into(),
        appears_transparent: true,
        traffic_light_position: Some(point(px(12.), px(16.))),
    }),
    ..Default::default()
}
```

交通灯按钮（关闭/最小化/全屏）位置可自定义。

## 55.3 触控板手势

### PinchEvent（捏合缩放）

```rust
.on_event(EventListener::Pinch, cx.listener(|this, event: &PinchEvent, _, cx| {
    // event.delta: 缩放增量（正数=放大，负数=缩小）
    // 0.1 表示 10% 的增量
    // event.position: 手势中心位置
    this.zoom += event.delta;
    cx.notify();
}))
```

### Magnification

连续放大事件：

```rust
// 在 PlatformWindow 中
fn on_magnify(&self, callback: Box<dyn FnMut()>);
```

### 触控板滚动

```rust
.on_scroll_wheel(cx.listener(|this, event: &ScrollWheelEvent, _, cx| {
    // event.delta: ScrollDelta
    //   - Pixels: 精确像素偏移（触控板）
    //   - Lines: 行数（鼠标滚轮）
    match event.delta {
        ScrollDelta::Pixels(delta) => {
            // 触控板：平滑滚动
            this.scroll_offset += delta;
        }
        ScrollDelta::Lines(delta) => {
            // 鼠标滚轮：按行滚动
            this.scroll_offset += delta.lines_per_row() * line_height;
        }
    }
    cx.notify();
}))
```

### Touch 阶段

```rust
// 触控事件阶段追踪
enum TouchPhase {
    Started,
    Moved,
    Ended,
    Cancelled,
}
```

### 压力感应（Force Touch）

```rust
.on_event(EventListener::MouseDown, cx.listener(|_, event: &MouseDownEvent, _, _| {
    if let Some(pressure) = event.pressure {
        match pressure.stage {
            PressureStage::Light => { /* 轻按 */ }
            PressureStage::Deep => { /* 重按（触觉反馈） */ }
        }
    }
}))
```

## 55.4 其他 macOS 特性

### 拖放文件

```rust
fn on_open_urls(&self, callback: Box<dyn FnMut(Vec<String>)>);
```

Finder 拖放文件到应用图标触发。

### 原生文件对话框

```rust
fn prompt_for_paths(&self, options: PathPromptOptions)
    -> oneshot::Receiver<Result<Option<Vec<PathBuf>>>>;
```

使用 `NSOpenPanel` 提供原生文件选择对话框。

### 最近文档

```rust
fn add_recent_document(&self, path: &Path);
```

添加到 macOS 最近文档列表。

### URL Scheme 注册

```rust
fn register_url_scheme(&self, url: &str) -> Task<Result<()>>;
```

注册自定义 URL scheme（如 `zed://`），使应用可以通过浏览器链接打开。

### Find Pasteboard

macOS 的搜索剪贴板：

```rust
#[cfg(target_os = "macos")]
fn read_from_find_pasteboard(&self) -> Option<ClipboardItem>;

#[cfg(target_os = "macos")]
fn write_to_find_pasteboard(&self, item: ClipboardItem);
```

与其他应用的搜索栏共享搜索文本。
