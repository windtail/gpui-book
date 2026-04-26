# 第 44 章：UniformList

## 44.1 何时使用 UniformList

### 44.1.1 所有项高度相同

当列表每项高度一致时，`UniformList` 比 `List` 更简单高效。它不依赖 Taffy 布局系统，而是测量第一个元素后直接线性排列其余元素。

```rust
// 每项高度固定为 32px
uniform_list(cx.view().entity_id(), 1000, |this, range, cx| {
    range.map(|ix| div().h_8().child(format!("Item {}", ix))).collect()
})
```

### 44.1.2 比 List 更简单高效

对比 `List` 和 `UniformList`：

| 特性 | List | UniformList |
|------|------|-------------|
| 项高度 | 可变 | 固定 |
| 高度计算 | SumTree | 测量首项 |
| 状态 | `ListState` | 无需外部状态 |
| API 复杂度 | 较高 | 较低 |

当所有项等高时，选 `UniformList`。

## 44.2 `uniform_list()` 函数

### 44.2.1 基本用法

```rust
use gpui::uniform_list;

fn build_list(count: usize) -> impl IntoElement {
    uniform_list("my-list", count, |_this, range, _cx| {
        range
            .map(|ix| {
                div()
                    .h_8()       // 固定高度
                    .px_3()
                    .child(format!("Row {}", ix))
            })
            .collect::<Vec<_>>()
    })
}
```

参数：
- `id`: `impl Into<ElementId>` 唯一标识
- `item_count`: `usize` 总项数
- `f`: `Fn(Range<usize>, &mut Window, &mut App) -> Vec<R>` 渲染可见范围

回调接收 `Range<usize>`（可见项索引范围），返回 `Vec<impl IntoElement>`。

### 44.2.2 项渲染回调

在回调中只创建可见项：

```rust
struct FileList {
    files: Vec<String>,
}

impl FileList {
    fn render_list(&self, count: usize) -> impl IntoElement {
        let files = &self.files;
        uniform_list("file-list", count, move |_this, visible_range, _cx| {
            visible_range
                .filter_map(|ix| {
                    files.get(ix).map(|file| {
                        div()
                            .h_6()
                            .px_2()
                            .hover(|this| this.bg(rgb(0x333333)))
                            .child(file.clone())
                    })
                })
                .collect::<Vec<_>>()
        })
    }
}
```

## 44.3 滚动控制

### 44.3.1 `UniformListScrollHandle`

`UniformListScrollHandle` 控制滚动位置，需要存在 View 的字段中：

```rust
use gpui::{uniform_list, UniformListScrollHandle};

struct ScrollableList {
    scroll_handle: UniformListScrollHandle,
}

impl ScrollableList {
    fn new() -> Self {
        Self {
            scroll_handle: UniformListScrollHandle::new(),
        }
    }

    fn render(&self, count: usize) -> impl IntoElement {
        uniform_list("scroll-list", count, |_this, range, _cx| {
            range
                .map(|ix| div().h_8().child(format!("Item {}", ix)))
                .collect::<Vec<_>>()
        })
        .with_scroll_handle(self.scroll_handle.clone())
    }
}
```

### 44.3.2 `scroll_to_item()` 滚动到指定项

```rust
// 滚动到第 50 项
self.scroll_handle.scroll_to_item(50, ScrollStrategy::Top);

// 严格滚动（即使项已可见也强制滚动）
self.scroll_handle.scroll_to_item_strict(50, ScrollStrategy::Center);

// 带偏移的滚动
self.scroll_handle.scroll_to_item_with_offset(
    50,
    ScrollStrategy::Top,
    3, // 在目标位置上方留 3 个项的空间
);
```

### 44.3.3 `ScrollStrategy`：Top/Center/Bottom/Nearest

```rust
use gpui::ScrollStrategy;

// Top：项的顶部对齐视口顶部
scroll_handle.scroll_to_item(ix, ScrollStrategy::Top);

// Center：项居中显示
scroll_handle.scroll_to_item(ix, ScrollStrategy::Center);

// Bottom：项的底部对齐视口底部
scroll_handle.scroll_to_item(ix, ScrollStrategy::Bottom);

// Nearest：项不可见时滚动最小距离使其可见
scroll_handle.scroll_to_item(ix, ScrollStrategy::Nearest);
```

## 44.4 装饰器

### 44.4.1 `UniformListDecoration` trait

`UniformListDecoration` 允许在 UniformList 上添加覆盖层装饰：

```rust
use gpui::{UniformListDecoration, AnyElement, Pixels, Bounds, Point, App, Window};

struct StripedRows;

impl UniformListDecoration for StripedRows {
    fn compute(
        &self,
        visible_range: Range<usize>,
        bounds: Bounds<Pixels>,
        scroll_offset: Point<Pixels>,
        item_height: Pixels,
        _window: &mut Window,
        _cx: &mut App,
    ) -> Vec<AnyElement> {
        visible_range
            .filter(|ix| ix % 2 == 1)
            .map(|ix| {
                let y = ix as f32 * item_height.0 + scroll_offset.y.0;
                div()
                    .absolute()
                    .top(px(y))
                    .left_0()
                    .right_0()
                    .h(item_height)
                    .bg(rgb(0x222222))
                    .into_any_element()
            })
            .collect()
    }
}
```

### 44.4.2 自定义行样式

使用装饰器添加条纹背景：

```rust
fn build_list(count: usize) -> impl IntoElement {
    uniform_list("my-list", count, |_this, range, _cx| {
        range
            .map(|ix| div().h_8().child(format!("Item {}", ix)))
            .collect::<Vec<_>>()
    })
    .with_decoration(StripedRows)
}
```

UniformList 的适用场景：
- 文件列表、表格行、菜单项等所有项等高的场景
- 不需要 `ListState`，API 更简洁
- 通过 `UniformListScrollHandle` 控制滚动
- 需要装饰时用 `UniformListDecoration`
