# 第 48 章：字体管理

GPUI 的字体系统位于 `crates/gpui/src/text_system.rs`。它封装了平台差异，提供统一的字体描述、加载、度量查询接口。

## 48.1 Font struct

字体用 `Font` 结构体描述：

```rust
pub struct Font {
    pub family: SharedString,
    pub features: FontFeatures,
    pub fallbacks: Option<FontFallbacks>,
    pub weight: FontWeight,
    pub style: FontStyle,
}
```

最常用的构造方式：

```rust
use gpui::{font, FontStyle, FontWeight};

// 默认字体
let default_font = font(".SystemUIFont");

// 指定字体系列
let helvetica = font("Helvetica");

// 链式调用设置字重和样式
let bold_italic = font("IBM Plex Sans")
    .bold()
    .italic();

// FontFeatures 控制 OpenType 特性
let with_features = Font {
    family: "IBM Plex Mono".into(),
    weight: FontWeight::NORMAL,
    style: FontStyle::Normal,
    features: FontFeatures::default(),
    fallbacks: None,
};
```

`Font` 实现了 `Default`，默认返回 `.SystemUIFont`——在 macOS 上是 San Francisco，在 Linux 上按发行版不同回退到不同字体。

## 48.2 FontWeight

`FontWeight` 是一个包装 `f32` 的新类型，范围 100–900：

```rust
use gpui::FontWeight;

// 预定义常量
let thin       = FontWeight::THIN;        // 100
let light      = FontWeight::LIGHT;       // 300
let normal     = FontWeight::NORMAL;      // 400
let medium     = FontWeight::MEDIUM;      // 500
let semibold   = FontWeight::SEMIBOLD;    // 600
let bold       = FontWeight::BOLD;        // 700
let black      = FontWeight::BLACK;       // 900

// 自定义数值
let custom = FontWeight(550.0);

// 从字符串解析（实现了 FromStr）
let parsed: FontWeight = "700".parse().unwrap();
```

所有 9 个标准字重作为一个常量数组暴露：

```rust
for weight in FontWeight::ALL {
    println!("{:?}", weight);
}
```

## 48.3 FontStyle

```rust
pub enum FontStyle {
    Normal,   // 常规
    Italic,   // 斜体（独立字型）
    Oblique,  // 倾斜（算法变形）
}
```

```rust
use gpui::FontStyle;

let italic_font = Font {
    family: "IBM Plex Sans".into(),
    style: FontStyle::Italic,
    ..Default::default()
};
```

`Italic` 和 `Oblique` 的区别：Italic 使用专门设计的斜体字型，Oblique 是对常规字型的算法倾斜。多数情况下使用 `Italic`。

## 48.4 字体加载

### 系统字体

`TextSystem` 在创建时扫描系统已安装字体：

```rust
// text_system.rs 内部
pub fn all_font_names(&self) -> Vec<String> {
    let mut names = self.platform_text_system.all_font_names();
    names.extend(
        self.fallback_font_stack.iter()
            .map(|font| font.family.to_string()),
    );
    names.push(".SystemUIFont".to_string());
    names.sort();
    names.dedup();
    names
}
```

### 自定义字体

通过 `add_fonts` 加载自定义字体数据：

```rust
use gpui::App;

fn load_custom_font(cx: &mut App) {
    // 从嵌入资源或文件系统读取字体二进制数据
    let font_data: Vec<u8> = include_bytes!("../assets/MyCustomFont.ttf").to_vec();

    cx.text_system()
        .add_fonts(vec![std::borrow::Cow::Owned(font_data)])
        .expect("failed to load font");
}
```

### FontId 标识符

每个唯一字体配置对应一个 `FontId`：

```rust
pub struct FontId(pub usize);
pub struct FontFamilyId(pub usize);
```

`FontId` 是不透明句柄，由 `TextSystem::resolve_font` 分配。内部用 `RwLock<FxHashMap<Font, Result<FontId>>>` 做缓存，相同 `Font` 描述复用同一个 `FontId`。

```rust
let font_id = cx.text_system().resolve_font(&font("Helvetica"));
// 第二次调用返回同一个 ID
let same_id = cx.text_system().resolve_font(&font("Helvetica"));
assert_eq!(font_id, same_id);
```

## 48.5 FontMetrics

`FontMetrics` 包含字型的所有关键度量值：

```rust
pub struct FontMetrics {
    pub units_per_em: u32,          // em 方格的单位数（通常 1000 或 2048）
    pub ascent: f32,                // 基线到顶部的距离
    pub descent: f32,               // 基线到底部的距离（通常为负）
    pub line_gap: f32,              // 行间距
    pub underline_position: f32,    // 下划线位置
    pub underline_thickness: f32,   // 下划线粗细
    pub cap_height: f32,            // 大写字母高度
    pub x_height: f32,              // 小写 x 高度
    pub bounding_box: Bounds<f32>,  // 字体包围盒
}
```

这些值是原始字体单位，`TextSystem` 提供缩放后的像素值查询：

```rust
let font_id = cx.text_system().resolve_font(&font("Helvetica"));
let font_size = gpui::px(16.0);

// 缩放后的度量值
let ascent = cx.text_system().ascent(font_id, font_size);
let descent = cx.text_system().descent(font_id, font_size);
let cap = cx.text_system().cap_height(font_id, font_size);
let x_h = cx.text_system().x_height(font_id, font_size);

// 包围盒
let bbox = cx.text_system().bounding_box(font_id, font_size);

// 字符级度量
let char_bounds = cx.text_system().typographic_bounds(font_id, font_size, 'M')?;
let char_advance = cx.text_system().advance(font_id, font_size, 'M')?;

// em 宽度（M 字符宽度）
let em_width = cx.text_system().em_width(font_id, font_size)?;
let ch_width = cx.text_system().ch_width(font_id, font_size)?; // 0 字符宽度
```

基线偏移计算公式：

```rust
pub fn baseline_offset(
    &self,
    font_id: FontId,
    font_size: Pixels,
    line_height: Pixels,
) -> Pixels {
    let ascent = self.ascent(font_id, font_size);
    let descent = self.descent(font_id, font_size);
    let padding_top = (line_height - ascent - descent) / 2.;
    padding_top + ascent
}
```

这个公式确保文本在行高中垂直居中。

## 48.6 FontFeatures

OpenType 特性通过 `FontFeatures` 控制：

```rust
use gpui::FontFeatures;

// 启用特定 OpenType 特性
let features = FontFeatures::default()
    .enable("liga")  // 连字
    .enable("kern"); // 字距调整
```

在 `Font` 结构中传入：

```rust
let font = Font {
    family: "IBM Plex Mono".into(),
    features: FontFeatures::default(),
    fallbacks: None,
    weight: FontWeight::NORMAL,
    style: FontStyle::Normal,
};
```

常用特性：
- `liga`：标准连字
- `dlig`：自由连字
- `kern`：字距调整
- `calt`：上下文替换
- `ss01`–`ss20`：风格集

## 48.7 FontFallbacks 链

字体回退允许为不同脚本指定不同字体：

```rust
use gpui::FontFallbacks;

let font = Font {
    family: "IBM Plex Sans".into(),
    fallbacks: Some(FontFallbacks::new(vec![
        "Noto Sans CJK SC".into(),  // 中文
        "Noto Sans Arabic".into(),  // 阿拉伯文
        "Noto Color Emoji".into(),  // Emoji
    ])),
    weight: FontWeight::NORMAL,
    style: FontStyle::Normal,
    features: FontFeatures::default(),
};
```

`TextSystem` 内部维护了一个全局回退栈：

```rust
fallback_font_stack: smallvec![
    font(".ZedMono"),
    font(".ZedSans"),
    font("Helvetica"),
    font("Segoe UI"),     // Windows
    font("Ubuntu"),       // Gnome (Ubuntu)
    font("Adwaita Sans"), // Gnome 47
    font("Cantarell"),    // Gnome
    font("Noto Sans"),    // KDE
    font("DejaVu Sans"),
    font("Arial"),        // macOS, Windows
]
```

`resolve_font` 方法先尝试主字体，失败后依次尝试回退栈：

```rust
pub fn resolve_font(&self, font: &Font) -> FontId {
    if let Ok(font_id) = self.font_id(font) {
        return font_id;
    }
    for fallback in &self.fallback_font_stack {
        if let Ok(font_id) = self.font_id(fallback) {
            return font_id;
        }
    }
    panic!("failed to resolve font '{}' or any of the fallbacks", font.family);
}
```

## 48.8 字体名称映射

GPUI 定义了特殊字体名称的映射：

```rust
pub fn font_name_with_fallbacks<'a>(name: &'a str, system: &'a str) -> &'a str {
    match name {
        ".SystemUIFont" => system,
        ".ZedSans" | "Zed Plex Sans" => "IBM Plex Sans",
        ".ZedMono" | "Zed Plex Mono" => "Lilex",
        _ => name,
    }
}
```

这些虚拟名称在 Zed 编辑器中广泛使用，实际运行时会被映射到具体字体。
