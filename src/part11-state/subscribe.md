# 第 40 章：Subscribe 模式

## 40.1 `EventEmitter<E>` trait

### 40.1.1 定义事件类型

Subscribe 模式基于事件驱动。首先定义事件类型：

```rust
// 事件是普通的 struct 或 enum
enum BufferEvent {
    TextChanged { old: String, new: String },
    Saved { path: String },
    Closed,
}

// 带数据的 struct 事件
struct FileOpenedEvent {
    path: String,
    size: usize,
    encoding: String,
}
```

### 40.1.2 实现 trait

在 Entity 上实现 `EventEmitter<E>` 来启用事件发射：

```rust
use gpui::{Context, EventEmitter};

struct Buffer {
    text: String,
}

impl EventEmitter<BufferEvent> for Buffer {}

impl Buffer {
    fn set_text(&mut self, new_text: String, cx: &mut Context<Self>) {
        let old = std::mem::replace(&mut self.text, new_text.clone());
        cx.emit(BufferEvent::TextChanged { old, new: new_text });
    }

    fn save(&mut self, cx: &mut Context<Self>) {
        cx.emit(BufferEvent::Saved { path: "/path/to/file".into() });
    }
}
```

`EventEmitter` 是 marker trait，空实现即可。关键是在方法中调用 `cx.emit()`。

## 40.2 `cx.emit()`

### 40.2.1 发射事件

`cx.emit(event)` 将事件广播给所有订阅者：

```rust
struct TabBar {
    active_tab: usize,
}

enum TabEvent {
    TabClosed(usize),
    TabSelected(usize),
    TabAdded(String),
}

impl EventEmitter<TabEvent> for TabBar {}

impl TabBar {
    fn close_tab(&mut self, index: usize, cx: &mut Context<Self>) {
        // 发射事件
        cx.emit(TabEvent::TabClosed(index));
    }

    fn select_tab(&mut self, index: usize, cx: &mut Context<Self>) {
        self.active_tab = index;
        cx.emit(TabEvent::TabSelected(index));
    }
}
```

### 40.2.2 事件类型的设计

一个 Entity 可以实现多个 `EventEmitter`：

```rust
struct Editor {}

enum EditorEvent {
    BufferChanged,
    CursorMoved { row: usize, col: usize },
    SelectionChanged,
}

enum ScrollEvent {
    ScrolledToBottom,
    ScrolledToTop,
}

// 实现多个 EventEmitter
impl EventEmitter<EditorEvent> for Editor {}
impl EventEmitter<ScrollEvent> for Editor {}

impl Editor {
    fn scroll(&mut self, offset: f32, cx: &mut Context<Self>) {
        if offset >= 1.0 {
            cx.emit(ScrollEvent::ScrolledToBottom);
        }
    }
}
```

## 40.3 `cx.subscribe()`

### 40.3.1 订阅其他 Entity 的事件

`cx.subscribe()` 注册回调来监听特定 Entity 的事件：

```rust
struct Workspace {
    tab_bar: Entity<TabBar>,
    open_files: Vec<String>,
    _tab_subscription: gpui::Subscription,
}

impl Workspace {
    fn new(cx: &mut Context<Self>) -> Self {
        let tab_bar = cx.new(|cx| {
            TabBar { active_tab: 0 }
        });

        // 订阅 tab_bar 的事件
        let _tab_subscription = cx.subscribe(&tab_bar, |this, tab_bar, event, cx| {
            match event {
                TabEvent::TabClosed(index) => {
                    this.open_files.remove(*index);
                    cx.notify();
                }
                TabEvent::TabSelected(index) => {
                    // 切换激活的 tab
                    cx.notify();
                }
                TabEvent::TabAdded(name) => {
                    this.open_files.push(name.clone());
                    cx.notify();
                }
            }
        });

        Self {
            tab_bar,
            open_files: Vec::new(),
            _tab_subscription,
        }
    }
}
```

### 40.3.2 Handler 签名

回调签名是 `FnMut(&mut Self, &Entity<T>, &Event, &mut Context<Self>)`：

```rust
cx.subscribe(&entity, |this, entity, event, cx| {
    // this: &mut Self    —— 订阅者自身
    // entity: &Entity<T> —— 事件发射者
    // event: &Event      —— 事件数据
    // cx: &mut Context<Self>
});
```

### 40.3.3 取消订阅

`Subscription` 被 drop 时自动取消订阅：

```rust
// 保存 Subscription 在字段中
struct MyView {
    _sub: gpui::Subscription,
}

// 不需要订阅时，清空字段
fn stop_listening(&mut self) {
    self._sub = gpui::Subscription::new(|| {});
    // 或者重新赋值一个新的 Subscription
}
```

## 40.4 `cx.subscribe_self()`

`cx.subscribe_self()` 让 Entity 订阅自身发射的事件：

```rust
struct Counter {
    value: i32,
    history: Vec<i32>,
    _self_sub: gpui::Subscription,
}

enum CounterEvent {
    Changed(i32),
    Reset,
}

impl EventEmitter<CounterEvent> for Counter {}

impl Counter {
    fn new(cx: &mut Context<Self>) -> Self {
        let _self_sub = cx.subscribe_self(|this, event, cx| {
            match event {
                CounterEvent::Changed(val) => {
                    this.history.push(*val);
                }
                CounterEvent::Reset => {
                    this.history.clear();
                }
            }
        });

        Self {
            value: 0,
            history: Vec::new(),
            _self_sub,
        }
    }

    fn increment(&mut self, cx: &mut Context<Self>) {
        self.value += 1;
        cx.emit(CounterEvent::Changed(self.value));
        // history 的更新由 subscribe_self 回调处理
    }
}
```

## 40.5 Event vs Observe

### 40.5.1 什么时候用 emit/subscribe

- 事件携带语义信息。`BufferEvent::Saved` 比 `observe` 的 "buffer 被修改了" 更具体
- 事件只在显式调用 `cx.emit()` 时触发，`observe` 在每次 `entity.update()` 时触发
- 多个不同类型的事件可以共存

```rust
// emit/subscribe：语义明确
cx.subscribe(&buffer, |this, buffer, event, cx| {
    match event {
        BufferEvent::Saved { path } => this.show_saved_notification(path),
        BufferEvent::Closed => this.remove_buffer(),
        BufferEvent::TextChanged { .. } => this.update_word_count(),
    }
});
```

### 40.5.2 什么时候用 observe

- 关心所有变化，不需要区分具体类型
- 被观察的 Entity 没有实现 `EventEmitter`
- 简单的"状态变了就重绘"场景

```rust
// observe：简单直接
cx.observe(&config, |this, config, cx| {
    this.apply_config(config.read(cx));
    cx.notify();
});
```

### 40.5.3 混合使用

```rust
struct EditorView {
    buffer: Entity<Buffer>,
    _observe_sub: gpui::Subscription,
    _event_sub: gpui::Subscription,
}

impl EditorView {
    fn new(buffer: Entity<Buffer>, cx: &mut Context<Self>) -> Self {
        // observe 用于通用状态同步
        let _observe_sub = cx.observe(&buffer, |this, buffer, cx| {
            this.dirty = buffer.read(cx).is_dirty();
        });

        // subscribe 用于特定事件处理
        let _event_sub = cx.subscribe(&buffer, |this, buffer, event, cx| {
            match event {
                BufferEvent::Saved { path } => {
                    this.show_notification(&format!("Saved: {}", path));
                }
                _ => {}
            }
        });

        Self {
            buffer,
            _observe_sub,
            _event_sub,
        }
    }
}
```

## 40.6 实战：事件驱动的通知系统

```rust
use gpui::{div, Context, Entity, EventEmitter, Render, Styled, Subscription, prelude::*};

// 事件定义
enum NotificationEvent {
    Info(String),
    Warning(String),
    Error(String),
}

// 通知中心 —— 发射事件
struct NotificationCenter {
    notifications: Vec<Notification>,
}

struct Notification {
    message: String,
    level: NotificationLevel,
}

enum NotificationLevel { Info, Warning, Error }

impl EventEmitter<NotificationEvent> for NotificationCenter {}

impl NotificationCenter {
    fn push(&mut self, event: NotificationEvent, cx: &mut Context<Self>) {
        cx.emit(event);
    }

    fn clear(&mut self, cx: &mut Context<Self>) {
        self.notifications.clear();
        cx.notify();
    }
}

// 订阅者 —— 监听通知
struct StatusBar {
    notification_center: Entity<NotificationCenter>,
    current_message: Option<String>,
    _sub: Subscription,
}

impl StatusBar {
    fn new(nc: Entity<NotificationCenter>, cx: &mut Context<Self>) -> Self {
        let _sub = cx.subscribe(&nc, |this, _nc, event, cx| {
            match event {
                NotificationEvent::Info(msg) => {
                    this.current_message = Some(format!("ℹ {}", msg));
                    cx.notify();
                }
                NotificationEvent::Warning(msg) => {
                    this.current_message = Some(format!("⚠ {}", msg));
                    cx.notify();
                }
                NotificationEvent::Error(msg) => {
                    this.current_message = Some(format!("✖ {}", msg));
                    cx.notify();
                }
            }
        });

        Self {
            notification_center: nc,
            current_message: None,
            _sub,
        }
    }
}

impl Render for StatusBar {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div().p_2().children(self.current_message.as_deref().map(|m| m.to_string()))
    }
}
```

Subscribe 模式的核心规则：
- 实现 `EventEmitter<E>` 空 trait 启用事件
- `cx.emit(event)` 广播事件
- `cx.subscribe(&entity, ...)` 监听特定 Entity 的事件
- `cx.subscribe_self(...)` 监听自身事件
- `Subscription` 是 RAII 句柄，drop 时自动取消
