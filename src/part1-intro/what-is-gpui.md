# 第 1 章：什么是 GPUI

## 1.1 从 Zed 编辑器说起

GPUI 是 [Zed](https://zed.dev) 编辑器的渲染引擎。Zed 的开发者在构建编辑器时发现了一个问题：现有的 GUI 框架无法满足他们对性能的需求。

Electron 应用内存占用高，启动慢。egui 和 Iced 等 Rust GUI 框架适合工具类应用，但无法胜任编辑器级别的复杂交互——语法高亮、大量文本渲染、低延迟输入、GPU 加速滚动。

于是他们自己做了一个。这个引擎最初只是 Zed 的一部分，后来被抽离为独立的 crate：`gpui`。

```rust
// Zed 的 main.rs 入口（简化版）
fn main() {
    let app = Application::new();
    app.run(|cx| {
        // 创建主窗口
        cx.open_window(WindowOptions::default(), |window, cx| {
            cx.new(|cx| Workspace::new(None, window, cx))
        });
    });
}
```

一个 GPUI 应用的核心就是这个模式：创建 `Application`，调用 `run`，在回调中打开窗口并创建根 View。

## 1.2 GPUI 的核心设计理念

### 1.2.1 GPU 优先渲染

GPUI 不做 CPU 端的光栅化。所有绘制操作被收集到一个 `Scene` 结构中，包含以下原始类型：

```rust
// gpui/src/scene.rs - Scene 中的图元类型
pub enum Primitive {
    Shadow,         // 阴影
    Quad,           // 矩形（背景、边框）
    Path,           // 自定义路径
    Underline,      // 下划线
    MonochromeSprite, // 单色贴图（图标、文字）
    SubpixelSprite,   // 子像素贴图（ClearType 文字）
    PolychromeSprite, // 彩色贴图（图片、SVG）
    Surface,          // 平台表面（视频帧）
}
```

每一帧，GPUI 将 `Scene` 中的图元通过各平台的原生图形 API 提交到 GPU：macOS 使用 Metal（`gpui_macos` crate），Linux 使用 wgpu（`gpui_wgpu` crate），Windows 使用 DirectX 11（`gpui_windows` crate）。这保证了即使界面极其复杂，渲染也能维持 60fps。

```rust
// 自定义 Canvas 元素的绘制回调
div().child(
    canvas(
        |_bounds, window, cx| { /* prepaint */ },
        |bounds, window, cx| {
            // 这里的绘制操作直接进入 GPU Scene
            window.paint_quad(quad(bounds, bg));
            window.paint_path(path);
        },
    )
    .size(px(200.))
)
```

### 1.2.2 混合模式：声明式 View + 命令式 Element

GPUI 没有选择单一的渲染模式。它结合了两者的优点：

**View 层是声明式的。** 你描述 UI 应该长什么样：

```rust
struct Counter {
    count: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .items_center()
            .justify_center()
            .size_full()
            .child(format!("Count: {}", self.count))
            .child(
                button("increment", "Increment")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.count += 1;
                        cx.notify();
                    })),
            )
    }
}
```

**Element 层是命令式的。** 当需要精细控制时，你直接控制布局、预绘制、绘制三个阶段：

```rust
struct ProgressBar {
    progress: f32, // 0.0 ~ 1.0
}

impl Element for ProgressBar {
    type RequestLayoutState = ();
    type PrepaintState = ();

    fn request_layout(&mut self, _id: Option<&GlobalElementId>, window: &mut Window, cx: &mut App)
        -> (LayoutId, ())
    {
        let layout_id = window.request_layout(
            Style::default(),
            None, // 没有子元素
            cx,
        );
        (layout_id, ())
    }

    fn prepaint(&mut self, _global_id: Option<&GlobalElementId>, bounds: Bounds<Pixels>,
        _: &mut Self::RequestLayoutState, window: &mut Window, cx: &mut App)
    {
        // 不需要预绘制
    }

    fn paint(&mut self, _global_id: Option<&GlobalElementId>, bounds: Bounds<Pixels>,
        _: &mut Self::RequestLayoutState, _: &mut Self::PrepaintState,
        window: &mut Window, cx: &mut App)
    {
        // 直接绘制进度条
        let filled = Bounds::new(bounds.origin, size(bounds.size.width * self.progress, bounds.size.height));
        window.paint_quad(filled_quad(filled, green()));
    }
}
```

大多数情况下你只需要写 View 层。自定义 Element 是为那些 div 无法表达的场景准备的——图表、自定义滚动条、游戏画面等。

### 1.2.3 单一所有者：Entity 模型

GPUI 的状态管理遵循一个原则：**App 拥有所有状态**。

你永远不会直接持有某个 View 或 Entity 的可变引用。取而代之的是 `Entity<T>` 句柄：

```rust
// Entity 是 GPUI 的智能指针
struct AppState {
    current_file: Option<PathBuf>,
    unsaved_changes: bool,
}

// 创建 Entity
let app_state: Entity<AppState> = cx.new(|cx| AppState {
    current_file: None,
    unsaved_changes: false,
});

// 只读访问
app_state.read(cx, |state, _| {
    println!("file: {:?}", state.current_file);
});

// 可变访问
app_state.update(cx, |state, cx| {
    state.current_file = Some(PathBuf::from("main.rs"));
    state.unsaved_changes = true;
    cx.notify(); // 通知观察者
});
```

`Entity<T>` 内部使用引用计数和运行时借用检查。它类似于 `Rc<RefCell<T>>`，但有三个关键区别：

1. **运行时借用检查**：同时存在多个只读引用是安全的，但两个可变引用会 panic
2. **弱引用**：`WeakEntity<T>` 不会阻止 Entity 被释放
3. **类型擦除**：`AnyEntity` 可以跨类型存储

这就是 GPUI 的"三注册"概念的基础：Entity（数据）、View（状态）、Element（渲染）各司其职，通过 App 统一调度。

## 1.3 GPUI vs 其他 Rust GUI 框架

### 1.3.1 与 egui 对比（纯立即模式）

egui 是纯立即模式框架。每一帧都重新构建整个 UI：

```rust
// egui - 每帧执行
egui::Window::new("My Window").show(ctx, |ui| {
    ui.add(egui::Label::new("Hello"));
});
```

优点：代码简单，状态自然就是局部变量。
缺点：每帧重建，不适合复杂布局或大量控件。

GPUI 采用混合模式。View 只在数据变化时重新渲染：

```rust
// GPUI - 只在 cx.notify() 时重新渲染
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(format!("Count: {}", self.count))
    }
}
```

### 1.3.2 与 Iced 对比（Elm 架构）

Iced 使用 Elm 架构：Message -> update -> View：

```rust
// Iced - 消息驱动
fn update(model: &mut Model, message: Message) {
    match message {
        Message::Increment => model.count += 1,
    }
}

fn view(model: &Model) -> Element<Message> {
    column![button("+").on_press(Message::Increment)]
}
```

优点：状态流清晰，容易测试。
缺点：间接层多，消息类型随着应用增长变得臃肿。

GPUI 使用 Entity + 回调模式，更直接：

```rust
// GPUI - 直接回调
button("increment", "+")
    .on_click(cx.listener(|this, _, _, cx| {
        this.count += 1;  // 直接修改状态
        cx.notify();       // 触发重新渲染
    }))
```

### 1.3.3 与 Slint 对比（DSL + 嵌入式）

Slint 使用自定义 DSL（.slint 文件），适合嵌入式系统和 QML 风格的开发：

```slint
// Slint DSL
component App {
    counter := 0;
    VerticalLayout {
        Text { text: "Count: \{counter}"; }
        Button {
            text: "Increment";
            clicked => { counter += 1; }
        }
    }
}
```

优点：与 Rust 代码分离，设计者可以参与。
缺点：DSL 学习成本，与 Rust 生态的集成有限。

GPUI 完全使用 Rust 代码，利用 Rust 的类型系统和宏：

```rust
// GPUI - 纯 Rust
div()
    .flex()
    .flex_col()
    .gap_2()
    .child(Text::new(format!("Count: {}", counter)))
    .child(Button::new("increment", "Increment"))
```

### 1.3.4 对比总结

| 特性 | GPUI | egui | Iced | Slint |
|------|------|------|------|-------|
| 渲染模式 | 混合 | 立即 | 保留（Elm） | 保留（DSL） |
| 渲染后端 | GPU | GPU（wgpu） | GPU（wgpu） | 多后端 |
| 状态管理 | Entity | 局部变量 | Message | DSL 属性 |
| 适用场景 | 编辑器、复杂应用 | 工具、调试面板 | 中等复杂度应用 | 嵌入式、IoT |
| 学习曲线 | 中高 | 低 | 中 | 中（DSL） |
| 生态 | Zed 生态 | 独立 | 独立 | 独立 |

## 1.4 GPUI 的成熟度与生态

### 1.4.1 当前状态与已知限制

GPUI 是一个生产级框架——Zed 编辑器每天都在使用它。但它也有一些限制：

- **Windows 支持**：仍在开发中，基础功能可用
- **API 稳定性**：作为内部框架演化而来，API 仍在调整
- **控件库**：没有内置的完整控件库（如表格、树形控件），需要自己组合 `div` 构建
- **无障碍访问**：基础支持存在，但不如成熟 GUI 框架完善

### 1.4.2 平台支持

| 平台 | 状态 | 渲染后端 | 窗口系统 |
|------|------|----------|----------|
| macOS | 完整 | Metal | 原生 |
| Linux | 完整 | wgpu | Wayland / X11 |
| Windows | 开发中 | DirectX 11 | 原生 |

GPUI 的平台后端代码不在 `gpui/src/platform/` 目录中，而是作为独立的 workspace crate：
- `gpui_macos/` — Metal 渲染 (`metal_renderer.rs`)
- `gpui_linux/` — wgpu 渲染，通过 `gpui_wgpu/` crate
- `gpui_windows/` — DirectX 11 渲染 (`directx_renderer.rs`)
- `gpui_wgpu/` — 跨平台 wgpu 渲染器（用于 Linux 和 headless 模式）

Linux 下同时支持 Wayland 和 X11。编译时通过 feature flag 选择：

```toml
[dependencies]
gpui = { version = "0.1", features = ["wayland"] }
# 或
gpui = { version = "0.1", features = ["x11"] }
```

### 1.4.3 Zed 中的应用实例

Zed 的整个界面都是 GPUI 构建的：

- **编辑器区域**：自定义 Element 渲染文本，使用虚拟列表处理大文件
- **侧边栏**：Tree View，使用 Entity 存储文件树状态
- **面板系统**：可拖拽分割的 Pane，每个 Pane 持有不同的 View
- **命令面板**：anchored 弹出层 + UniformList 搜索列表
- **标题栏**：自定义绘制，隐藏原生装饰

查看 Zed 源码中的 `crates/` 目录，每个 crate 都是一个 GPUI View 的集合：

```
crates/
├── editor/          # 核心编辑器组件
├── workspace/       # 窗口和面板管理
├── ui/             # 通用 UI 组件（按钮、标签等）
├── file_finder/    # 文件搜索面板
└── terminal/       # 终端组件
```

## 1.5 阅读本书的预备知识

### 1.5.1 Rust 基础知识

你需要熟悉：

- **所有权和借用**：理解 `&T`、`&mut T`、`Rc`、`RefCell`
- **Trait**：能看懂 `impl Trait for Type` 语法，了解 trait bound
- **闭包**：`Fn`、`FnMut`、`FnOnce` 的区别
- **生命周期**：基础理解即可，GPUI 大量使用运行时借用检查来避开静态生命周期限制
- **宏**：`derive` 宏和过程宏的基本概念

### 1.5.2 异步编程概念

GPUI 内置了 async 支持。你需要了解：

- `Future` 和 `async/await` 语法
- 执行器（executor）的概念
- `tokio` 或 `async-std` 的使用经验

```rust
// GPUI 中的异步加载
cx.spawn(|this, mut cx| async move {
    let data = http_client.get(url).await?;
    this.update(&mut cx, |this, cx| {
        this.data = Some(data);
        cx.notify();
    })
}).detach();
```

### 1.5.3 GUI 编程基本术语

- **事件循环（Event Loop）**：GUI 程序的核心循环，处理输入、更新状态、渲染帧
- **渲染管线（Render Pipeline）**：从 UI 描述到屏幕像素的处理流程
- **布局（Layout）**：计算每个元素的大小和位置
- **即时模式（Immediate Mode）**：每帧重新构建 UI
- **保留模式（Retained Mode）**：UI 结构持久化，只更新变化的部分

## 本章小结

GPUI 是一个 GPU 加速的混合模式 GUI 框架，来自 Zed 编辑器。它的核心设计是：

1. **GPU 优先渲染**：所有绘制走 GPU
2. **混合模式**：View 层声明式，Element 层命令式
3. **Entity 模型**：App 拥有所有状态，通过句柄访问
4. **Action/Keymap**：内置完整的命令和快捷键系统

下一章，我们来搭建开发环境。
