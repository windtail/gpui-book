# 第 42 章：Subscription 生命周期

## 42.1 `Subscription` struct

### 42.1.1 RAII 模式

`Subscription` 是一个 RAII 句柄。持有它表示订阅处于活跃状态，drop 时自动取消订阅。

```rust
use gpui::Subscription;

struct MyView {
    data_source: Entity<DataSource>,
    _observe_sub: Subscription,    // 保存为字段
    _event_sub: Subscription,
}

impl MyView {
    fn new(data_source: Entity<DataSource>, cx: &mut Context<Self>) -> Self {
        let _observe_sub = cx.observe(&data_source, |this, source, cx| {
            this.data = source.read(cx).clone();
            cx.notify();
        });

        let _event_sub = cx.subscribe(&data_source, |this, source, event, cx| {
            match event {
                DataSourceEvent::Updated(data) => {
                    this.data = data.clone();
                    cx.notify();
                }
            }
        });

        Self {
            data_source,
            _observe_sub,
            _event_sub,
        }
    }
}
```

### 42.1.2 Drop 时自动取消

`Subscription` 的 `Drop` 实现调用 unsubscribe 回调：

```rust
// 内部实现（简化）
pub struct Subscription {
    unsubscribe: Option<Box<dyn FnOnce()>>,
}

impl Drop for Subscription {
    fn drop(&mut self) {
        if let Some(unsubscribe) = self.unsubscribe.take() {
            unsubscribe();
        }
    }
}
```

效果：当 `MyView` 被 drop 时，字段中的 `Subscription` 也被 drop，回调被移除。被观察的 Entity 不再向已销毁的 View 发送通知。

## 42.2 `Subscription::detach()`

### 42.2.1 手动解除关联

`detach()` 消费 `Subscription`，移除 unsubscribe 行为。订阅本身继续存在，直到被观察的 Entity 被 drop。

```rust
let sub = cx.observe(&entity, |this, entity, cx| {
    cx.notify();
});

// 调用 detach 后，sub 被消费
sub.detach();

// 订阅仍然存在：Entity update 仍会触发回调
// 但不再能通过 sub 取消它
```

### 42.2.2 延长生命周期

`detach()` 让订阅的存活不再依赖 `Subscription` 句柄：

```rust
struct LongLivedView {
    _sub: Option<Subscription>,
}

impl LongLivedView {
    fn start_observing(&mut self, entity: Entity<SomeEntity>, cx: &mut Context<Self>) {
        let sub = cx.observe(&entity, |this, entity, cx| {
            this.handle_update(entity, cx);
        });

        // 不保存 sub —— 直接 detach
        // 订阅会一直存在直到 entity 被销毁
        sub.detach();
    }
}
```

慎用 `detach()`。如果 View 被销毁但 Entity 仍在，回调中 `Entity::update(cx, ...)` 可能会 panic（因为 View 已不存在）。

## 42.3 `Subscription::join()`

### 42.3.1 组合多个订阅

`Subscription::join()` 将两个 `Subscription` 合并为一个：

```rust
let sub_a = cx.observe(&source_a, |this, source, cx| {
    this.update_a(source.read(cx));
});

let sub_b = cx.subscribe(&source_b, |this, source, event, cx| {
    this.handle_event(event);
});

// 合并为一个 Subscription
let combined = Subscription::join(sub_a, sub_b);

// drop combined 时，sub_a 和 sub_b 都被取消
```

### 42.3.2 统一生命周期管理

当 View 有多个订阅时，`join` 减少字段数量：

```rust
// 不用 join：三个字段
struct ViewA {
    _sub1: Subscription,
    _sub2: Subscription,
    _sub3: Subscription,
}

// 用 join：一个字段
struct ViewB {
    _sub: Subscription,
}

impl ViewB {
    fn new(cx: &mut Context<Self>) -> Self {
        let sub1 = cx.observe(&src1, ...);
        let sub2 = cx.subscribe(&src2, ...);
        let sub3 = cx.observe_global::<Config>(...);

        let _sub = Subscription::join(
            Subscription::join(sub1, sub2),
            sub3,
        );

        Self { _sub }
    }
}
```

## 42.4 内存考量

### 42.4.1 `SubscriberSet` 内部结构

GPUI 使用 `SubscriberSet<EmitterKey, Callback>` 管理订阅：

```rust
pub(crate) struct SubscriberSet<EmitterKey, Callback>(
    Rc<RefCell<SubscriberSetState<EmitterKey, Callback>>>,
);

struct SubscriberSetState<EmitterKey, Callback> {
    subscribers: BTreeMap<EmitterKey, Option<BTreeMap<usize, Subscriber<Callback>>>>,
    next_subscriber_id: usize,
}

struct Subscriber<Callback> {
    active: Rc<Cell<bool>>,    // 是否已激活
    dropped: Rc<Cell<bool>>,   // 是否已 drop
    callback: Callback,
}
```

`SubscriberSet` 内部结构要点：
- 以 `EmitterKey`（通常是 `EntityId`）分组
- 每个订阅有唯一 `subscriber_id`
- `active` 标记控制回调是否被调用
- `dropped` 标记处理取消期间的边界情况

### 42.4.2 泄漏检测

未保存 `Subscription` 的 observe/subscribe 不会泄漏内存，但会一直存在直到被观察的 Entity 被销毁：

```rust
// 不保存 Subscription
fn setup_listener(entity: Entity<Data>, cx: &mut App) {
    cx.observe(&entity, |entity, cx| {
        // 回调持有 entity 的强引用
        // 这个 observe 会一直存在直到 entity 被 drop
    });
}
```

对于 `subscribe`，如果不保存 `Subscription`，回调同样持续存在。这在单次临时监听时可能导致意外行为。

### 42.4.3 大量订阅的性能

每个 Entity update 都会遍历该 Entity 的所有订阅者：

```rust
// 场景：一个 DataSource 被 1000 个 View 观察
for _ in 0..1000 {
    cx.new(|cx| DataView::new(data_source, cx));
}

// 每次 data_source.update() 都会触发 1000 次回调
data_source.update(cx, |ds, cx| {
    ds.value += 1;
    // 1000 个 observe 回调被调用
});
```

优化策略：
- 减少 observe 数量：让中间 Entity 聚合状态
- 使用 subscribe 替代 observe：只在显式 emit 时触发
- 批量更新：一次 `update()` 只触发一次 observe 回调

Subscription 管理总结：
- `Subscription` 是 RAII 句柄，drop 时自动取消
- `Subscription::detach()` 让订阅持久化到 Entity 生命周期
- `Subscription::join(a, b)` 合并多个订阅为一个句柄
- 不保存 `Subscription` = 无法取消订阅
- 每个 `update()` 触发所有活跃的 observe 回调
