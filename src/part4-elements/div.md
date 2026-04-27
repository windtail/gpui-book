# 第12章：div 容器

## 12.1 `div()` 是什么

### 12.1.1 HTML div 的类比

`div()` 在 GPUI 中的角色类似 HTML 中的 `<div>`：一个通用容器，支持布局、样式和交互。

```rust
use gpui::{div, div::Div, Styled, ParentElement, IntoElement};

// HTML: <div style="display:flex; padding:16px;">Hello</div>
// GPUI:
div()
    .flex()
    .p_4()
    .child("Hello")
```

### 12.1.2 `Div` struct 的结构

```rust
pub struct Div {
    interactivity: Interactivity,       // 事件处理、状态追踪
    children: SmallVec<[AnyElement; 2]>, // 子元素
    prepaint_listener: Option<...>,      // prepaint 回调
    image_cache: Option<...>,            // 图片缓存
    prepaint_order_fn: Option<...>,      // 动态 prepaint 顺序
}
```

`Div` 将能力拆分为两个核心字段：
- `Interactivity`：鼠标/键盘事件、hover/active/focus 状态
- `children`：子元素集合

### 12.1.3 `Interactivity` + `StyleRefinement`

```rust
// Interactivity 处理交互
impl Div {
    pub fn on_click(mut self, handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static) -> Self {
        self.interactivity.on_click = Some(Box::new(handler));
        self
    }

    pub fn on_hover(mut self, handler: impl Fn(&bool, &mut Window, &mut App) + 'static) -> Self {
        self.interactivity.on_hover = Some(Box::new(handler));
        self
    }
}

// StyleRefinement 处理样式（通过 Styled trait）
impl Styled for Div {
    fn style(&mut self) -> &mut StyleRefinement {
        &mut self.interactivity.style
    }
}
```

`StyleRefinement` 采用增量模式：每个样式方法只修改它关心的字段，未设置的字段保持 `None`，最终与父样式合并。

## 12.2 FluentBuilder 模式

### 12.2.1 方法链的流畅 API

```rust
div()
    .flex()            // 返回 Self
    .flex_col()        // 返回 Self
    .gap_2()           // 返回 Self
    .p_4()             // 返回 Self
    .child("Header")   // 返回 Self
    .child("Body")     // 返回 Self
    .child("Footer")   // 返回 Self
```

所有 `Div` 的方法都返回 `Self`，支持无限链式调用。

### 12.2.2 `FluentBuilder` trait

```rust
pub trait FluentBuilder: Sized {
    fn with(mut self, f: impl FnOnce(&mut Self)) -> Self {
        f(&mut self);
        self
    }
}

impl<T: IntoElement> FluentBuilder for T {}
```

`FluentBuilder` 为所有 `IntoElement` 类型自动实现。`with()` 方法允许在链中执行复杂逻辑：

```rust
div()
    .flex()
    .with(|this| {
        if self.items.is_empty() {
            this.child(div().child("No items"))
        } else {
            this.children(self.items.iter().map(|i| div().child(i.name.as_str())))
        }
    })
```

### 12.2.3 链式调用的限制

`div()` 是值类型，一旦调用 `.into_element()` 或 `.into_any_element()` 后就不能继续使用 builder 方法：

```rust
// 错误：into_element 消费了 div
let el = div().flex().into_element();
el.child("Hello");  // 编译错误：el 是 AnyElement，没有 child 方法

// 正确：先完成链式调用
let el = div().flex().child("Hello").into_element();
```

## 12.3 子元素管理

### 12.3.1 `ParentElement` trait

```rust
pub trait ParentElement {
    fn extend(&mut self, elements: impl IntoIterator<Item = AnyElement>);

    fn child(mut self, child: impl IntoElement) -> Self {
        self.extend(std::iter::once(child.into_element().into_any()));
        self
    }

    fn children(mut self, children: impl IntoIterator<Item = impl IntoElement>) -> Self {
        self.extend(children.into_iter().map(|child| child.into_any_element()));
        self
    }
}
```

`child()` 和 `children()` 都是 `extend()` 的便利包装。

### 12.3.2 `.child()` 添加单个子元素

```rust
div()
    .child("字符串字面量")     // &'static str
    .child(SharedString::from("动态字符串"))
    .child(div())             // 嵌套 div
    .child(div().child("按钮"))
    .child(img("icon.png"))   // 图片
```

### 12.3.3 `.children()` 批量添加

```rust
struct FileTree { files: Vec<String> }

impl Render for FileTree {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex_col()
            .children(
                self.files.iter().map(|f| {
                    div().p_2().child(f.as_str())
                })
            )
    }
}
```

### 12.3.4 条件子元素：`.when()`

```rust
div()
    .child("Always visible")
    .when(self.show_details, |this| {
        this.child(div().child("Details panel"))
    })
    .when_some(self.error.as_ref(), |this, error| {
        this.child(div().text_red_500().child(error.as_str()))
    })
```

`when_some()` 接受 `Option<&T>`，当值为 `Some` 时执行闭包。

## 12.4 div 的能力概览

### 12.4.1 布局：flex、grid、absolute

```rust
// Flexbox
div()
    .flex()           // display: flex
    .flex_row()       // flex-direction: row
    .flex_col()       // flex-direction: column
    .justify_center() // justify-content: center
    .items_start()    // align-items: flex-start
    .flex_1()         // flex: 1
    .gap_2()          // gap: 0.5rem
    .flex_wrap()      // flex-wrap: wrap

// Grid
div()
    .grid()
    .grid_cols(3)     // 3列网格
    .col_span_2()     // 跨2列

// Absolute
div()
    .relative()       // position: relative（父容器）
    .child(
        div()
            .absolute() // position: absolute
            .top_0()
            .right_0()
    )
```

### 12.4.2 样式：颜色、边框、阴影

```rust
div()
    // 背景
    .bg(red_500())
    .bg(rgb(0x3b82f6))
    .bg(rgba(0x00000080))

    // 边框
    .border_1()
    .border_color(gray_300())
    .rounded_md()

    // 阴影
    .shadow_md()

    // 尺寸
    .w(px(200.0))
    .h(re(0.5))
    .min_w(px(100.0))
    .max_h_full()

    // 内边距/外边距
    .p_4()
    .m_2()
    .px_4()
    .py_2()
    .mt_4()
```

### 12.4.3 交互：点击、悬停、拖拽

```rust
div()
    .id("clickable")
    .on_click(|event: &ClickEvent, window: &mut Window, cx: &mut App| {
        println!("Clicked at {:?}", event.position);
    })
    .on_hover(|hovering: &bool, window: &mut Window, cx: &mut App| {
        if *hovering {
            println!("Mouse entered");
        } else {
            println!("Mouse left");
        }
    })
    .on_scroll_wheel(|event: &ScrollWheelEvent, window: &mut Window, cx: &mut App| {
        println!("Scroll delta: {:?}", event.delta);
    })
```

### 12.4.4 状态类：hover、active、focus

```rust
div()
    .id("styled-button")
    .px_4()
    .py_2()
    .rounded_md()
    .bg(blue_500())
    .text_white()
    .hover(|style| style.bg(blue_600()))           // hover 时变深色
    .active(|style| style.bg(blue_700()))           // 按下时更深
    .focus(|style| style.ring_2().ring_blue_400())  // 聚焦时显示光环
```

状态样式通过传入闭包修改 `Style`，只在对应状态激活时应用。

## 12.5 实用技巧

### 12.5.1 `id()` 的作用与重要性

```rust
// 必须给需要事件的 div 设置 id
div()
    .id("main-container")  // 没有 id 的事件注册无效
    .on_click(|_, _, cx| { ... })
    .on_hover(|_, _, cx| { ... })
```

`id()` 的作用：
1. 事件处理的前提：事件系统通过 id 追踪元素
2. Inspector 中识别元素
3. 跨帧状态持久化（hover、active 等）

ID 类型：
```rust
div().id("string_id")                    // ElementId::Name
div().id(0)                              // ElementId::Integer
div().id(("row", 5))                     // ElementId::NamedInteger (via From impl)
```

不支持 3 元素或更多元素的元组——GPUI 的 `From` 实现仅支持 `(&str, usize)` 二元组。
```

### 12.5.2 `.group()` 组合样式

```rust
div()
    .group("card")  // 定义组名
    .child(
        div()
            .child("Card title")
            .child(
                // 子元素可以引用父组的 hover 状态
                div()
                    .group_hover("card", |style| style.visible())
                    .invisible()  // 默认不可见，父 card hover 时可见
            )
    )
```

`group()` 允许子元素根据父元素的状态改变样式。

### 12.5.3 `div` 嵌套深度的性能影响

```rust
// 过深嵌套：每层都有 Interactivity 开销
div()                          // 层1
    .child(div()               // 层2
        .child(div()           // 层3
            .child(div()       // 层4
                .child(div()   // 层5
                    .child("终于到内容了")
                )
            )
        )
    )

// 推荐：扁平化
div()
    .flex()
    .gap_4()
    .p_4()
    .child("Header")
    .child("Body")
    .child("Footer")
```

经验法则：**布局需要的嵌套保留，纯粹为了样式的嵌套尽量合并**。每多一层 div，就多一个 Element 的创建、布局、绘制开销。

### 12.5.4 `image_cache()` 优化图片渲染

```rust
div()
    .image_cache(retain_all("photo-cache"))
    .child(img("photo1.jpg"))
    .child(img("photo2.jpg"))
    .child(img("photo3.jpg"))
```

`image_cache()` 在 div 位置插入图片缓存，避免相同图片重复加载。

### 12.5.5 `with_dynamic_prepaint_order()` 控制子元素 prepaint 顺序

```rust
// 在某些场景下，子元素的 prepaint 顺序很重要
div()
    .with_dynamic_prepaint_order(|window, cx| {
        // 返回子元素索引的 prepaint 顺序
        vec![1, 0, 2]  // 先 prepaint 索引1的子元素，再0，再2
    })
    .child(editor_a)
    .child(editor_b)
    .child(editor_c)
```

用于 autoscroll 等场景，需要特定子元素先更新状态。
