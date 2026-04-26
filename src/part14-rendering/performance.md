# 第 53 章：性能优化

GPUI 的性能优化覆盖从视图层到渲染管线的多个层级。

## 53.1 视图缓存策略

### AnyView::cached()

`cached()` 将整个视图的渲染结果缓存为一个 sprite：

```rust
use gpui::AnyView;

// 缓存一个复杂视图
let cached_view = AnyView::new(complex_view).cached(window);

// 在 render 中使用
div().child(cached_view)
```

缓存失效条件：
- 被缓存的视图调用 `cx.notify()`
- 窗口缩放因子变化
- 缓存被显式清除

### 何时使用缓存

```rust
// 适合缓存：
// 1. 静态内容，很少变化
// 2. 渲染代价高（大量子元素、复杂路径）
// 3. 在滚动列表等场景中被频繁重绘

// 不适合缓存：
// 1. 每帧都在变化的内容（缓存无效化开销更大）
// 2. 简单的内容（cache sprite 的开销高于直接渲染）
// 3. 大量不同实例（每个都占一个 sprite，Atlas 压力大）
```

示例——缓存列表中的复杂行：

```rust
struct ListItem {
    data: String,
    // ...
}

impl Render for ListItem {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let content = div()
            .flex()
            .gap_2()
            .child(icon(self.icon()))
            .child(self.label())
            .child(self.badge());

        // 这一行内容复杂，缓存起来
        AnyView::new(content).cached(window)
    }
}
```

## 53.2 最小化重渲染

### 精确的 cx.notify()

`cx.notify()` 只标记当前视图需要重新渲染：

```rust
struct Counter {
    count: u32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(format!("Count: {}", self.count))
            .child(
                button("Increment").on_click(cx.listener(|this, _, _, cx| {
                    this.count += 1;
                    cx.notify();  // 只通知当前视图
                })),
            )
    }
}
```

不要过度调用 `notify()`。只在状态真正改变时调用。

### observe 的粒度控制

`cx.observe()` 只在被观察的 Entity 被 `notify()` 时触发回调：

```rust
// 粗粒度：整个 Entity 变化就重绘
cx.observe(&model, |_, _, cx| cx.notify()).detach();

// 细粒度：只关心特定字段
cx.observe(&model, |this, model, cx| {
    if this.last_count != model.count {
        this.last_count = model.count;
        cx.notify();
    }
}).detach();
```

### cx.observe_global()

观察全局状态变化：

```rust
cx.observe_global::<ThemeSettings>(|this, cx| {
    // 主题切换时重绘
    cx.notify();
}).detach();
```

## 53.3 Window::refresh() 控制

`refresh()` 触发整个窗口重新渲染：

```rust
// 在 Window 中
window.refresh();  // 强制下一帧重新渲染所有内容
```

与 `cx.notify()` 的区别：

| 方法 | 范围 | 使用场景 |
|------|------|----------|
| `cx.notify()` | 当前 View | 视图自身状态变化 |
| `window.refresh()` | 整个窗口 | 全局性变化（主题、缩放因子） |

```rust
// 主题切换时刷新整个窗口
fn theme_changed(&mut self, cx: &mut Context<Self>) {
    // 重新计算所有颜色
    self.recompute_colors();
    // 通知所有视图
    cx.refresh();  // 触发全局刷新
}
```

## 53.4 request_animation_frame()

安排回调在下一帧执行：

```rust
// 在 View 中
cx.request_animation_frame();

// 更精细的控制：带闭包的动画帧
window.request_animation_frame(cx, |window, cx| {
    // 在下一帧渲染前执行
    // 可以做动画插值
});
```

与 `cx.notify()` 的区别：
- `request_animation_frame()` 只安排一次渲染
- `cx.notify()` 标记视图 dirty，可能在当前帧或下一帧处理

动画循环示例：

```rust
struct Spinner {
    rotation: f32,
    last_frame: Option<Instant>,
}

impl Render for Spinner {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 更新旋转
        let now = Instant::now();
        if let Some(last) = self.last_frame {
            let delta = now.duration_since(last).as_secs_f32();
            self.rotation += delta * 180.0; // 180 度/秒
        }
        self.last_frame = Some(now);

        // 请求下一帧
        cx.request_animation_frame();

        div()
            .size_6()
            .rotate(deg(self.rotation))
            .child(div().bg(gpui::blue()).size_full().rounded_full())
    }
}
```

## 53.5 profiler 模块

GPUI 内置性能分析器，追踪 task 执行时间。

### profiler 模块结构

```rust
// profiler.rs
pub struct TaskTiming {
    pub location: &'static core::panic::Location<'static>,
    pub start: Instant,
    pub end: Option<Instant>,
}

pub struct ThreadTaskTimings {
    pub thread_name: Option<String>,
    pub thread_id: ThreadId,
    pub timings: Vec<TaskTiming>,
    pub total_pushed: u64,
}
```

### 启用/禁用

```rust
use gpui::profiler;

// 运行时启用
profiler::set_enabled(true);

// 禁用并清除数据
profiler::set_enabled(false);
```

### ProfilingCollector

收集所有线程的 task 时间：

```rust
use gpui::profiler::{ProfilingCollector, ThreadTaskTimings};

let collector = ProfilingCollector::new(startup_time);

// 获取所有线程的耗时
let all_timings: Vec<ThreadTaskTimings> = ThreadTaskTimings::convert(
    &GLOBAL_THREAD_TIMINGS.lock(),
);

// 只获取自上次调用以来的新数据
let deltas = collector.collect_unseen(all_timings);

for delta in deltas {
    println!("Thread: {:?}", delta.thread_name);
    for timing in &delta.new_timings {
        println!(
            "  {}:{} - {}ns",
            timing.location.file,
            timing.location.line,
            timing.duration
        );
    }
}
```

### 环形缓冲区

每个线程的 task 时间存储在固定大小的环形缓冲区中：

```rust
// 16MiB 的 TaskTiming 条目
const MAX_TASK_TIMINGS: usize = (16 * 1024 * 1024) / core::mem::size_of::<TaskTiming>();
```

超出容量时最早的条目被淘汰。`ProfilingCollector` 用 cursor 追踪已读取位置：

```rust
let prev_cursor = self.cursors.get(&thread.thread_id).copied().unwrap_or(0);
let buffer_start = thread.total_pushed.saturating_sub(buffer_len);

if prev_cursor < buffer_start {
    // cursor 落后——部分条目已丢失
    // 返回缓冲区中剩余的所有数据
    thread.timings.as_slice()
}
```

### 序列化

性能数据可以序列化后导出：

```rust
use gpui::profiler::{SerializedThreadTaskTimings, SerializedTaskTiming};

let anchor = collector.startup_time();
let serialized: Vec<SerializedThreadTaskTimings> = all_timings
    .into_iter()
    .map(|t| SerializedThreadTaskTimings::convert(anchor, t))
    .collect();

// 导出为 JSON
let json = serde_json::to_string(&serialized)?;
```

## 53.6 性能 checklist

### 渲染优化
- [ ] 对静态复杂内容使用 `AnyView::cached()`
- [ ] 对列表使用 `ListState` / `uniform_list` 而非直接渲染所有子元素
- [ ] 避免过深的 div 嵌套（每个 div 都是一个元素调用）
- [ ] 减少 `opacity` 使用（需要额外合成层）

### 状态更新
- [ ] 精确使用 `cx.notify()`，只标记真正变化的视图
- [ ] 批量更新后只调用一次 `notify()`
- [ ] `observe` 回调中加条件判断再 `notify()`

### 文本渲染
- [ ] 复用 `WindowTextSystem` 的布局缓存
- [ ] 对长文本使用 `compute_glyph_raster_data` 预计算
- [ ] 合理设置 `line_clamp` 限制包裹行数

### 动画
- [ ] 用 `request_animation_frame()` 驱动动画
- [ ] 避免在 `render` 中做耗时计算
- [ ] 动画中只更新变化的部分

### 异步
- [ ] 用 `cx.spawn()` 而非 `spawn()` 来保证 View 生命周期
- [ ] 耗时操作放 `background_spawn()`
- [ ] 用 `FallibleTask` 优雅处理取消

###  profiling
- [ ] 开发时启用 `profiler::set_enabled(true)`
- [ ] 定期收集 `ThreadTaskTimings` 分析瓶颈
- [ ] 用 `SEED` 环境变量重现随机测试失败
