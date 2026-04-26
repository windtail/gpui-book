# 第 56 章：Linux 后端

GPUI 的 Linux 后端支持 Wayland 和 X11 两种显示协议，使用 wgpu 作为 GPU 后端。

## 56.1 wgpu 渲染

Linux 使用 wgpu API 进行渲染。wgpu 是一个跨平台的图形 API，基于 WebGPU 标准，在 Linux 上底层可以映射到 Vulkan、OpenGL 等驱动。

GPUI 通过 `gpui_wgpu` crate 提供渲染能力，在 Wayland 和 X11 窗口实现中均导入：

```rust
use gpui_wgpu::{WgpuRenderer, WgpuSurfaceConfig};
```

`WgpuRenderer` 负责将 Scene 绘制到窗口，`WgpuSurfaceConfig` 配置 surface 参数：

```rust
// platform/linux/window.rs（Wayland 和 X11 共用）
impl PlatformWindow for LinuxWindow {
    fn draw(&self, scene: &Scene, fps: Option<f32>) -> Result<()> {
        // 1. 配置 wgpu surface
        let config = WgpuSurfaceConfig {
            width: self.bounds.size.width as u32,
            height: self.bounds.size.height as u32,
            // ...
        };

        // 2. 使用 WgpuRenderer 渲染 scene
        self.renderer.draw(scene, &config)?;

        // 3. Present
        self.renderer.present()?;
        Ok(())
    }
}
```

### wgpu 特性

wgpu 提供了跨平台的统一抽象：

```rust
// wgpu 自动选择底层后端（Vulkan/OpenGL 等）
// GPUI 代码层面只与 wgpu API 交互，不直接触碰 Vulkan
```

### GPU 设备初始化

```rust
// wgpu 实例化与设备选择
async fn init_gpu(&self) -> Result<(wgpu::Device, wgpu::Queue)> {
    let instance = wgpu::Instance::default();
    let surface = instance.create_surface(&self.window)?;

    let adapter = instance
        .request_adapter(&wgpu::RequestAdapterOptions {
            compatible_surface: Some(&surface),
            power_preference: wgpu::PowerPreference::HighPerformance,
            ..Default::default()
        })
        .await?;

    let (device, queue) = adapter
        .request_device(&wgpu::DeviceDescriptor::default())
        .await?;

    Ok((device, queue))
}
```

## 56.2 Wayland vs X11

GPUI 通过 feature flags 支持两种协议：

```toml
[dependencies.gpui]
features = ["wayland", "x11"]  # 同时启用
```

### 运行时选择

```rust
pub fn guess_compositor() -> &'static str {
    #[cfg(feature = "wayland")]
    let wayland_display = std::env::var_os("WAYLAND_DISPLAY");
    #[cfg(feature = "x11")]
    let x11_display = std::env::var_os("DISPLAY");

    if wayland_display.is_some_and(|d| !d.is_empty()) {
        "Wayland"
    } else if x11_display.is_some_and(|d| !d.is_empty()) {
        "X11"
    } else {
        "Headless"
    }
}
```

优先级：Wayland > X11 > Headless。

### Wayland

使用 `smithay-client-toolkit` (sctk) + `calloop`：

```rust
#[cfg(feature = "wayland")]
mod wayland {
    use smithay_client_toolkit as sctk;

    // 连接 Wayland compositor
    let conn = Connection::connect_to_env()?;
    let (globals, event_queue) = registry_queue_init(&conn)?;

    // 创建必要的 Wayland 对象
    let compositor = globals.bind(&event_queue, 4..=6, ())?;
    let shm = globals.bind(&event_queue, 1..=1, ())?;
    let xdg_wm_base = globals.bind(&event_queue, 2..=5, ())?;
}
```

### X11

使用 `x11rb`：

```rust
#[cfg(feature = "x11")]
mod x11 {
    use x11rb::connection::Connection;

    let (conn, screen_num) = x11rb::connect(None)?;

    // 创建窗口
    let window = conn.generate_id()?;
    conn.create_window(
        x11rb::COPY_DEPTH_FROM_PARENT,
        window,
        screen.root,
        0, 0, width, height,
        0,
        x11rb::WINDOW_CLASS_INPUT_OUTPUT,
        screen.root_visual,
        &[/* attributes */],
    )?;
}
```

### 差异对比

| 特性 | Wayland | X11 |
|------|---------|-----|
| 合成 | 强制合成 | 可选 |
| 安全 | 窗口隔离 | 所有窗口可互相读取 |
| 缩放 | 每输出缩放 | 全局缩放 |
| 协议 | 现代扩展 | 传统扩展 |
| 平铺 | 原生支持 | 需要外部 WM |

## 56.3 Compositor 检测

### 检测当前运行的合成器

```rust
fn compositor_name(&self) -> &'static str {
    // 返回 "" （默认不检测具体合成器名称）
    // 可以通过环境变量或其他方式检测
}
```

### Layer Shell (Wayland)

Layer Shell 协议允许窗口在普通层之上或之下绘制：

```rust
#[cfg(all(target_os = "linux", feature = "wayland"))]
pub mod layer_shell;
```

Layer Shell 用于：
- 系统托盘
- 通知弹出层
- 状态栏
- 覆盖层

```rust
// 创建 layer surface
use crate::platform::layer_shell::LayerShell;

let layer = LayerShell::new(
    &connection,
    Layer::Top,       // 在最上层
    "my-app-overlay", // namespace
    Some(&output),    // 绑定到特定输出
)?;

layer.set_size(width, height);
layer.set_anchor(Anchor::Top | Anchor::Right);
layer.set_margin(Margin { top: 10, right: 10, ..Default::default() });
```

### 无头模式

```rust
if std::env::var_os("ZED_HEADLESS").is_some() {
    return "Headless";
}
```

无头模式不创建窗口，适用于 CI/CD 和服务器环境。

## 56.4 窗口装饰

### 客户端装饰 vs 服务端装饰

Linux 允许应用选择由谁绘制窗口装饰：

```rust
pub enum WindowDecorations {
    Server,  // 由窗口管理器绘制标题栏
    Client,  // 应用自己绘制
}
```

使用客户端装饰时，需要处理窗口控制按钮：

```rust
// 检测平铺状态
if let Decorations::Client { tiling } = window.decorations() {
    if tiling.top {
        // 窗口被吸附到顶部
        // 隐藏上边框
    }
}
```

### 窗口控制

```rust
pub struct WindowControls {
    pub fullscreen: bool,
    pub maximize: bool,
    pub minimize: bool,
}
```

不同平台支持的控制按钮不同。Wayland compositor 可能不支持最大化。

### 窗口按钮布局

```rust
fn button_layout(&self) -> Option<WindowButtonLayout> {
    // 返回窗口控制按钮的顺序和位置
    // 如: [Close, Minimize, Maximize] 或 [Maximize, Minimize, Close]
}
```

## 56.5 Linux 特有功能

### Primary Selection

Linux 有独立的 primary selection（中键粘贴）：

```rust
fn read_from_primary(&self) -> Option<ClipboardItem>;
fn write_to_primary(&self, item: ClipboardItem);
```

```rust
// 写入 primary selection（选中文本时）
platform.write_to_primary(ClipboardItem::new_string(selected_text));

// 中键粘贴读取
if let Some(item) = platform.read_from_primary() {
    if let Some(text) = item.text() {
        editor.insert_text(text, cx);
    }
}
```

### XDG 门户

文件对话框使用 XDG 桌面门户：

```rust
fn prompt_for_paths(&self, options: PathPromptOptions)
    -> oneshot::Receiver<Result<Option<Vec<PathBuf>>>>;
```

通过 `xdg-desktop-portal` D-Bus API 与宿主桌面环境集成。

### wgpu 与 GPU 选择

```rust
// 通过 wgpu 选择物理 GPU
async fn pick_physical_device(
    instance: &wgpu::Instance,
    surface: &wgpu::Surface,
) -> Result<wgpu::Adapter> {
    let adapter = instance
        .request_adapter(&wgpu::RequestAdapterOptions {
            compatible_surface: Some(surface),
            // 优先选择独立 GPU
            power_preference: wgpu::PowerPreference::HighPerformance,
            ..Default::default()
        })
        .await?;

    let info = adapter.get_info();
    log::info!("Using GPU: {} (type: {:?})", info.name, info.device_type);

    Ok(adapter)
}
```

### 屏幕捕获

通过 `screen-capture` feature：

```rust
#[cfg(feature = "screen-capture")]
pub mod scap_screen_capture;
```

使用 PipeWire + xdg-desktop-portal 在 Wayland 上捕获屏幕。
