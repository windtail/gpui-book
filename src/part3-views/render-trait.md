# 第8章：Render Trait

## 8.1 `Render` trait 定义

### 8.1.1 `render(&mut self, window, cx) -> impl IntoElement`

```rust
use gpui::{div, button, IntoElement, Context, Window, SharedString, App, Pixels};

struct Counter {
    count: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(format!("Count: {}", self.count))
            .child(
                button()
                    .label("+1")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.count += 1;
                        cx.notify();
                    })),
            )
    }
}
```

`Render` 是 View 到 Element 树的转换协议。每次 GPUI 决定重绘一个 View 时，调用 `render()` 方法，将 View 的当前状态转化为 Element 树。

```rust
// 来自 GPUI 源码
pub trait Render: 'static + Sized {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement;
}
```

返回值 `impl IntoElement` 意味着你可以返回任何实现了 `IntoElement` 的类型：`div()`、`button()`、`"Hello"`、`Component<MyStruct>` 等。

### 8.1.2 为什么需要 `&mut self`

`render()` 签名是 `&mut self` 而非 `&self`，原因在于 GPUI 的渲染模型允许在渲染时做轻量级的状态调整：

```rust
struct ExpensiveList {
    items: Vec<String>,
    cached_display: Option<AnyElement>,
}

impl Render for ExpensiveList {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 在 render 中缓存结果，避免重复计算
        let cached = self.cached_display.get_or_insert_with(|| {
            self.items
                .iter()
                .fold(div(), |parent, item| parent.child(item.as_str()))
                .into_any_element()
        });

        div().child(cached.clone())
    }
}
```

`&mut self` 允许你在 render 中修改内部缓存，同时不会触发 `cx.notify()`（GPUI 区分了 render 内部的修改和用户操作触发的修改）。

### 8.1.3 返回值的生命周期

`render()` 返回的元素在当前帧结束后被丢弃。下一帧如果 View 仍然是 dirty 的，`render()` 会被再次调用，生成一棵全新的 Element 树。

```rust
struct Timer {
    seconds: u64,
}

impl Render for Timer {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 每次调用都返回新的 Element，旧的被丢弃
        div()
            .child(format!("Elapsed: {}s", self.seconds))
            .child(
                button()
                    .label("Reset")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.seconds = 0;
                        cx.notify();
                    })),
            )
    }
}
```

GPUI 通过 `GlobalElementId` 追踪跨帧的元素身份（第11章详述），所以你不需要担心 Element 身份问题。

## 8.2 `RenderOnce` trait

### 8.2.1 无状态组件的优化

`RenderOnce` 用于不需要持久状态的组件。它的 `render()` 获取 `self`（所有权）而非 `&mut self`：

```rust
use gpui::{div, IntoElement, Window, App, RenderOnce, SharedString, Pixels, rgb};

#[derive(IntoElement)]
struct StatusBadge {
    label: SharedString,
    active: bool,
}

impl RenderOnce for StatusBadge {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let color = if self.active { rgb(0x22c55e) } else { rgb(0x6b7280) };

        div()
            .flex()
            .items_center()
            .gap_1()
            .child(
                div()
                    .rounded_full()
                    .bg(color)
                    .size(Pixels(8.0)),
            )
            .child(div().ml_1().child(self.label))
    }
}
```

```rust
pub trait RenderOnce: 'static {
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement;
}
```

### 8.2.2 `RenderOnce` vs `Render` 的选择

| 场景 | 用 `Render` | 用 `RenderOnce` |
|------|-------------|-----------------|
| View 根类型 | 是 | 否 |
| 需要保持内部状态 | 是 | 否 |
| 只读数据展示 | 否 | 是 |
| 可复用 UI 片段 | 否 | 是 |
| 需要 `cx.observe()` | 是 | 否 |

经验法则：**根 View 用 `Render`，子组件用 `RenderOnce`**。

```rust
// 根 View：需要状态，用 Render
struct TodoList {
    items: Vec<String>,
    filter: String,
}
impl Render for TodoList { ... }

// 子组件：纯展示，用 RenderOnce
#[derive(IntoElement)]
struct TodoItem {
    text: SharedString,
    completed: bool,
}
impl RenderOnce for TodoItem { ... }
```

### 8.2.3 `Component<C>` 包装器

`#[derive(IntoElement)]` 宏为 `RenderOnce` 类型自动生成 `IntoElement` 实现，内部使用 `Component<C>` 包装：

```rust
// 你写的代码
#[derive(IntoElement)]
struct Avatar { url: SharedString }
impl RenderOnce for Avatar { ... }

// 宏生成的等价代码（简化版）
impl IntoElement for Avatar {
    type Element = Component<Self>;
    fn into_element(self) -> Self::Element {
        Component::new(self)
    }
}
```

`Component<C>` 本身实现了 `Element` trait。它在 `request_layout` 阶段调用 `C::render()` 生成子 Element 树，然后将布局委托给子元素。

## 8.3 从 render 返回元素树

### 8.3.1 `div()` 作为根元素

几乎所有 `render()` 方法都以 `div()` 作为根元素：

```rust
impl Render for UserProfile {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .gap_2()
            .p_4()
            .child(div().text_xl().font_semibold().child(self.name.as_str()))
            .child(div().text_sm().text_gray_500().child(self.bio.as_str()))
            .child(
                div()
                    .flex()
                    .gap_2()
                    .child(button().label("Follow").on_click(cx.listener(|this, _, _, cx| {
                        this.following = true;
                        cx.notify();
                    })))
                    .child(button().label("Message").on_click(cx.listener(|this, _, _, cx| {
                        this.show_message = true;
                        cx.notify();
                    }))),
            )
    }
}
```

`div()` 提供了一个容器，可以在其上应用布局、样式和事件处理。

### 8.3.2 条件渲染

使用 `.when()` 方法链进行条件渲染：

```rust
struct Notification {
    unread: bool,
    avatar_url: Option<SharedString>,
    message: SharedString,
    timestamp: Option<SharedString>,
}

impl Render for Notification {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .when(self.unread, |this| this.bg(Colors::get(cx).background_accent))
            .when_some(self.avatar_url.clone(), |this, url| {
                this.child(img(url.as_str()).size(Pixels(32.0)))
            })
            .child(div().child(self.message.as_str()))
            .when_some(self.timestamp.clone(), |this, ts| {
                this.child(div().text_xs().text_gray_400().child(ts.as_str()))
            })
    }
}
```

`.when()` 和 `.when_some()` 都是 `FluentBuilder` 上的方法，接受闭包返回修改后的 builder。

### 8.3.3 列表渲染与迭代

使用 `.children()` 配合迭代器渲染列表：

```rust
struct FileList {
    files: Vec<FileInfo>,
    selected_index: Option<usize>,
}

impl Render for FileList {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex_col()
            .children(
                self.files.iter().enumerate().map(|(i, file)| {
                    div()
                        .id(("file", i))
                        .p_2()
                        .hover(|style| style.bg(Colors::get(cx).element_active))
                        .on_click(cx.listener(move |this, _, _, cx| {
                            this.selected_index = Some(i);
                            cx.notify();
                        }))
                        .child(file.name.as_str())
                }),
            )
    }
}
```

注意 `id()` 的用法：使用 `("file", i)` 为每个元素生成唯一 ID，这对事件处理和 Inspector 调试都重要。

## 8.4 视图作为 UI 状态容器

### 8.4.1 View struct 应该包含什么

View 的 struct 只应包含**渲染所需的数据**和**组件内部状态**：

```rust
struct SearchBar {
    query: SharedString,         // 输入状态
    results: Vec<SearchResult>,   // 搜索结果
    is_loading: bool,             // 加载状态
    selected_index: usize,        // UI 焦点状态
    // 不需要放：数据库连接、网络客户端等
}
```

不属于 UI 状态的数据（数据库连接、线程池、文件句柄）应该放在独立的 Entity 中，通过 `Entity<T>` 句柄引用。

### 8.4.2 计算属性与缓存

在 render 中计算复杂结果时，使用内部字段缓存：

```rust
struct DataTable {
    rows: Vec<Row>,
    sort_column: Option<ColumnId>,
    sort_ascending: bool,
    sorted_rows: Option<Vec<Row>>,  // 缓存
}

impl Render for DataTable {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let rows = self.sorted_rows.get_or_insert_with(|| {
            let mut sorted = self.rows.clone();
            if let Some(col) = self.sort_column {
                sorted.sort_by(|a, b| {
                    let cmp = a.get(col).cmp(&b.get(col));
                    if self.sort_ascending { cmp } else { cmp.reverse() }
                });
            }
            sorted
        });

        div().children(rows.iter().map(|row| {
            div().child(row.name.as_str())
        }))
    }
}
```

当排序条件改变时，将 `sorted_rows` 设为 `None` 强制重新计算。

### 8.4.3 性能敏感的 render 优化

避免在 render 中做耗时操作：

```rust
// 错误：render 中执行文件读取
impl Render for FileBrowser {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let files = std::fs::read_dir(&self.path).unwrap()  // 阻塞渲染！
            .collect::<Vec<_>>();
        div().children(files.iter().map(|f| div().child(f.to_string_lossy().as_ref())))
    }
}

// 正确：异步加载，render 只读取缓存结果
struct FileBrowser {
    path: PathBuf,
    files: Option<Vec<PathBuf>>,
}

impl FileBrowser {
    fn load_files(&mut self, cx: &mut Context<Self>) {
        let path = self.path.clone();
        cx.spawn(|this, cx| async move {
            let files = std::fs::read_dir(&path)?
                .filter_map(|e| e.ok().map(|e| e.path()))
                .collect::<Vec<_>>();
            this.update(&mut cx, |this, cx| {
                this.files = Some(files);
                cx.notify();
            })
        }).detach();
    }
}

impl Render for FileBrowser {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        match &self.files {
            None => div().child("Loading..."),
            Some(files) => div().children(files.iter().map(|f| {
                div().child(f.to_string_lossy().as_ref())
            })),
        }
    }
}
```

## 8.5 `#[derive(IntoElement)]`

### 8.5.1 什么时候需要这个 derive

当你的类型实现了 `RenderOnce` 并且想在元素链中直接使用时，需要 `IntoElement`：

```rust
// 没有 derive，无法这样写
div()
    .child(StatusBadge { label: "Active".into(), active: true })  // 编译错误

// 加上 derive
#[derive(IntoElement)]
struct StatusBadge { label: SharedString, active: bool }

// 现在可以正常使用了
div()
    .child(StatusBadge { label: "Active".into(), active: true })  // OK
```

### 8.5.2 与 `RenderOnce` 的关系

`#[derive(IntoElement)]` 要求类型实现了 `RenderOnce`：

```rust
#[derive(IntoElement)]
struct Card {
    title: SharedString,
    body: SharedString,
}

// 必须配套实现 RenderOnce
impl RenderOnce for Card {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div()
            .border_1()
            .rounded_md()
            .p_4()
            .child(div().font_semibold().child(self.title))
            .child(div().child(self.body))
    }
}
```

### 8.5.3 常见编译错误

**错误1：缺少 `RenderOnce` 实现**

```
error[E0277]: the trait bound `Card: RenderOnce` is not satisfied
```

解决：为类型实现 `RenderOnce`。

**错误2：字段没有实现 `IntoElement`**

```rust
#[derive(IntoElement)]
struct BadComponent {
    custom_type: MyStruct,  // MyStruct 没有实现必要 trait
}
```

解决：让 `MyStruct` 实现 `IntoElement` 或改用 `SharedString` 等基本类型。

**错误3：生命周期问题**

```rust
#[derive(IntoElement)]
struct BadComponent<'a> {
    text: &'a str,  // 生命周期参数与 'static 要求冲突
}
```

解决：使用 `SharedString` 代替 `&str`，因为 `RenderOnce` 要求 `'static`。
