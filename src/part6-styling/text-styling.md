# 第25章：文本样式

## 25.1 字体设置

### 25.1.1 `font_family()` 字体系列

设置文本使用的字体族：

```rust
div()
    .font_family("Arial")
    .child("使用 Arial 字体")

// 等宽字体（代码显示）
div()
    .font_family("JetBrains Mono")
    .child("const x: i32 = 42;")

// 系统默认字体
div()
    .child("使用系统默认字体")
```

`font_family()` 在 `TextStyleRefinement` 中设置 `font_family` 字段，该值会级联到所有子元素。

### 25.1.2 `text_size()` 字号

使用 `AbsoluteLength` 设置精确字号：

```rust
use gpui::{px, rems};

div()
    .text_size(px(16.))     // 16 像素
    .text_size(rems(1.0))   // 1rem（通常 = 16px）
    .text_size(rems(0.875)) // 14px
    .child("不同字号设置方式")
```

### 25.1.3 `text_xs/sm/base/lg/xl/2xl` 快捷方法

GPUI 提供类 Tailwind 的字号快捷方法：

```rust
div()
    .text_xs()     // 0.75rem = 12px
    .text_sm()     // 0.875rem = 14px
    .text_base()   // 1.0rem = 16px
    .text_lg()     // 1.125rem = 18px
    .text_xl()     // 1.25rem = 20px
    .text_2xl()    // 1.5rem = 24px
    .text_3xl()    // 1.875rem = 30px
    .child("快捷字号")
```

这些方法设置 `font_size` 为 `rems()` 值，会跟随根字体大小缩放。

## 25.2 字重与字形

### 25.2.1 `font_weight()` 字重

`FontWeight` 控制文本粗细：

```rust
use gpui::FontWeight;

div()
    .font_weight(FontWeight::THIN)       // 100
    .child("极细")

div()
    .font_weight(FontWeight::EXTRA_LIGHT) // 200
    .child("特细")

div()
    .font_weight(FontWeight::LIGHT)      // 300
    .child("细")

div()
    .font_weight(FontWeight::NORMAL)     // 400
    .child("常规")

div()
    .font_weight(FontWeight::MEDIUM)     // 500
    .child("中等")

div()
    .font_weight(FontWeight::SEMIBOLD)   // 600
    .child("半粗")

div()
    .font_weight(FontWeight::BOLD)       // 700
    .child("粗体")

div()
    .font_weight(FontWeight::EXTRA_BOLD) // 800
    .child("特粗")

div()
    .font_weight(FontWeight::BLACK)      // 900
    .child("黑体")
```

常用模式：

```rust
// 标题用粗体
div()
    .text_xl()
    .font_weight(FontWeight::BOLD)
    .child("章节标题")

// 正文用常规
div()
    .text_base()
    .font_weight(FontWeight::NORMAL)
    .child("正文内容")

// 标签用半粗体
div()
    .text_sm()
    .font_weight(FontWeight::SEMIBOLD)
    .child("标签文字")
```

### 25.2.2 `font_style()` 斜体

```rust
use gpui::FontStyle;

div()
    .font_style(FontStyle::Normal)
    .child("正常文本")

div()
    .font_style(FontStyle::Italic)
    .child("斜体文本")

div()
    .font_style(FontStyle::Oblique)
    .child("倾斜文本（机械倾斜，非真正的斜体字形）")
```

`FontStyle` 有三个变体：`Normal`（正常）、`Italic`（使用专门的斜体字形）和 `Oblique`（将正体字形机械倾斜）。`Italic` 和 `Oblique` 的视觉效果类似，但 `Italic` 使用专门设计的斜体字形，而 `Oblique` 只是将常规字形倾斜。

快捷方法：

```rust
div().italic().child("斜体")      // 等同于 font_style(FontStyle::Italic)
div().not_italic().child("正常")   // 等同于 font_style(FontStyle::Normal)
```

## 25.3 文本颜色与装饰

### 25.3.1 `text_color()`

设置文本颜色，该值会级联到子元素：

```rust
div()
    .text_color(red)
    .child("红色文本")

// 在父容器上设置，影响所有子文本
div()
    .text_color(cx.theme().text)
    .child(div().child("继承父文本颜色"))
    .child(div().child("也继承"))
```

### 25.3.2 `underline()` 下划线

```rust
div()
    .underline()
    .child("带下划线的文本")

// 自定义下划线
div()
    .text_decoration_color(red)
    .text_decoration_2()  // 2px 粗
    .child("红色2px下划线")
```

### 25.3.3 `strikethrough()` 删除线

`line_through()` 方法添加删除线：

```rust
div()
    .line_through()
    .child("删除线文本")

// 在已完成任务列表中使用
div()
    .when(task.completed, |this| this.line_through())
    .child(&task.name)
```

移除装饰：

```rust
div()
    .text_decoration_none()
    .child("无任何装饰的文本")
```

## 25.4 文本排版

### 25.4.1 `text_left/center/right/justify`

文本水平对齐：

```rust
// 左对齐（默认）
div().text_left().child("左对齐文本")

// 居中
div().text_center().child("居中文本")

// 右对齐
div().text_right().child("右对齐文本")

// 两端对齐
div().text_justify().child("两端对齐的文本，每行首尾都对齐到容器边缘")
```

### 25.4.2 `line_height()` 行高

```rust
use gpui::{rems, DefiniteLength};

// 固定行高
div()
    .line_height(rems(1.5))  // 1.5rem = 24px
    .child("多行文本的第一行")
    .child("多行文本的第二行");

// 相对行高（相对于字体大小）
div()
    .line_height(DefiniteLength::Fraction(1.5))  // 字体大小的 1.5 倍
    .child("1.5倍行高")
```

紧凑和宽松行高：

```rust
// 紧凑（适合数据表格）
div().line_height(rems(1.0)).child("紧凑行高")

// 标准
div().line_height(rems(1.5)).child("标准行高")

// 宽松（适合阅读）
div().line_height(rems(2.0)).child("宽松行高")
```

### 25.4.3 `tracking()` 字间距

```rust
use gpui::DefiniteLength;

// 收紧字间距
div()
    .tracking(DefiniteLength::Absolute(px(-0.5).into()))
    .child("紧凑字间距")

// 宽松字间距
div()
    .tracking(DefiniteLength::Absolute(px(2.0).into()))
    .child("宽松字间距")
```

### 25.4.4 `leading()` 行间距

`leading` 控制行与行之间的额外间距，与 `line_height` 不同，它是在行高之外的附加间距：

```rust
div()
    .leading(rems(0.5))  // 额外 8px 行间距
    .child("多行文本段落")
```

## 25.5 文本截断

### 25.5.1 `truncate()` 省略号

`truncate()` 组合了三个样式：隐藏溢出、不换行、添加省略号：

```rust
// truncate 等同于：
// overflow_hidden().whitespace_nowrap().text_ellipsis()

div()
    .w(px(200.))
    .truncate()
    .child("这是一段很长的文本，超出宽度后会显示省略号")
// 输出：这是一段很长的文本，超...
```

`text_ellipsis()` 只添加省略号，不控制换行：

```rust
div()
    .overflow_hidden()
    .whitespace_nowrap()
    .text_ellipsis()
    .child("手动组合省略号效果")
```

### 25.5.2 `overflow` 处理

文本溢出时的完整处理：

```rust
// 单行截断
div()
    .w(px(300.))
    .overflow_hidden()
    .whitespace_nowrap()
    .text_ellipsis()
    .child("很长的文件路径/目录/子目录/文件名")

// 起始省略号（适合文件路径）
div()
    .w(px(300.))
    .overflow_hidden()
    .whitespace_nowrap()
    .text_ellipsis_start()
    .child("/home/user/projects/my-app/src/components/Header.tsx")
// 输出：...ponents/Header.tsx

// 多行截断
div()
    .w(px(300.))
    .line_clamp(3)  // 最多显示3行
    .child("第一行文本...\n第二行文本...\n第三行文本...\n第四行会被隐藏")
```

## 25.6 文本样式综合示例

### 25.6.1 标题组件

```rust
struct Heading {
    level: u8,
    text: String,
}

impl Render for Heading {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        match self.level {
            1 => div()
                .text_3xl()
                .font_weight(FontWeight::BOLD)
                .child(&self.text),
            2 => div()
                .text_2xl()
                .font_weight(FontWeight::BOLD)
                .child(&self.text),
            3 => div()
                .text_xl()
                .font_weight(FontWeight::SEMIBOLD)
                .child(&self.text),
            _ => div()
                .text_lg()
                .font_weight(FontWeight::SEMIBOLD)
                .child(&self.text),
        }
    }
}
```

### 25.6.2 代码片段组件

```rust
struct CodeSnippet {
    code: String,
    language: String,
}

impl Render for CodeSnippet {
    fn render(&mut self, window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let colors = Colors::for_appearance(window);
        div()
            .flex()
            .flex_col()
            .rounded_lg()
            .border_1()
            .border_color(colors.border)
            .overflow_hidden()
            .child(
                div()
                    .flex()
                    .items_center()
                    .justify_between()
                    .px_3()
                    .py_1()
                    .bg(colors.container)
                    .border_b_1()
                    .border_color(colors.border)
                    .child(div().text_xs().font_family("JetBrains Mono").child(&self.language))
            )
            .child(
                div()
                    .p_3()
                    .text_sm()
                    .font_family("JetBrains Mono")
                    .text_color(colors.text)
                    .child(&self.code)
            )
    }
}
```

### 25.6.3 标签（Badge）组件

```rust
struct Badge {
    text: String,
    variant: BadgeVariant,
}

enum BadgeVariant {
    Default,
    Success,
    Warning,
    Error,
}

impl Render for Badge {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let (bg_color, text_color) = match self.variant {
            BadgeVariant::Success => (green, white),
            BadgeVariant::Warning => (yellow, black),
            BadgeVariant::Error => (red, white),
            BadgeVariant::Default => (gray, white),
        };

        div()
            .flex()
            .items_center()
            .px_2()
            .py_0()
            .rounded_full()
            .text_xs()
            .font_weight(FontWeight::MEDIUM)
            .bg(bg_color)
            .text_color(text_color)
            .child(&self.text)
    }
}
```
