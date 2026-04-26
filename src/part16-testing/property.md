# 第 60 章：属性测试

GPUI 集成 `proptest` 库，提供属性测试支持。

## 60.1 #[gpui::property_test]

`#[gpui::property_test]` 宏结合 proptest 的随机输入生成与 GPUI 的确定性调度：

```rust
use gpui::*;

#[gpui::property_test]
async fn test_counter_never_negative(
    cx: &mut TestAppContext,
    seed: u64,
    initial_value in prop::num::i32::ANY,
) {
    let counter = cx.new(|cx| Counter { count: initial_value });

    // 无论初始值如何，某些不变量应该保持
    cx.read(|cx| {
        // 测试逻辑
    });
}
```

### 与普通 #[gpui::test] 的区别

| 特性 | `#[gpui::test]` | `#[gpui::property_test]` |
|------|----------------|------------------------|
| 输入 | 固定或手动变化 | proptest 自动生成 |
| 运行次数 | 1 次或指定次数 | 默认 100+ 次 |
| 缩小 | 无 | 自动缩小到最小失败用例 |
| Seed 控制 | `SEED` 环境变量 | `SEED` 变量统一控制 |

## 60.2 Proptest 集成

### seed_strategy

GPUI 提供 `seed_strategy()` 桥接 `SEED` 环境变量到 proptest：

```rust
pub fn seed_strategy() -> impl Strategy<Value = u64> {
    match std::env::var("SEED") {
        Ok(val) => Just(val.parse().unwrap()).boxed(),  // 固定值
        Err(_) => any::<u64>().no_shrink().boxed(),     // 随机值
    }
}
```

当 `SEED` 设置时，不 shrink（缩减），因为所有 scheduler seed 等价。

### apply_seed_to_proptest_config

统一控制 scheduler seed 和 case 生成 seed：

```rust
pub fn apply_seed_to_proptest_config(
    mut config: proptest::test_runner::Config,
) -> proptest::test_runner::Config {
    let seed = env::var("SEED")
        .ok()
        .and_then(|val| val.parse::<u64>().ok())
        .unwrap_or(0);
    config.rng_seed = proptest::test_runner::RngSeed::Fixed(seed);
    config
}
```

`SEED` 环境变量同时控制：
1. GPUI scheduler 的调度顺序
2. proptest 的测试用例生成

### run_test_once

属性测试使用 `run_test_once` 运行单次测试：

```rust
pub fn run_test_once(seed: u64, test_fn: Box<dyn UnwindSafe + FnOnce(TestDispatcher)>) {
    let result = panic::catch_unwind(|| {
        let dispatcher = TestDispatcher::new(seed);
        let scheduler = dispatcher.scheduler().clone();
        test_fn(dispatcher);
        scheduler.end_test();
    });

    match result {
        Ok(_) => {}
        Err(e) => panic::resume_unwind(e),
    }
}
```

## 60.3 种子控制与重现

### 重现失败

```bash
# 使用失败时的 seed 重现
SEED=12345 cargo test test_counter_never_negative
```

同一个 seed 保证：
- proptest 生成完全相同的测试用例序列
- GPUI scheduler 以相同顺序执行任务

### 默认行为

不设置 `SEED` 时：
- `seed_strategy()` 返回 `any::<u64>().no_shrink()`
- `apply_seed_to_proptest_config` 使用 seed 0
- 每次运行产生不同的随机输入

### 确定性调试

```bash
# 固定 seed，完全可重现
SEED=42 cargo test my_property_test

# 查看测试输出中的 failing seed
# failing seed: 12345
# You can rerun from this seed by setting the environmental variable SEED to 12345
```

## 60.4 属性测试示例

### 列表操作不变量

```rust
#[gpui::property_test]
async fn test_list_operations_preserve_order(
    cx: &mut TestAppContext,
    seed: u64,
    items in prop::collection::vec(any::<String>(), 1..50),
) {
    let state = cx.new(|cx| ListState::new(
        items.len(),
        move |index, _, _| {
            div().child(items[index].clone())
        },
    ));

    // 无论如何操作，列表项的相对顺序应保持不变
}
```

### 文本布局属性

```rust
#[gpui::property_test]
async fn test_text_wrapping_preserves_words(
    cx: &mut TestAppContext,
    seed: u64,
    text in "[a-zA-Z ]{10..200}",  // 随机英文文本
) {
    // 换行不应该拆分单词
    // 每行应该是完整的单词序列
}
```

### 颜色系统不变量

```rust
#[gpui::property_test]
async fn test_color_conversion_round_trip(
    cx: &mut TestAppContext,
    seed: u64,
    h in 0.0..360.0f32,
    s in 0.0..100.0f32,
    l in 0.0..100.0f32,
    a in 0.0..1.0f32,
) {
    let color = hsla(h, s, l, a);
    let rgb = color.to_rgb();
    let back = Hsla::from_rgb(rgb);
    // 转换回原值（允许浮点误差）
}
```

## 60.5 属性测试策略

### 自定义策略

```rust
use proptest::prelude::*;

// 有效的 GPUI 测试输入
fn valid_text_input() -> impl Strategy<Value = String> {
    "[\\u{0020}-\\u{D7FF}\\u{E000}-\\u{FFFD}\\u{10000}-\\u{10FFFF}]{1..100}"
}

fn reasonable_font_size() -> impl Strategy<Value = f32> {
    8.0..200.0
}
```

### 组合策略

```rust
fn test_scenario() -> impl Strategy<Value = (String, f32, FontWeight)> {
    (
        valid_text_input(),
        reasonable_font_size(),
        prop_oneof![
            Just(FontWeight::NORMAL),
            Just(FontWeight::BOLD),
            Just(FontWeight::LIGHT),
        ],
    )
}
```
