# 第 54 章：Platform Trait

GPUI 通过 `Platform` trait 抽象所有操作系统差异。每个平台（macOS、Linux）提供各自的实现。

## 54.1 Platform trait 概览

`Platform` trait 定义在 `crates/gpui/src/platform.rs`：

```rust
pub trait Platform: 'static {
    // 执行器
    fn background_executor(&self) -> BackgroundExecutor;
    fn foreground_executor(&self) -> ForegroundExecutor;
    fn text_system(&self) -> Arc<dyn PlatformTextSystem>;

    // 应用生命周期
    fn run(&self, on_finish_launching: Box<dyn 'static + FnOnce()>);
    fn quit(&self);
    fn restart(&self, binary_path: Option<PathBuf>);
    fn activate(&self, ignoring_other_apps: bool);
    fn hide(&self);
    fn hide_other_apps(&self);
    fn unhide_other_apps(&self);

    // 显示器
    fn displays(&self) -> Vec<Rc<dyn PlatformDisplay>>;
    fn primary_display(&self) -> Option<Rc<dyn PlatformDisplay>>;
    fn active_window(&self) -> Option<AnyWindowHandle>;

    // 窗口
    fn open_window(
        &self,
        handle: AnyWindowHandle,
        options: WindowParams,
    ) -> anyhow::Result<Box<dyn PlatformWindow>>;
    fn window_appearance(&self) -> WindowAppearance;

    // URL 和路径
    fn open_url(&self, url: &str);
    fn prompt_for_paths(&self, options: PathPromptOptions)
        -> oneshot::Receiver<Result<Option<Vec<PathBuf>>>>;
    fn prompt_for_new_path(&self, directory: &Path, suggested_name: Option<&str>)
        -> oneshot::Receiver<Result<Option<PathBuf>>>;

    // 系统级操作
    fn set_cursor_style(&self, style: CursorStyle);
    fn should_auto_hide_scrollbars(&self) -> bool;
    fn read_from_clipboard(&self) -> Option<ClipboardItem>;
    fn write_to_clipboard(&self, item: ClipboardItem);

    // 菜单（macOS 特有）
    fn set_menus(&self, menus: Vec<Menu>, keymap: &Keymap);
    fn set_dock_menu(&self, menu: Vec<MenuItem>, keymap: &Keymap);

    // 回调
    fn on_quit(&self, callback: Box<dyn FnMut()>);
    fn on_reopen(&self, callback: Box<dyn FnMut()>);

    // 热状态
    fn thermal_state(&self) -> ThermalState;
    fn on_thermal_state_change(&self, callback: Box<dyn FnMut()>);

    // 键盘
    fn keyboard_layout(&self) -> Box<dyn PlatformKeyboardLayout>;
    fn keyboard_mapper(&self) -> Rc<dyn PlatformKeyboardMapper>;
}
```

### Platform 初始化

```rust
// 应用启动时选择平台实现
#[cfg(target_os = "macos")]
let platform = MacPlatform::new();

#[cfg(any(target_os = "linux", target_os = "freebsd"))]
let platform = LinuxPlatform::new();

let app = App::new_app(platform, asset_source, http_client);
```

### PlatformTextSystem

字体相关的平台抽象：

```rust
pub trait PlatformTextSystem: Send + Sync {
    fn add_fonts(&self, fonts: Vec<Cow<'static, [u8]>>) -> Result<()>;
    fn all_font_names(&self) -> Vec<String>;
    fn font_id(&self, font: &Font) -> Result<FontId>;
    fn font_metrics(&self, font_id: FontId) -> FontMetrics;
    fn typographic_bounds(&self, font_id: FontId, glyph_id: GlyphId) -> Result<Bounds<f32>>;
    fn advance(&self, font_id: FontId, glyph_id: GlyphId) -> Result<Size<f32>>;
    fn glyph_for_char(&self, font_id: FontId, ch: char) -> Option<GlyphId>;
    fn glyph_raster_bounds(&self, params: &RenderGlyphParams) -> Result<Bounds<DevicePixels>>;
    fn rasterize_glyph(
        &self,
        params: &RenderGlyphParams,
        raster_bounds: Bounds<DevicePixels>,
    ) -> Result<(Size<DevicePixels>, Vec<u8>)>;
    fn layout_line(&self, text: &str, font_size: Pixels, runs: &[FontRun]) -> LineLayout;
    fn recommended_rendering_mode(&self, font_id: FontId, font_size: Pixels) -> TextRenderingMode;
}
```

macOS 使用 Core Text，Linux 使用 `cosmic-text`（封装了 fontdb + rustybuzz + swash）。

### PlatformWindow

每个平台的窗口实现：

```rust
pub trait PlatformWindow {
    fn bounds(&self) -> Bounds<Pixels>;
    fn is_fullscreen(&self) -> bool;
    fn content_size(&self) -> Size<Pixels>;
    fn scale_factor(&self) -> f64;
    fn appearance(&self) -> WindowAppearance;
    fn display(&self) -> Rc<dyn PlatformDisplay>;
    fn mouse_position(&self) -> Point<Pixels>;
    fn cursor_style(&self) -> CursorStyle;

    fn draw(&self, scene: &Scene, fps: Option<f32>) -> Result<()>;
    fn swap_buffers(&self) -> Result<()>;
    fn pre_present(&self) {}

    // 事件
    fn on_event(&self, event: Box<dyn PlatformInput>);
    fn on_resize(&self, callback: Box<dyn FnMut()>);
    fn on_moved(&self, callback: Box<dyn FnMut()>);
    fn on_should_close(&self, callback: Box<dyn FnMut()>);
    fn on_close(&self, callback: Box<dyn FnMut()>);
    fn on_appearance_changed(&self, callback: Box<dyn FnMut()>);

    // 输入
    fn set_prompt_for_input_text(&self, prompt: Option<InputPrompt>);
    fn set_input_handler(&mut self, input_handler: PlatformInputHandler);
    fn take_input_handler(&mut self) -> Option<PlatformInputHandler>;
}
```

## 54.2 Feature Flags

GPUI 使用 feature flags 控制平台功能：

```toml
[dependencies.gpui]
features = [
    "wayland",         # Linux Wayland 支持
    "x11",             # Linux X11 支持
    "screen-capture",  # 屏幕录制（需要平台权限）
]
```

条件编译：

```rust
#[cfg(all(target_os = "linux", feature = "wayland"))]
pub mod layer_shell;

#[cfg(feature = "screen-capture")]
pub mod scap_screen_capture;
```

### 运行时检测

```rust
#[cfg(any(target_os = "linux", target_os = "freebsd"))]
pub fn guess_compositor() -> &'static str {
    if std::env::var_os("ZED_HEADLESS").is_some() {
        return "Headless";
    }

    #[cfg(feature = "wayland")]
    let wayland_display = std::env::var_os("WAYLAND_DISPLAY");
    #[cfg(not(feature = "wayland"))]
    let wayland_display: Option<std::ffi::OsString> = None;

    #[cfg(feature = "x11")]
    let x11_display = std::env::var_os("DISPLAY");
    #[cfg(not(feature = "x11"))]
    let x11_display: Option<std::ffi::OsString> = None;

    if wayland_display.is_some_and(|d| !d.is_empty()) {
        "Wayland"
    } else if x11_display.is_some_and(|d| !d.is_empty()) {
        "X11"
    } else {
        "Headless"
    }
}
```

## 54.3 平台能力检测

### ThermalState

检测系统热节流状态：

```rust
pub enum ThermalState {
    Nominal,   // 无限制
    Fair,      // 轻度限制
    Serious,   // 中度限制
    Critical,  // 严重限制
}

// 根据热状态调整行为
match platform.thermal_state() {
    ThermalState::Nominal => { /* 正常 */ }
    ThermalState::Fair => { /* 减少后台工作 */ }
    ThermalState::Serious => { /* 降低动画帧率 */ }
    ThermalState::Critical => { /* 暂停非关键渲染 */ }
}
```

### 滚动条样式

```rust
fn should_auto_hide_scrollbars(&self) -> bool;
```

macOS 返回 `true`（自动隐藏），Linux 返回 `false`（始终显示）。

### 平台特有剪贴板

Linux 有 primary selection（中键粘贴）：

```rust
#[cfg(any(target_os = "linux", target_os = "freebsd"))]
fn read_from_primary(&self) -> Option<ClipboardItem>;

#[cfg(any(target_os = "linux", target_os = "freebsd"))]
fn write_to_primary(&self, item: ClipboardItem);
```

macOS 有 search pasteboard：

```rust
#[cfg(target_os = "macos")]
fn read_from_find_pasteboard(&self) -> Option<ClipboardItem>;

#[cfg(target_os = "macos")]
fn write_to_find_pasteboard(&self, item: ClipboardItem);
```

### 键盘布局

```rust
fn keyboard_layout(&self) -> Box<dyn PlatformKeyboardLayout>;
fn keyboard_mapper(&self) -> Rc<dyn PlatformKeyboardMapper>;
```

平台检测键盘布局变化并映射按键到字符。

### 窗口装饰

Linux 支持服务端和客户端装饰：

```rust
pub enum WindowDecorations {
    Server,  // 窗口管理器提供标题栏
    Client,  // 应用自己绘制标题栏
}

pub enum Decorations {
    Server,
    Client { tiling: Tiling },  // 带平铺状态
}
```

```rust
pub struct Tiling {
    pub top: bool,
    pub bottom: bool,
    pub left: bool,
    pub right: bool,
}
```

检测窗口是否被平铺管理器吸附：

```rust
if let Decorations::Client { tiling } = window.decorations() {
    if tiling.top && tiling.bottom && tiling.left && tiling.right {
        // 窗口被最大化/平铺
        // 隐藏自定义标题栏
    }
}
```
