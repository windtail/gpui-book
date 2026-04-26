# 第 50 章：行布局

行布局负责将已塑形的文本拆分成多行，处理换行、缩进、截断和光标定位。

## 50.1 LineWrapper

`LineWrapper` 是行换行的核心算法：

```rust
pub struct LineWrapper {
    text_system: Arc<TextSystem>,
    pub(crate) font_id: FontId,
    pub(crate) font_size: Pixels,
    cached_ascii_char_widths: [Option<Pixels>; 128],
    cached_other_char_widths: HashMap<char, Pixels>,
}
```

通过 `TextSystem` 获取：

```rust
use gpui::text::LineWrapperHandle;

// 从 TextSystem 获取 LineWrapperHandle
let handle: LineWrapperHandle = cx.text_system().line_wrapper(
    font("Helvetica"),
    gpui::px(16.0),
);

// 使用 handle 包装行
let fragments = &[LineFragment::text("Hello world, this is a long line of text")];
let wrap_width = gpui::px(200.0);

for boundary in handle.wrap_line(fragments, wrap_width) {
    println!(
        "Wrap at byte index {}, next indent: {}",
        boundary.ix, boundary.next_indent
    );
}
```

`wrap_line` 返回一个惰性迭代器，每次产生一个 `Boundary`：

```rust
pub struct Boundary {
    pub ix: usize,         // 换行前的最后一个字符的字节索引
    pub next_indent: u32,  // 下一行的缩进空格数（最大 256）
}
```

### 换行算法

算法核心逻辑：

1. 遍历字符，累计宽度
2. 当宽度超过 `wrap_width` 时：
   - 优先在词边界断开（空格之后、非空白字符之前）
   - 如果没有词边界，强制在当前字符断开
3. 记录首行缩进，续行自动应用相同缩进

```rust
// 词判断逻辑
pub(crate) fn is_word_char(c: char) -> bool {
    c.is_ascii_alphanumeric() ||
    matches!(c, '\u{00C0}'..='\u{00FF}') || // Latin-1
    matches!(c, '\u{0100}'..='\u{017F}') || // Latin Extended-A
    matches!(c, '\u{0180}'..='\u{024F}') || // Latin Extended-B
    matches!(c, '\u{0400}'..='\u{04FF}') || // Cyrillic
    matches!(c, '\u{1E00}'..='\u{1EFF}') || // Latin Extended Additional
    matches!(c, '\u{0300}'..='\u{036F}') || // Combining Diacritical
    matches!(c, '\u{0980}'..='\u{09FF}') || // Bengali
    matches!(c, '-' | '_' | '.' | '\'' | '’' | '$' | '%' | '@' | '#' | '^' | '~' | ',' | '=' | ':' | ';' | '⋯')
}
```

CJK 字符不被视为词字符（`你好` 返回 false），因此每个 CJK 字符都可以作为换行候选点。

### LineFragment 混合文本与元素

```rust
pub enum LineFragment<'a> {
    Text { text: &'a str },
    Element { width: Pixels, len_utf8: usize },
}

// 行中可以嵌入固定宽度元素
let fragments = &[
    LineFragment::text("Hello "),
    LineFragment::element(gpui::px(20.0), 1), // 内联图标占 1 字节
    LineFragment::text(" world"),
];

for boundary in wrapper.wrap_line(fragments, gpui::px(100.0)) {
    // 元素被视为词边界候选点
}
```

### 文本截断

```rust
use gpui::text::{TruncateFrom, LineWrapper};

let (truncated, new_runs) = wrapper.truncate_line(
    "Hello world this is a very long text".into(),
    gpui::px(150.0),
    "…",
    &runs,
    TruncateFrom::End,  // 从末尾截断
);
// 结果: "Hello world this is…"
```

`TruncateFrom::Start` 从开头截断，保留末尾内容：

```rust
let (truncated, _) = wrapper.truncate_line(
    "a/b/c/d/e/f/g/h/i.rs".into(),
    gpui::px(100.0),
    "…",
    &runs,
    TruncateFrom::Start,
);
// 结果: "…h/i.rs"
```

## 50.2 WrappedLine 与 WrappedLineLayout

### WrappedLineLayout

`WrappedLineLayout` 是换行后的布局结果：

```rust
pub struct WrappedLineLayout {
    pub unwrapped_layout: Arc<LineLayout>,
    pub wrap_boundaries: SmallVec<[WrapBoundary; 1]>,
    pub wrap_width: Option<Pixels>,
}

pub struct WrapBoundary {
    pub run_ix: usize,    // 换行处的 run 索引
    pub glyph_ix: usize,  // 换行处的 glyph 索引
}
```

常用方法：

```rust
let layout: Arc<WrappedLineLayout> = /* ... */;

// 总宽度
let width = layout.width();

// 文本尺寸（给定行高）
let size = layout.size(line_height);

// 换行边界
let boundaries = layout.wrap_boundaries();

// 行数
let line_count = layout.wrap_boundaries.len() + 1;

// 位置 → 字节索引
let index = layout.index_for_position(
    gpui::point(gpui::px(50.0), gpui::px(12.0)),
    line_height,
)?;

// 字节索引 → 位置
let pos = layout.position_for_index(index, line_height);
```

### WrappedLine

`WrappedLine` 包含装饰信息，可以直接绘制：

```rust
pub struct WrappedLine {
    pub(crate) layout: Arc<WrappedLineLayout>,
    pub text: SharedString,
    pub(crate) decoration_runs: Vec<DecorationRun>,
}

// 绘制
wrapped_line.paint(
    gpui::point(gpui::px(10.0), gpui::px(20.0)),
    line_height,
    TextAlign::Left,
    Some(bounds),  // 用于对齐的边界
    window,
    cx,
)?;

// 只绘制背景色
wrapped_line.paint_background(origin, line_height, TextAlign::Left, Some(bounds), window, cx)?;
```

### 光标位置计算

文本编辑器中最常见的操作是根据鼠标位置找到字符索引：

```rust
// 精确索引：光标必须在字符边界上
let result = layout.index_for_position(pos, line_height);
match result {
    Ok(index) => println!("Exact: byte {}", index),
    Err(closest) => println!("Between bytes, closest: {}", closest),
}

// 最近索引：总是返回最近的字符边界
let closest = layout.closest_index_for_position(pos, line_height);
```

对应的反向操作：

```rust
// 在 LineLayout 上按字节索引查 X 位置
let x = layout.x_for_index(byte_index);

// 查找字符所在字体
let font_id = layout.font_id_for_index(byte_index);
```

## 50.3 UTF-8 vs UTF-16 范围转换

IME（输入法编辑器）使用 UTF-16 编码报告文本范围，而 GPUI 内部使用 UTF-8。这需要在两者之间转换。

### 为什么需要转换

```
文本: "你好"
UTF-8 字节索引: [0, 3, 6]   (每个中文字符 3 字节)
UTF-16 码元索引: [0, 1, 2]   (每个 BMP 字符 1 个码元)
```

IME 报告选区 `(start=1, end=2)` 指的是 UTF-16 码元索引，需要转为 UTF-8 字节索引 `(3, 6)`。

### 转换逻辑

在 `EntityInputHandler` 的实现中：

```rust
impl EntityInputHandler for MyEditor {
    fn text_for_range(
        &mut self,
        range_utf16: Range<usize>,
        cx: &mut Window,
    ) -> Result<Option<(String, Range<usize>)>> {
        let text = "Hello 你好";

        // UTF-16 → UTF-8 转换
        let utf8_start = utf16_to_utf8_index(text, range_utf16.start);
        let utf8_end = utf16_to_utf8_index(text, range_utf16.end);

        Ok(Some((
            text[utf8_start..utf8_end].to_string(),
            utf8_start..utf8_end,
        )))
    }
}

fn utf16_to_utf8_index(text: &str, utf16_index: usize) -> usize {
    let mut utf16_count = 0;
    for (utf8_index, ch) in text.char_indices() {
        if utf16_count >= utf16_index {
            return utf8_index;
        }
        utf16_count += ch.len_utf16();
    }
    text.len()
}

fn utf8_to_utf16_index(text: &str, utf8_index: usize) -> usize {
    let mut utf16_count = 0;
    for (i, ch) in text.char_indices() {
        if i >= utf8_index {
            break;
        }
        utf16_count += ch.len_utf16();
    }
    utf16_count
}
```

### ShapedGlyph.index 使用 UTF-8

```rust
pub struct ShapedGlyph {
    pub index: usize,  // UTF-8 字节索引，不是字符索引
}

// 对于 ASCII "abc"：
// glyph[0].index = 0, glyph[1].index = 1, glyph[2].index = 2

// 对于 "你好"：
// glyph[0].index = 0, glyph[1].index = 3
```

`ShapedGlyph.index` 直接使用 UTF-8 字节偏移，这使得 `text[..glyph.index]` 可以安全切片。

## 50.4 双向文本支持

GPUI 通过 `rustybuzz`（HarfBuzz 的 Rust 移植）处理双向文本。文本塑形阶段自动处理 RTL 脚本：

```rust
// PlatformTextSystem::layout_line 内部调用 rustybuzz
// rustybuzz 处理：
// 1. 字符到 glyph 映射（包括连字、替换）
// 2. 双向重排序（Bidi algorithm）
// 3. Glyph 定位（kerning、mark-to-base）

// 阿拉伯文示例
let shaped = window_text_system.shape_line(
    "مرحبا بالعالم".into(),  // "Hello World" in Arabic
    gpui::px(16.0),
    &[TextRun {
        len: 27,
        font: font("Noto Sans Arabic"),
        color: gpui::white(),
        ..Default::default()
    }],
    None,
);

// glyph 位置已经按照 RTL 顺序排列
for glyph in &shaped.runs[0].glyphs {
    // glyph.position.x 从右到左递增
}
```

混合 LTR/RTL 文本（如阿拉伯文中嵌入英文）：

```rust
// "هذا test 文本" - 阿拉伯文 + 英文 + 中文混合
let shaped = window_text_system.shape_line(
    "هذا test 文本".into(),
    gpui::px(16.0),
    &[TextRun {
        len: 21, // UTF-8 字节长度
        font: font(".SystemUIFont"),
        color: gpui::white(),
        ..Default::default()
    }],
    None,
);
```

Bidi 算法会自动：
1. 检测每个字符的书写方向
2. 将文本拆分为 runs
3. 反转 RTL runs 的顺序
4. 处理嵌入和覆盖

## 50.5 行布局缓存

`LineLayoutCache` 使用双帧缓存策略：

```rust
pub(crate) struct LineLayoutCache {
    previous_frame: Mutex<FrameCache>,
    current_frame: RwLock<FrameCache>,
    platform_text_system: Arc<dyn PlatformTextSystem>,
}

struct FrameCache {
    lines: FxHashMap<Arc<CacheKey>, Arc<LineLayout>>,
    wrapped_lines: FxHashMap<Arc<CacheKey>, Arc<WrappedLineLayout>>,
    used_lines: Vec<Arc<CacheKey>>,
    used_wrapped_lines: Vec<Arc<CacheKey>>,
}
```

查找策略：
1. 先查 `current_frame`
2. 未命中则从 `previous_frame` 迁移
3. 都未命中则重新塑形

帧结束时交换：

```rust
pub fn finish_frame(&self) {
    let mut prev_frame = self.previous_frame.lock();
    let mut curr_frame = self.current_frame.write();
    std::mem::swap(&mut *prev_frame, &mut *curr_frame);
    // 清空（现在是旧帧的）current_frame
    curr_frame.lines.clear();
    curr_frame.wrapped_lines.clear();
    // ...
}
```

静态文本在连续帧中不会重新塑形，只需要查一次缓存。

### 哈希缓存（零分配查找）

对于不想分配 `SharedString` 的场景，可以用哈希作为缓存键：

```rust
// 缓存命中时无需分配
let layout = window_text_system.try_layout_line_by_hash(
    text_hash,      // u64 哈希
    text_len,       // UTF-8 字节长度
    font_size,
    &font_runs,
    force_width,
)?;
// layout: Option<Arc<LineLayout>>
```

适用于 Editor 等需要频繁查询相同文本内容的场景。
