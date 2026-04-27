# 第23章：颜色系统

## 23.1 `Hsla` 颜色模型

### 23.1.1 色相、饱和度、亮度、透明度

`Hsla` 是 GPUI 的内部颜色表示格式，包含四个分量：

```rust
use gpui::Hsla;

// 各分量范围：
// hue:        0.0 ~ 360.0  色相（角度）
// saturation: 0.0 ~ 100.0  饱和度（百分比）
// lightness:  0.0 ~ 100.0  亮度（百分比）
// alpha:      0.0 ~ 1.0    透明度
```

色相环对应：
- 0° / 360°：红色
- 60°：黄色
- 120°：绿色
- 180°：青色
- 240°：蓝色
- 300°：品红

### 23.1.2 `hsla()` 构造函数

```rust
use gpui::hsla;

let red = hsla(0.0, 100.0, 50.0, 1.0);
let blue = hsla(240.0, 100.0, 50.0, 1.0);
let semi_transparent = hsla(180.0, 80.0, 60.0, 0.5);
```

`Hsla` 可以从字符串解析：

```rust
// CSS 格式解析
let color: Hsla = "hsl(240, 100%, 50%)".parse().unwrap();
let color: Hsla = "#ff0000".parse().unwrap();
let color: Hsla = "rgb(255, 0, 0)".parse().unwrap();
```

## 23.2 RGB 颜色

### 23.2.1 `rgb()` 构造函数

`rgb()` 接受 24 位整数（0xRRGGBB），返回 `Rgba`：

```rust
use gpui::rgb;

let red = rgb(0xff0000);       // 纯红
let green = rgb(0x00ff00);     // 纯绿
let blue = rgb(0x0000ff);      // 纯蓝
let white = rgb(0xffffff);     // 白色
let black = rgb(0x000000);     // 黑色
let gray = rgb(0x808080);      // 中灰
```

### 23.2.2 `rgba()` 带透明度

`rgba()` 接受 32 位整数（0xRRGGBBAA）：

```rust
use gpui::rgba;

let semi_white = rgba(0xffffff80);  // 50% 透明白
let transparent_red = rgba(0xff00007f);  // ~50% 透明红
```

`Rgba` 结构体：

```rust
#[repr(C)]
pub struct Rgba {
    pub r: f32,  // 0.0 ~ 1.0
    pub g: f32,
    pub b: f32,
    pub a: f32,  // 0.0 = 完全透明, 1.0 = 完全不透明
}
```

### 23.2.3 颜色混合

`Rgba` 提供 `blend()` 方法：

```rust
let background = rgb(0xffffff);
let overlay = rgba(0x00000080);  // 半透明黑色

let result = background.blend(overlay);
// result 是白色上覆盖半透明黑色的混合结果
```

## 23.3 命名颜色

### 23.3.1 `gpui::colors` 模块

GPUI 提供内置的颜色常量，通过 `gpui::colors` 模块访问：

```rust
use gpui::colors::*;

// 常用颜色常量
red      // #ef4444
green    // #22c55e
blue     // #3b82f6
yellow   // #eab308
purple   // #a855f7
orange   // #f97316
pink     // #ec4899
cyan     // #06b6d4
white    // #ffffff
black    // #000000
gray     // #6b7280
```

这些常量在 `gpui::prelude` 中自动导入，可以直接使用：

```rust
div()
    .bg(red)
    .text_color(white)
    .child("红底白字")
```

### 23.3.2 内置颜色常量

常用颜色表：

```rust
// 基础色
red, green, blue, yellow, purple, orange, pink, cyan

// 灰度色阶
slate_50, slate_100, ..., slate_900
gray_50, gray_100, ..., gray_900
zinc_50, zinc_100, ..., zinc_900
neutral_50, neutral_100, ..., neutral_900
stone_50, stone_100, ..., stone_900
```

## 23.4 背景与填充

### 23.4.1 `bg()` 方法

`bg()` 接受 `impl Into<Fill>` 类型的颜色值：

```rust
use gpui::{rgb, rgba, hsla};

div()
    .bg(red)                    // 命名颜色
    .bg(rgb(0x2a63d9))          // RGB 十六进制
    .bg(rgba(0x00000080))       // 带透明度
    .bg(hsla(240.0, 80.0, 60.0, 1.0))  // HSLA
    .child("多种颜色设置方式")
```

### 23.4.2 渐变背景

GPUI 支持线性渐变作为背景填充：

```rust
use gpui::linear_gradient;

div()
    .bg(linear_gradient(
        135.0,  // 角度（度）
        linear_color_stop(rgb(0x667eea), 0.0),  // 起点颜色
        linear_color_stop(rgb(0x764ba2), 1.0),  // 终点颜色
    ))
    .p_6()
    .child("渐变背景")
```

`linear_gradient(angle, from, to)` 创建线性渐变，返回 `Background` 类型，直接用于 `.bg()`。角度 0° 表示从顶部开始，顺时针递增。

## 23.5 外观 (Appearance) 系统

### 23.5.1 `DefaultAppearance`

GPUI 提供两个不同的外观类型，需注意区分：

**`DefaultAppearance`** 仅有两个变体：

```rust
use gpui::DefaultAppearance;

match appearance {
    DefaultAppearance::Light => println!("亮色模式"),
    DefaultAppearance::Dark => println!("暗色模式"),
}
```

**`WindowAppearance`** 是窗口级别的外观类型，额外包含 macOS 特有的振动效果变体：

```rust
use gpui::WindowAppearance;

let appearance = window.appearance();
match appearance {
    WindowAppearance::Light => println!("亮色模式"),
    WindowAppearance::Dark => println!("暗色模式"),
    WindowAppearance::VibrantLight => println!("macOS 浅色振动模式"),
    WindowAppearance::VibrantDark => println!("macOS 暗色振动模式"),
}
```

`VibrantLight` 和 `VibrantDark` 属于 `WindowAppearance`，不属于 `DefaultAppearance`。

### 23.5.2 `Colors` 主题色

`Colors` 结构体提供一套语义化颜色，支持亮色/暗色主题切换：

```rust
use gpui::Colors;

pub struct Colors {
    pub text: Rgba,          // 文本颜色
    pub selected_text: Rgba, // 选中文本颜色
    pub background: Rgba,    // 背景颜色
    pub disabled: Rgba,      // 禁用状态颜色
    pub selected: Rgba,      // 选中状态颜色
    pub border: Rgba,        // 边框颜色
    pub separator: Rgba,     // 分隔线颜色
    pub container: Rgba,     // 容器背景颜色
}
```

获取主题颜色：

```rust
// 根据当前窗口外观获取
let colors = Colors::for_appearance(window);

// 或手动选择
let dark_colors = Colors::dark();
let light_colors = Colors::light();
```

实际应用：

```rust
struct ThemedCard;

impl Render for ThemedCard {
    fn render(&mut self, window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let colors = Colors::for_appearance(window);
        div()
            .p_4()
            .rounded_lg()
            .bg(colors.background)
            .border_1()
            .border_color(colors.border)
            .child(
                div()
                    .text_color(colors.text)
                    .child("自适应主题颜色的卡片")
            )
    }
}
```

### 23.5.3 暗色/亮色切换

通过全局状态管理主题：

```rust
#[derive(Default)]
struct ThemeState {
    dark: bool,
}

impl Global for ThemeState {}

// 切换主题
fn toggle_theme(cx: &mut App) {
    cx.update_global(|state: &mut ThemeState, _cx| {
        state.dark = !state.dark;
    });
}

// 渲染时响应主题
impl Render for MyApp {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let is_dark = cx.global::<ThemeState>().dark;
        let colors = if is_dark {
            Colors::dark()
        } else {
            Colors::light()
        };

        div()
            .size_full()
            .bg(colors.background)
            .text_color(colors.text)
            .child("主题自适应的应用")
    }
}
```

### 23.5.4 `GlobalColors` 全局颜色

GPUI 提供 `GlobalColors` 作为全局状态包装：

```rust
use gpui::{GlobalColors, DefaultColors};

// 从全局获取颜色
let colors = cx.global::<GlobalColors>();
let colors = cx.default_colors();  // 通过 DefaultColors trait

// 设置全局颜色
cx.set_global(GlobalColors(Arc::new(my_colors)));
```

通过 `Colors::get_global(cx)` 快捷访问：

```rust
let colors = Colors::get_global(cx);
div()
    .bg(colors.background)
    .text_color(colors.text)
    .child("使用全局颜色的元素")
```
