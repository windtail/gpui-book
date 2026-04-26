# 第 2 章：环境搭建

## 2.1 系统要求

### 2.1.1 Rust 版本与工具链

GPUI 需要较新的 Rust 版本。建议使用最新的 stable 工具链：

```bash
rustup update stable
rustc --version
# rustc 1.85.0 (4d91de4e4 2025-02-17)
```

推荐使用 `rust-analyzer` 作为 IDE 支持。GPUI 大量使用泛型和宏，良好的 IDE 支持能显著提升开发体验。

### 2.1.2 macOS 依赖

macOS 需要安装 Xcode Command Line Tools：

```bash
xcode-select --install
```

GPUI 在 macOS 上使用 Metal 进行渲染，不需要额外的 GPU 驱动安装。

### 2.1.3 Linux 依赖

Linux 需要 wgpu 渲染所需的图形库和窗口系统的开发库。

**Ubuntu / Debian：**

```bash
# Wayland 开发库
sudo apt install libwayland-dev libxkbcommon-dev

# X11 开发库（如需 X11 支持）
sudo apt install libx11-dev libx11-xcb-dev libxcb-render0-dev \
    libxcb-cursor-dev libasound2-dev

# 如果 wgpu 需要 Vulkan 后端（可选，Vulkan 只是 wgpu 的后端之一）
sudo apt install libvulkan1 mesa-vulkan-drivers
```

**Fedora：**

```bash
sudo dnf install wayland-devel libxkbcommon-devel \
    libX11-devel libX11-xcb-devel libxcb-devel alsa-lib-devel \
    mesa-vulkan-drivers
```

**Arch Linux：**

```bash
sudo pacman -S wayland libxkbcommon \
    libx11 libxcb alsa-lib vulkan-icd-loader
```

GPUI 在 Linux 上通过 `gpui_wgpu` crate 使用 wgpu 进行渲染，wgpu 会自动选择后端的图形 API（Vulkan、GL 等），通常不需要单独安装 Vulkan SDK。

## 2.2 Cargo.toml 配置

### 2.2.1 添加 GPUI 依赖

GPUI 已发布到 crates.io，推荐直接通过 cargo 安装：

```toml
[package]
name = "my-gpui-app"
version = "0.1.0"
edition = "2021"

[dependencies]
gpui = "0.2"  # 使用最新版本，或指定具体版本号如 "0.2.0"
```

如果需要跟踪最新的开发版本（可能包含未发布的特性或破坏性变更），可以从 git 引用：

```toml
[dependencies]
gpui = { git = "https://github.com/zed-industries/zed", branch = "main" }
```

或者如果你本地有 Zed 源码：

```toml
[dependencies]
gpui = { path = "../zed/crates/gpui" }
```

### 2.2.2 常用 Feature Flags

```toml
[dependencies]
gpui = {
    version = "0.2",
    features = [
        "wayland",     # Linux Wayland 支持
        # "x11",       # Linux X11 支持
        # "test-support", # 测试支持
    ]
}
```

如果从 git 引用：

```toml
[dependencies]
gpui = {
    git = "https://github.com/zed-industries/zed",
    branch = "main",
    features = [
        "wayland",     # Linux Wayland 支持
        # "x11",       # Linux X11 支持（与 wayland 互斥）
        # "test-support", # 测试支持
        # "inspector",    # Element Inspector
    ]
}
```

Feature flag 说明：

| Feature | 用途 | 默认 |
|---------|------|------|
| `wayland` | Linux Wayland 支持 | 关 |
| `x11` | Linux X11 支持 | 关 |
| `test-support` | 测试工具（`TestAppContext`） | 关 |
| `inspector` | 开发时 Element Inspector | 关 |

### 2.2.3 依赖的传递性

GPUI 依赖了一些 crate，它们会自动被拉入，你不需要手动添加：

```toml
# 这些会被 gpui 自动引入，你不需要手动添加
gpui-util          # 工具函数
gpui-shared-string # 高效字符串
gpui-macros        # 过程宏
gpui-platform      # 平台抽象层
gpui-wgpu          # wgpu 渲染后端
collections        # FxHashMap 等集合
refineable         # 样式细化
```

如果你使用 git 依赖并需要直接依赖其中的某个（例如写测试），可以显式添加：

```toml
[dependencies]
gpui = "0.2"  # 或 git 依赖
gpui-macros = { git = "https://github.com/zed-industries/zed", branch = "main" }
```

## 2.3 项目结构

### 2.3.1 推荐的目录布局

对于小型应用：

```
my-gpui-app/
├── Cargo.toml
└── src/
    └── main.rs
```

对于中等规模应用：

```
my-gpui-app/
├── Cargo.toml
└── src/
    ├── main.rs          # 入口：Application 创建和启动
    ├── app/             # 应用级状态和逻辑
    │   ├── mod.rs
    │   └── state.rs
    ├── views/           # View 定义
    │   ├── mod.rs
    │   ├── main_view.rs
    │   └── sidebar.rs
    └── components/      # 可复用的 UI 组件
        ├── mod.rs
        └── button.rs
```

对于大型应用，可以拆分成多个 crate：

```
my-app/
├── Cargo.toml           # workspace
├── crates/
│   ├── app/             # 主入口
│   ├── core/            # 核心逻辑
│   └── ui/              # 共享组件
└── assets/              # 静态资源
```

### 2.3.2 main.rs 入口约定

一个标准的 `main.rs` 结构：

```rust
use gpui::*;

fn main() {
    let app = Application::new();
    app.run(|cx| {
        setup_menus(cx);
        open_main_window(cx);
    });
}

fn setup_menus(cx: &mut App) {
    // 注册应用菜单和快捷键
}

fn open_main_window(cx: &mut App) {
    cx.open_window(WindowOptions::default(), |window, cx| {
        cx.new(|cx| MyMainView::new(window, cx))
    });
}

struct MyMainView {
    // 状态字段
}

impl MyMainView {
    fn new(window: &mut Window, cx: &mut Context<Self>) -> Self {
        Self {}
    }
}

impl Render for MyMainView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .child("Hello, GPUI!")
    }
}
```

### 2.3.3 模块化建议

- **将 View 拆分为独立文件**：每个 View 一个文件，包含 struct、impl、Render
- **将组件放在 `components/` 目录**：可复用的 UI 片段（按钮变体、输入框等）
- **将状态和逻辑分离**：不要在 View struct 中放业务逻辑，使用单独的 Entity 存储状态
- **使用 `mod.rs` 管理导出**：让上级模块统一导出子模块的公开类型

```rust
// views/mod.rs
mod main_view;
mod sidebar;

pub use main_view::MainView;
pub use sidebar::Sidebar;
```

## 2.4 gpui::prelude 导览

### 2.4.1 为什么需要 prelude

GPUI 有大量的 trait 需要在每个文件中可用。如果不用 prelude，你的文件开头会变成这样：

```rust
use gpui::{
    AppContext, BorrowAppContext, Context, Element, InteractiveElement,
    IntoElement, ParentElement, Refineable, Render, RenderOnce,
    StatefulInteractiveElement, Styled, StyledImage, VisualContext,
};
use gpui::util::FluentBuilder;
```

这很冗长且容易遗漏。prelude 将它们一次性导入。

### 2.4.2 prelude 导入了什么

`gpui::prelude` 导入了以下 13 个 trait：

```rust
// gpui/src/prelude.rs 完整内容
pub use crate::{
    AppContext,           // App 上下文统一接口
    BorrowAppContext,     // 全局状态访问
    Context,              // Entity 专用上下文
    Element,              // 元素渲染协议
    InteractiveElement,   // 可交互元素（点击、悬停等）
    IntoElement,          // 元素转换协议
    ParentElement,        // 容器元素（.child()、.children()）
    Refineable,           // 样式细化
    Render,               // 视图渲染协议
    RenderOnce,           // 无状态组件渲染协议
    StatefulInteractiveElement, // 带状态的交互元素
    Styled,               // 样式协议（.bg()、.flex() 等）
    StyledImage,          // 图片样式
    VisualContext,        // 窗口操作上下文
};
pub use crate::util::FluentBuilder;  // 方法链 fluent API
```

每个 trait 的作用：

| Trait | 用途 | 何时需要 |
|-------|------|----------|
| `AppContext` | 统一的上下文接口，提供 `new()`、`update_entity()` 等方法 | 操作 Entity 时 |
| `Context` | Entity 专用的上下文类型 | 在 Entity 的回调中 |
| `Render` | 将 View 渲染为元素树 | 实现 View 时 |
| `RenderOnce` | 无状态组件 | 编写组件时 |
| `Styled` | 提供 `.bg()`、`.flex()` 等样式方法 | 使用 `div()` 时 |
| `ParentElement` | 提供 `.child()`、`.children()` | 添加子元素时 |
| `InteractiveElement` | 提供 `.on_click()` 等事件方法 | 处理交互时 |
| `IntoElement` | 将类型转换为元素 | 返回 `impl IntoElement` 时 |
| `FluentBuilder` | 方法链 builder 模式 | 几乎所有 UI 构建代码 |
| `VisualContext` | 提供 `new_window_entity()` 等 | 在窗口上下文中创建 Entity 时 |
| `BorrowAppContext` | 提供 `set_global()` 等 | 操作全局状态时 |
| `Refineable` | 样式合并 | 内部使用 |
| `StyledImage` | 图片样式方法 | 使用 `img()` 时 |

### 2.4.3 何时使用完整路径

大多数情况下，`use gpui::prelude::*;` 就够了。但在以下情况需要完整路径：

**1. 类型冲突时**

```rust
// 如果你也定义了一个叫 Render 的 trait
use gpui::Render as GpuiRender;

impl GpuiRender for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
    }
}
```

**2. 导出公开 API 时**

```rust
// 在库的公开 API 中，显式导出类型更清晰
pub use gpui::{Entity, Context, Window};
```

**3. 在宏中**

```rust
// 过程宏内部使用完整路径，避免命名空间问题
#[derive(gpui::Render)]
struct MyView;
```

### 2.4.4 推荐的导入方式

每个 GPUI 文件开头的标准导入：

```rust
use gpui::*;
use gpui::prelude::*;
```

或者更简洁地（因为 `gpui::*` 不包含 prelude 中的 trait）：

```rust
use gpui::{prelude::*, *};
```

## 本章小结

环境搭建完成。你现在应该：

1. 确认 Rust 工具链已安装
2. 根据操作系统安装系统依赖（wgpu 所需图形库、Wayland/X11）
3. 创建项目并添加 GPUI 依赖（`gpui = "0.2"` 或通过 git）
4. 了解 `gpui::prelude` 导入了哪些 trait

下一章，我们写第一个能运行的 GPUI 应用。
