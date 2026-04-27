# 第13章：文本元素

## 13.1 静态文本

### 13.1.1 `"Hello"` 字面量直接作为元素

```rust
use gpui::{div, IntoElement, Element};

// &'static str 直接实现 Element 和 IntoElement
div()
    .child("Hello, World!")
    .child("Line 1")
    .child("Line 2")
```

`&'static str` 的 Element 实现在 GPUI 中是最轻量的文本渲染路径：

```rust
// 简化版源码
impl Element for &'static str {
    type RequestLayoutState = LayoutId;
    type PrepaintState = ();

    fn request_layout(&mut self, _, _, window: &mut Window, cx: &mut App) -> (LayoutId, ()) {
        let text_layout = window.text_system().shape_text(
            *self,
            font(),           // 默认字体
            font_size(),      // 默认字号
            LineWrapper::default(),
        );
        let style = Style {
            size: Size {
                width: Length::Auto,
                height: Length::Definite(text_layout.size.height.into()),
            },
            ..Default::default()
        };
        let id = window.request_layout(style, [], cx);
        (id, id)
    }

    fn prepaint(&mut self, _, bounds, state, window, cx) {
        // 文本不需要 hitbox
    }

    fn paint(&mut self, _, bounds, _, _, window, cx) {
        window.paint_text(
            bounds.origin,
            *self,
            default_color(),
            ...
        );
    }
}
```

### 13.1.2 `SharedString` 的性能优势

```rust
use gpui::SharedString;

// SharedString 是 Arc<str> 的包装，克隆开销是指针拷贝
let s1: SharedString = "Hello".into();
let s2 = s1.clone();  // 只是增加引用计数，不复制字符串
```

`SharedString` vs `String`：
| 类型 | 克隆开销 | 内存 |
|------|----------|------|
| `String` | O(n) 复制 | 每份独立堆内存 |
| `SharedString` | O(1) 指针 | 共享底层 `Arc<str>` |

在 View 中存储动态文本时优先使用 `SharedString`：

```rust
struct Display {
    // 优于 String
    title: SharedString,
    description: SharedString,
}

impl Render for Display {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(self.title.as_str())   // SharedString → &str → Element
            .child(self.description.as_str())
    }
}
```

## 13.2 `StyledText`

### 13.2.1 多文本段落

`StyledText` 用于渲染包含多种样式的文本块：

```rust
use gpui::{StyledText, TextRun, TextStyle, Font, FontWeight, TextElement};

let text = StyledText::new(
    "Hello World".to_string()
).with_runs(vec![
    TextRun {
        len: 5,  // "Hello"
        font: Font {
            weight: FontWeight::BOLD,
            ..Default::default()
        },
        color: rgb(0x3b82f6),
        background_color: None,
        underline: None,
        strikethrough: None,
    },
    TextRun {
        len: 6,  // " World"
        font: Font::default(),
        color: rgb(0x000000),
        background_color: None,
        underline: None,
        strikethrough: None,
    },
]);

div().child(text)
```

### 13.2.2 `TextRun` 与样式跨度

```rust
struct CodeLine {
    code: String,
    highlights: Vec<(usize, usize, Hsla)>,  // (start, end, color)
}

impl Render for CodeLine {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let mut runs = Vec::new();
        let mut pos = 0;

        for (start, end, color) in &self.highlights {
            if pos < *start {
                runs.push(TextRun {
                    len: start - pos,
                    color: rgb(0x000000),
                    ..Default::default()
                });
            }
            runs.push(TextRun {
                len: end - start,
                color: *color,
                font: Font {
                    family: "JetBrains Mono".into(),
                    ..Default::default()
                },
                ..Default::default()
            });
            pos = *end;
        }

        if pos < self.code.len() {
            runs.push(TextRun {
                len: self.code.len() - pos,
                ..Default::default()
            });
        }

        div().child(StyledText::new(self.code.clone()).with_runs(runs))
    }
}
```

`TextRun` 的关键规则：
- `len` 是**字节长度**，不是字符数
- 所有 `len` 之和必须等于文本总长度
- `color` 控制文本颜色，`background_color` 控制背景高亮

### 13.2.3 文本测量

```rust
// 获取 StyledText 的布局尺寸
let styled = StyledText::new("Measure me".into());
let layout = window.text_system().shape_text(
    &styled.text,
    styled.default_style.font,
    styled.default_style.font_size,
    LineWrapper::new(
        available_width,
        styled.runs.iter(),
    ),
);

println!("Width: {}", layout.size.width);
println!("Height: {}", layout.size.height);
```

文本测量在 `request_layout` 阶段自动进行。如果需要根据文本尺寸做布局决策，可以提前测量。

## 13.3 `InteractiveText`

### 13.3.1 可点击文本

```rust
use gpui::InteractiveText;

struct LinkView {
    url: String,
}

impl Render for LinkView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let styled = StyledText::new(self.url.clone()).with_runs(vec![
            TextRun {
                len: self.url.len(),
                color: rgb(0x3b82f6),
                underline: true,
                ..Default::default()
            },
        ]);

        InteractiveText::new("link", styled)
            .on_click(vec![0..self.url.len()], |index, window, cx| {
                // index: 被点击的文本范围索引
                window.open_url("https://example.com");
            })
    }
}
```

`InteractiveText::new()` 的第一个参数是 `ElementId`，用于追踪点击事件对应的文本范围。

### 13.3.2 Tooltip 支持

```rust
InteractiveText::new("tooltip-text", styled_text)
    .on_click(vec![0..10], |_index, _window, _cx| { ... })
    .tooltip(|cx| {
        div()
            .p_2()
            .rounded_md()
            .bg(gray_800())
            .child(div().text_white().child("Click to open"))
    })
```

### 13.3.3 富文本交互

```rust
struct MarkdownPreview {
    content: String,
    links: Vec<(String, Range<usize>)>,
}

impl Render for MarkdownPreview {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let styled = self.build_styled_text();

        InteractiveText::new("markdown", styled)
            .on_click(this.links.iter().map(|(_, r)| r.clone()).collect(), |index, _window, _cx| {
                // 根据 index 判断点击了哪个链接
            })
                for range in ranges {
                    // 判断点击是否落在链接范围内
                    for (url, link_range) in &this.links {
                        if range.overlaps(&link_range) {
                            cx.open_url(url);
                            break;
                        }
                    }
                }
            }))
    }
}
```

`on_click` 回调接收被点击的文本 `Range<usize>` 集合，可用于判断具体点击了哪个链接区域。
