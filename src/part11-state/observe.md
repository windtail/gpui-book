# 第 39 章：Observe 模式

## 39.1 `cx.observe()`

### 39.1.1 观察外部 Entity

`cx.observe()` 让一个 Entity 监听另一个 Entity 的状态变化。被观察的 Entity 每次被 `update` 后，回调被触发。

```rust
struct Editor {
    buffer: Entity<Buffer>,
    word_count: usize,
    _buffer_observer: gpui::Subscription,
}

impl Editor {
    fn new(buffer: Entity<Buffer>, cx: &mut Context<Self>) -> Self {
        let word_count = buffer.read(cx).text().split_whitespace().count();

        // 观察 buffer 的变化
        let _buffer_observer = cx.observe(&buffer, |this: &mut Editor, buffer: Entity<Buffer>, cx: &mut Context<Editor>| {
            let count = buffer.read(cx).text().split_whitespace().count();
            this.word_count = count;
            cx.notify(); // buffer 变了，通知 UI 重绘
        });

        Self {
            buffer,
            word_count,
            _buffer_observer,
        }
    }
}
```

### 39.1.2 回调签名与触发条件

回调签名是 `FnMut(&mut Self, &Entity<T>, &mut Context<Self>)`：

```rust
cx.observe(&entity, |this, entity, cx| {
    // this: &mut Self        —— 观察者自身（可变）
    // entity: &Entity<T>     —— 被观察的 Entity 的句柄
    // cx: &mut Context<Self> —— 上下文

    let data = entity.read(cx); // 读取被观察者的最新状态
    this.update_from(&data);
    cx.notify();
});
```

触发条件：被观察的 Entity 被 `entity.update(cx, ...)` 调用时，观察者回调被触发。读取（`read`）不触发。

### 39.1.3 回调中的 `cx.notify()`

observe 回调负责决定是否通知 UI 重绘。如果不调用 `cx.notify()`，即使被观察的 Entity 变了，当前 View 也不会重新渲染。

```rust
// 需要 UI 响应
cx.observe(&buffer, |this, buffer, cx| {
    this.dirty = true;
    cx.notify(); // 触发 render() 重新执行
});

// 不需要 UI 响应（内部状态更新）
cx.observe(&config, |this, config, cx| {
    this.apply_config(config.read(cx));
    // 不调用 cx.notify() —— 不触发重绘
});
```

## 39.2 `cx.observe_self()`

### 39.2.1 自观察模式

`cx.observe_self()` 让 Entity 观察自己。当自身被 `update` 时触发回调。

```rust
impl MyView {
    fn new(cx: &mut Context<Self>) -> Self {
        let _self_observer = cx.observe_self(|this, cx| {
            println!("自身被更新了");
            // 通常这里需要 cx.notify() 但要注意避免无限循环
        });

        Self { _self_observer }
    }
}
```

### 39.2.2 使用场景

`observe_self` 的典型用法是聚合多个内部状态后统一通知：

```rust
struct Aggregator {
    count: usize,
    sum: f64,
    _self_observer: gpui::Subscription,
    pending_notify: bool,
}

impl Aggregator {
    fn new(cx: &mut Context<Self>) -> Self {
        let _self_observer = cx.observe_self(|this, cx| {
            if this.pending_notify {
                this.pending_notify = false;
                cx.notify();
            }
        });

        Self {
            count: 0,
            sum: 0.0,
            _self_observer,
            pending_notify: false,
        }
    }

    fn increment(&mut self, value: f64, cx: &mut Context<Self>) {
        self.count += 1;
        self.sum += value;
        self.pending_notify = true;
        // 不需要手动 cx.notify() —— observe_self 会处理
    }
}
```

实际上更常见的做法是直接在方法中调用 `cx.notify()`，`observe_self` 使用场景有限。

## 39.3 `cx.observe_global()`

### 39.3.1 观察全局状态变化

`cx.observe_global::<T>()` 监听全局状态的变更。当全局状态通过 `cx.update_global()` 或 `cx.set_global()` 被修改时，回调被触发。

```rust
struct ThemeAwareView {
    background_color: Hsla,
    _theme_observer: gpui::Subscription,
}

impl ThemeAwareView {
    fn new(cx: &mut Context<Self>) -> Self {
        let theme = cx.global::<AppTheme>();
        let background_color = theme.background;

        let _theme_observer = cx.observe_global::<AppTheme>(|this, cx| {
            // 主题变了，重新读取并应用
            let theme = cx.global::<AppTheme>();
            this.background_color = theme.background;
            cx.notify();
        });

        Self {
            background_color,
            _theme_observer,
        }
    }
}
```

### 39.3.2 自动响应主题切换

主题切换是 `observe_global` 最常见的场景：

```rust
struct AppTheme {
    background: Hsla,
    foreground: Hsla,
}

impl Global for AppTheme {}

// 任何 UI 组件都可以观察主题变化
impl StatusBar {
    fn new(cx: &mut Context<Self>) -> Self {
        let theme = cx.global::<AppTheme>();

        let _theme_observer = cx.observe_global::<AppTheme>(|this, cx| {
            let theme = cx.global::<AppTheme>();
            this.bar_color = theme.foreground;
            cx.notify();
        });

        Self {
            bar_color: theme.foreground,
            _theme_observer,
        }
    }
}

// 主题切换只需一行
fn switch_theme(cx: &mut App, new_theme: AppTheme) {
    cx.set_global(new_theme);
    // 所有 observe_global::<AppTheme> 的回调自动触发
}
```

## 39.4 Observer 回调生命周期

### 39.4.1 回调何时被调用

observe 回调在 `entity.update()` 完成后、当前 effect 刷新阶段被调用：

```rust
// 步骤 1: entity.update()
entity.update(cx, |entity, cx| {
    entity.value = 42;
    // 这里不触发 observe 回调
});
// update 返回后，effect 系统收集到 entity 被修改
// 步骤 2: 所有 observe(&entity) 的回调被调用
// 步骤 3: 回调中的 cx.notify() 被记录
// 步骤 4: 帧渲染时，被 notify 的 view 重新渲染
```

### 39.4.2 回调的清理

observe 回调被包装在 `Subscription` 中。`Subscription` 被 drop 时，回调被移除：

```rust
// 将 Subscription 存在字段中
struct MyView {
    _observer: gpui::Subscription,
}

// drop MyView 时，_observer 被 drop，回调被移除
```

如果不保存 `Subscription`，回调仍然有效，但无法手动取消：

```rust
// 回调会一直存在直到被观察的 Entity 被 drop
cx.observe(&buffer, |this, buffer, cx| {
    this.update(buffer.read(cx), cx);
});
```

## 39.5 性能考量

### 39.5.1 过多 observe 的影响

每个 observe 都会在 Entity update 时触发一次回调。如果大量 View observe 同一个 Entity，一次 update 会触发大量回调。

```rust
// 问题场景：1000 个 View observe 同一个 config
for _ in 0..1000 {
    cx.new(|cx| ConfigView::new(cx)); // 每个都 observe_global::<AppConfig>
}

cx.update_global::<AppConfig, _>(|config, cx| {
    config.theme = Dark;
});
// 1000 个 observe_global 回调被依次调用
```

### 39.5.2 批量更新的优化

减少 notify 频率：在 observe 回调中收集变化，批量通知。

```rust
struct BatchView {
    changes: Vec<Change>,
    _observer: gpui::Subscription,
}

impl BatchView {
    fn new(cx: &mut Context<Self>) -> Self {
        let _observer = cx.observe(&data_source, |this, source, cx| {
            // 收集变化而不是立即通知
            let new_changes = source.read(cx).pending_changes();
            this.changes.extend(new_changes);

            // 批量通知：只触发一次重绘
            cx.notify();
        });

        Self { changes: Vec::new(), _observer }
    }
}
```

observe 模式的核心规则：
- `cx.observe(&entity, ...)` 观察外部 Entity
- `cx.observe_self(...)` 观察自己
- `cx.observe_global::<T>(...)` 观察全局状态
- 回调中手动调用 `cx.notify()` 触发重绘
- `Subscription` 的 drop 自动取消订阅
