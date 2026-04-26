# 第 41 章：Global 状态

## 41.1 `Global` marker trait

### 41.1.1 标记全局单例

`Global` 是一个空的 marker trait。实现它后，类型可以被存储在 `App` 的全局存储中。

```rust
use gpui::Global;

struct AppConfig {
    theme: Theme,
    font_size: f32,
    auto_save: bool,
}

impl Global for AppConfig {}
```

### 41.1.2 类型安全的保证

只有实现了 `Global` 的类型才能通过 `cx.global()` / `cx.set_global()` 访问：

```rust
struct NotGlobal;
// 没有实现 Global

cx.set_global(NotGlobal);
// 编译错误：the trait `Global` is not implemented for `NotGlobal`

struct IsGlobal;
impl Global for IsGlobal {}

cx.set_global(IsGlobal); // 编译通过
```

`'static` 生命周期是 `Global` trait 的隐式约束：

```rust
struct WithLifetime<'a> {
    ref: &'a str,
}
// 无法实现 Global —— 不是 'static
```

## 41.2 `cx.set_global()` 与 `cx.global()`

### 41.2.1 设置全局状态

```rust
// 在应用启动时设置
fn setup(cx: &mut App) {
    cx.set_global(AppConfig {
        theme: Theme::Dark,
        font_size: 14.0,
        auto_save: true,
    });
}
```

### 41.2.2 读取全局状态

```rust
fn get_font_size(cx: &App) -> f32 {
    cx.global::<AppConfig>().font_size
}
```

`cx.global()` 在值不存在时 panic。不确定是否存在时用 `cx.try_global()`。

### 41.2.3 `cx.try_global()` 安全访问

```rust
fn get_theme(cx: &App) -> Theme {
    match cx.try_global::<AppConfig>() {
        Some(config) => config.theme.clone(),
        None => Theme::default(), // 配置未初始化时的默认值
    }
}
```

## 41.3 `cx.update_global()`

### 41.3.1 更新全局状态

```rust
cx.update_global::<AppConfig, _>(|config, cx| {
    config.font_size = 16.0;
    // 修改后，所有 observe_global::<AppConfig> 的回调被触发
});
```

### 41.3.2 闭包签名

闭包接收 `&mut T` 和 `&mut App`：

```rust
cx.update_global::<ThemeManager, _>(|manager, cx| {
    manager.switch_to_dark();
    // 可以进一步操作 App
    cx.refresh();
});
```

返回闭包的返回值：

```rust
let old_size = cx.update_global::<AppConfig, _>(|config, _| {
    let old = config.font_size;
    config.font_size = 18.0;
    old
});
```

## 41.4 `ReadGlobal` 与 `UpdateGlobal` trait

### 41.4.1 便捷访问方法

`ReadGlobal` 和 `UpdateGlobal` 为所有 `Global` 类型提供 blanket 实现：

```rust
use gpui::{Global, ReadGlobal, UpdateGlobal, App, BorrowAppContext};

// ReadGlobal 自动实现 —— 提供 global(cx) 方法
fn read_theme(cx: &App) {
    let theme = AppConfig::global(cx);
    // 等价于 cx.global::<AppConfig>()
}

// UpdateGlobal 自动实现 —— 提供 set_global 和 update_global 方法
fn set_theme(cx: &mut impl BorrowAppContext) {
    AppConfig::set_global(cx, AppConfig {
        theme: Theme::Light,
        font_size: 14.0,
        auto_save: true,
    });
    // 等价于 cx.set_global(AppConfig { ... })
}

fn update_theme(cx: &mut impl BorrowAppContext) {
    AppConfig::update_global(cx, |config, _| {
        config.font_size = 16.0;
    });
    // 等价于 cx.update_global::<AppConfig, _>(...)
}
```

### 41.4.2 实现 trait

`ReadGlobal` 和 `UpdateGlobal` 对所有 `Global` 类型自动实现，不需要手动写：

```rust
impl Global for MyConfig {}
// ReadGlobal 和 UpdateGlobal 自动可用
// MyConfig::global(cx)
// MyConfig::set_global(cx, value)
// MyConfig::update_global(cx, |config, _| { ... })
```

## 41.5 全局状态模式

### 41.5.1 主题配置

```rust
use gpui::{hsla, Hsla, Global};

struct ThemeColors {
    pub background: Hsla,
    pub foreground: Hsla,
    pub accent: Hsla,
    pub border: Hsla,
}

impl ThemeColors {
    pub fn dark() -> Self {
        Self {
            background: hsla(220.0, 0.1, 0.15, 1.0),
            foreground: hsla(220.0, 0.1, 0.85, 1.0),
            accent: hsla(210.0, 0.9, 0.6, 1.0),
            border: hsla(220.0, 0.1, 0.3, 1.0),
        }
    }

    pub fn light() -> Self {
        Self {
            background: hsla(220.0, 0.1, 0.95, 1.0),
            foreground: hsla(220.0, 0.1, 0.2, 1.0),
            accent: hsla(210.0, 0.9, 0.5, 1.0),
            border: hsla(220.0, 0.1, 0.7, 1.0),
        }
    }
}

impl Global for ThemeColors {}

// 设置初始主题
cx.set_global(ThemeColors::dark());

// 切换主题
fn toggle_theme(cx: &mut App) {
    let current = cx.global::<ThemeColors>();
    let new_theme = if current.background.l > 0.5 {
        ThemeColors::dark()
    } else {
        ThemeColors::light()
    };
    cx.set_global(new_theme);
}

// 在 View 中使用
impl MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let theme = cx.global::<ThemeColors>();
        div()
            .bg(theme.background)
            .text_color(theme.foreground)
            .child("Hello")
    }
}
```

### 41.5.2 应用设置

```rust
struct Settings {
    auto_save_interval: Duration,
    telemetry_enabled: bool,
    font_family: String,
}

impl Global for Settings {}

// 从文件加载
fn load_settings(cx: &mut App) {
    let settings = std::fs::read_to_string("settings.json")
        .ok()
        .and_then(|s| serde_json::from_str(&s).ok())
        .unwrap_or_default();
    cx.set_global(settings);
}

// 修改设置
fn set_auto_save(seconds: u64, cx: &mut App) {
    cx.update_global::<Settings, _>(|s, _| {
        s.auto_save_interval = Duration::from_secs(seconds);
    });
    save_settings(cx);
}
```

### 41.5.3 全局服务（日志、遥测）

```rust
struct TelemetryService {
    client: TelemetryClient,
    enabled: bool,
}

impl Global for TelemetryService {}

impl TelemetryService {
    fn record(event: &str, cx: &mut App) {
        let service = cx.global_mut::<Self>();
        if service.enabled {
            service.client.send(event);
        }
    }
}
```

## 41.6 Global 的限制与替代方案

Global 的局限：
- 每种类型只能有一个实例
- 所有代码都可以访问，没有访问控制
- 不能在 `App` 之外存活

替代方案：
- Entity：有类型身份，可以创建多个实例
- 闭包捕获：限定的作用域
- 参数传递：显式的数据流

```rust
// Global：任何地方都能访问
cx.global::<AppConfig>()

// Entity：需要持有引用
buffer.read(cx).text()

// 选择原则：
// - 全应用唯一、到处都要用的状态 → Global
// - 有多个实例、有明确所有者 → Entity
```
