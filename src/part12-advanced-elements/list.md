# 第 43 章：List 虚拟列表

## 43.1 为什么需要虚拟列表

### 43.1.1 大量数据的性能问题

用普通的 `.children()` 渲染大量元素会导致严重性能问题：

```rust
// 错误做法：渲染 10 万个 div
div().children(
    (0..100_000).map(|i| div().child(format!("Item {}", i)))
)
// 每一帧都要创建 10 万个 AnyElement，即使只有 20 个可见
```

### 43.1.2 视口渲染原理

虚拟列表（virtual list）只渲染视口内的元素：

```
总高度 = 所有元素高度之和（通过 SumTree 计算）
视口 = 用户可见区域
渲染 = 只创建视口内 + 少量 overdraw 的 AnyElement
滚动 = 更新 logical_scroll_top，重新计算可见范围
```

## 43.2 `ListState`

### 43.2.1 创建与配置

`ListState` 是侵入式状态，存储在 View 上：

```rust
use gpui::{list, List, ListState, ListAlignment};

struct MyListView {
    list_state: ListState,
    items: Vec<String>,
}

impl MyListView {
    fn new(items: Vec<String>) -> Self {
        let count = items.len();
        let state = ListState::new(
            count,                          // 初始项数
            ListAlignment::Top,             // 从顶部对齐（正常滚动）
            px(1000.0),                     // overdraw：视口上下各多渲染 1000px
        );

        Self {
            list_state: state,
            items,
        }
    }
}
```

`ListAlignment::Bottom` 用于聊天日志等从底部向上滚动的场景。

### 43.2.2 渲染项回调

`list()` 函数接收 `ListState` 和渲染回调：

```rust
impl Render for MyListView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let state = self.list_state.clone();
        let items = self.items.clone(); // 或 &self.items 通过闭包捕获

        list(state, move |ix, _window, _cx| {
            // ix 是当前项的索引
            let item = items.get(ix).map(|s| s.as_str()).unwrap_or("");
            div()
                .h_8()
                .px_3()
                .child(format!("Item {}: {}", ix, item))
                .into_any_element()
        })
    }
}
```

渲染回调签名：`FnMut(usize, &mut Window, &mut App) -> AnyElement`。

### 43.2.3 状态管理

`ListState` 内部维护：
- 每项的测量高度（首次渲染时测量）
- 当前滚动位置（`logical_scroll_top`）
- 焦点句柄跟踪

```rust
// 获取当前项数
let count = state.item_count();

// 获取滚动位置
let scroll_top = state.logical_scroll_top();
// ListOffset { item_ix: 42, offset_in_item: px(5.0) }

// 手动滚动
state.scroll_by(px(100.0));

// 滚动到底部
state.scroll_to_bottom();

// 滚动到指定项
state.scroll_to_reveal_item(100);
```

## 43.3 `list()` 元素

### 43.3.1 基本用法

```rust
use gpui::{div, list, List, ListState, ListSizingBehavior, prelude::*};

fn build_list(state: ListState) -> List {
    list(state, |ix, _window, _cx| {
        div()
            .h_6()
            .px_2()
            .child(format!("Row {}", ix))
            .into_any_element()
    })
    .with_sizing_behavior(ListSizingBehavior::Auto)
}
```

### 43.3.2 滚动集成

将 List 放在有 overflow 的容器中：

```rust
impl Render for MyListView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .w_full()
            .h_full()
            .overflow_hidden()
            .child(list(self.list_state.clone(), |ix, _window, _cx| {
                self.render_item(ix)
            }))
    }

    fn render_item(&self, ix: usize) -> AnyElement {
        div()
            .h_8()
            .border_b_1()
            .px_4()
            .child(self.items.get(ix).map(|s| s.as_str()).unwrap_or(""))
            .into_any_element()
    }
}
```

## 43.4 可变高度项

### 43.4.1 高度估算

List 使用 `SumTree` 跟踪每项的高度。未渲染过的项标记为 `Unmeasured`，使用 `size_hint` 估算。

```rust
// ListState 内部
enum ListItem {
    Unmeasured {
        size_hint: Option<Size<Pixels>>, // 估算尺寸
        focus_handle: Option<FocusHandle>,
    },
    Measured {
        size: Size<Pixels>,              // 精确尺寸
        focus_handle: Option<FocusHandle>,
    },
}
```

### 43.4.2 测量与修正

首次渲染时，List 会测量视口内的项来修正估算值。后续滚动时复用已测量的高度。

如果项的高度在运行时改变（如动态内容展开/折叠），需要通知 List 重新测量：

```rust
// 使用 remeasure_items 标记需要重新测量的范围
state.remeasure_items(5..10); // 第 5-9 项需要重新测量

// 或者用 remeasure 重新测量所有项
state.remeasure();
```

## 43.5 数据更新

### 43.5.1 `splice()` 局部更新

`splice()` 替换指定范围的项：

```rust
// 在位置 5 插入 3 个新项（替换 0 个旧项）
state.splice(5..5, 3);

// 删除位置 10-12 的 3 个项
state.splice(10..13, 0);

// 替换位置 0-2 的 3 个项为 5 个新项
state.splice(0..3, 5);
```

配合数据模型：

```rust
impl MyListView {
    fn insert_item(&mut self, index: usize, item: String) {
        self.items.insert(index, item);
        // 通知 List：在 index 处插入了 1 个项
        self.list_state.splice(index..index, 1);
    }

    fn remove_item(&mut self, index: usize) {
        self.items.remove(index);
        // 通知 List：在 index 处删除了 1 个项
        self.list_state.splice(index..index + 1, 0);
    }

    fn replace_items(&mut self, new_items: Vec<String>) {
        let old_len = self.items.len();
        self.items = new_items;
        self.list_state.splice(0..old_len, self.items.len());
    }
}
```

### 43.5.2 `reset()` 全量重置

`reset()` 完全重建列表状态，重置滚动位置到顶部：

```rust
fn reload_data(&mut self, new_items: Vec<String>) {
    self.items = new_items;
    self.list_state.reset(self.items.len());
    // 滚动位置被重置到顶部
}
```

`reset()` 和 `splice(0..old_len, new_len)` 的区别：
- `reset()` 清除 `logical_scroll_top`，滚回顶部
- `splice()` 尽量保持滚动位置不变

## 43.6 实战：大型数据列表

```rust
use gpui::{div, list, ListState, ListAlignment, Context, Render, Styled, prelude::*};
use std::time::Duration;

struct LargeListView {
    list_state: ListState,
    items: Vec<LogEntry>,
    filter: String,
}

struct LogEntry {
    timestamp: String,
    level: LogLevel,
    message: String,
}

enum LogLevel { Debug, Info, Warn, Error }

impl LargeListView {
    fn new(entries: Vec<LogEntry>) -> Self {
        let count = entries.len();
        Self {
            list_state: ListState::new(count, ListAlignment::Top, px(500.0)),
            items: entries,
            filter: String::new(),
        }
    }

    fn append_entry(&mut self, entry: LogEntry) {
        let index = self.items.len();
        self.items.push(entry);
        self.list_state.splice(index..index, 1);
        // 自动滚动到底部
        self.list_state.scroll_to_bottom();
    }

    fn render_entry(&self, ix: usize) -> AnyElement {
        let entry = &self.items[ix];
        let level_color = match entry.level {
            LogLevel::Debug => rgb(0x888888),
            LogLevel::Info => rgb(0x4488ff),
            LogLevel::Warn => rgb(0xffaa00),
            LogLevel::Error => rgb(0xff4444),
        };

        div()
            .flex()
            .gap_2()
            .px_2()
            .py_0p5()
            .child(div().text_color(level_color).child(&entry.level_label()))
            .child(div().text_color(rgb(0x666666)).child(&entry.timestamp))
            .child(div().child(&entry.message))
            .into_any_element()
    }
}

impl Render for LargeListView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let state = self.list_state.clone();

        div()
            .w_full()
            .h_full()
            .overflow_hidden()
            .bg(rgb(0x1e1e1e))
            .child(list(state, {
                let this = self;
                move |ix, _window, _cx| this.render_entry(ix)
            }))
    }
}
```

使用 List 的关键规则：
- `ListState` 存在 View 的字段上
- `list(state, render_fn)` 创建虚拟列表元素
- 数据变更时调用 `splice()` 或 `reset()` 通知 List
- overdraw 控制预渲染范围，影响滚动流畅度和内存使用
