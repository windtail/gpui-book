# 第 57 章：#[gpui::test]

GPUI 提供一等公民的测试支持。`#[gpui::test]` 宏为需要 App 上下文的测试提供确定性调度。

## 57.1 基本用法

```rust
use gpui::*;

#[gpui::test]
async fn test_counter(cx: &TestAppContext) {
    let counter = cx.new(|cx| Counter { count: 0 });

    counter.update(cx, |this, cx| {
        this.count += 1;
    });

    cx.read(|cx| {
        assert_eq!(counter.read(cx).count, 1);
    });
}
```

### 测试函数签名

测试函数接收 `&TestAppContext` 参数。可以是同步或异步：

```rust
// 同步测试
#[gpui::test]
fn test_sync(cx: &TestAppContext) {
    // ...
}

// 异步测试
#[gpui::test]
async fn test_async(cx: &TestAppContext) {
    // ...
}
```

### 多上下文测试（协作 UI）

请求多个上下文来测试协作场景：

```rust
#[gpui::test]
async fn test_collaboration(cx_a: &TestAppContext, cx_b: &TestAppContext) {
    // 两个独立的 App 实例
    // 可以模拟两个用户的交互
}
```

## 57.2 TestDispatcher

`#[gpui::test]` 宏将测试函数包装到 `TestDispatcher` 中。它提供确定性的任务调度：

```rust
// 测试运行时的核心流程
let dispatcher = TestDispatcher::new(seed);
let scheduler = dispatcher.scheduler().clone();
test_fn(dispatcher, seed);
scheduler.end_test();
```

`TestDispatcher` 确保：
- 所有前台/后台任务按确定性顺序执行
- 时间可以被推进（`advance_clock`）
- 随机性通过 seed 控制

## 57.3 SEED 环境变量

每次测试运行都使用一个 seed。设置 `SEED` 环境变量重现特定运行：

```bash
# 使用固定 seed 重现失败
SEED=42 cargo test test_counter

# 测试输出会显示 failing seed
# seed = 12345
# failing seed: 12345
# You can rerun from this seed by setting the environmental variable SEED to 12345
```

### seed 策略

```rust
// test.rs 中
pub fn seed_strategy() -> impl Strategy<Value = u64> {
    match std::env::var("SEED") {
        Ok(val) => Just(val.parse().unwrap()).boxed(),  // 固定 seed
        Err(_) => any::<u64>().no_shrink().boxed(),     // 随机 seed
    }
}
```

`SEED` 设置时不 shrink（缩减），因为所有 scheduler seed 在复杂度上等价。

## 57.4 ITERATIONS 环境变量

控制测试运行次数：

```bash
# 运行 100 次，每次使用不同的 seed
ITERATIONS=100 cargo test test_counter

# 从特定 seed 开始运行 50 次
SEED=42 ITERATIONS=50 cargo test test_counter
```

### 种子计算逻辑

```rust
fn calculate_seeds(iterations: u64, explicit_seeds: &[u64]) -> (impl Iterator<Item = u64>, bool) {
    // 1. SEED 环境变量优先
    // 2. 如果 iterations=1 且无显式 seed，使用 seed 0
    // 3. 如果 iterations>1，使用 0..iterations
    // 4. 如果 SEED 已设置，使用 SEED..SEED+iterations
    // 5. 显式 seeds 作为补充（被 SEED 覆盖时忽略）
}
```

## 57.5 重试机制

测试失败时可以自动重试：

```rust
pub fn run_test(
    num_iterations: usize,
    explicit_seeds: &[u64],
    max_retries: usize,   // 最大重试次数
    test_fn: &mut dyn Fn(TestDispatcher, u64),
    on_fail_fn: Option<fn()>,  // 失败时的回调
) {
    for seed in seeds {
        let mut attempt = 0;
        loop {
            let result = panic::catch_unwind(|| { /* 运行测试 */ });
            match result {
                Ok(_) => break,
                Err(error) => {
                    if attempt < max_retries {
                        println!("attempt {} failed, retrying", attempt);
                        attempt += 1;
                        std::mem::forget(error);  // 防止二次 unwind
                    } else {
                        panic::resume_unwind(error);
                    }
                }
            }
        }
    }
}
```

重试时对错误调用 `std::mem::forget`，防止 panic payload 在 drop 时再次触发 unwind。

## 57.6 Observation 流助手

`observe()` 将 Entity 的变化转换为异步流：

```rust
use gpui::test::observe;

#[gpui::test]
async fn test_counter_emits_changes(cx: &mut TestAppContext) {
    let counter = cx.new(|cx| Counter { count: 0 });

    // 创建变化流
    let mut changes = observe(&counter, cx);

    // 触发变化
    counter.update(cx, |this, cx| {
        this.count += 1;
        cx.notify();
    });

    // 等待流中的下一个事件
    changes.next().await;

    cx.read(|cx| {
        assert_eq!(counter.read(cx).count, 1);
    });
}
```

`Observation<T>` 实现了 `futures::Stream`：

```rust
pub struct Observation<T> {
    rx: Pin<Box<async_channel::Receiver<T>>>,
    _subscription: Subscription,
}

impl<T: 'static> futures::Stream for Observation<T> {
    type Item = T;

    fn poll_next(
        mut self: Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Option<Self::Item>> {
        self.rx.poll_next_unpin(cx)
    }
}
```

Subscription 保留时流活跃，drop 时自动取消。

## 57.7 与 cargo test 集成

`#[gpui::test]` 的输出与其他 Rust 测试运行器兼容：

```bash
# 标准 cargo test
cargo test

# 使用 cargo-nextest
cargo nextest run

# 过滤特定测试
cargo test test_counter -- --nocapture
```

GPUI 测试是标准 Rust 测试，可以混合使用普通 `#[test]` 和 `#[gpui::test]`。
