# 第 63 章：性能分析

GPUI 内置 profiler 模块，追踪 task 执行时间，帮助定位性能瓶颈。

## 63.1 profiler 模块结构

### TaskTiming

记录单个 task 的时间信息：

```rust
pub struct TaskTiming {
    pub location: &'static panic::Location<'static>,  // 源码位置
    pub start: Instant,                                // 开始时间
    pub end: Option<Instant>,                          // 结束时间（None 表示仍在执行）
}
```

`location` 精确到文件 + 行号，可以直接跳转到耗时代码。

### ThreadTaskTimings

收集单个线程的所有 task 计时：

```rust
pub struct ThreadTaskTimings {
    pub thread_name: Option<String>,
    pub thread_id: ThreadId,
    pub timings: Vec<TaskTiming>,
    pub total_pushed: u64,  // 已推入的总条目数
}
```

### ProfilingCollector

跨线程收集性能数据：

```rust
pub struct ProfilingCollector {
    startup_time: Instant,
    cursors: HashMap<ThreadId, u64>,  // 每个线程的读取位置
}
```

## 63.2 启用/禁用

```rust
use gpui::profiler;

// 运行时启用
profiler::set_enabled(true);

// 禁用并清除数据
profiler::set_enabled(false);
```

建议开发时启用，发布时禁用。profiler 设计为运行时开关，不需要重新编译。

## 63.3 收集数据

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
            timing.end.unwrap().duration_since(timing.start).as_nanos()
        );
    }
}
```

### collect_unseen

避免重复读取已处理的数据：

```rust
let prev_cursor = self.cursors.get(&thread.thread_id).copied().unwrap_or(0);
let buffer_start = thread.total_pushed.saturating_sub(buffer_len);

if prev_cursor < buffer_start {
    // cursor 落后——部分条目已被环形缓冲区淘汰
    // 返回缓冲区中剩余的所有数据
    thread.timings.as_slice()
}
```

## 63.4 环形缓冲区

每个线程的 task 时间存储在固定大小的环形缓冲区中：

```rust
// 16MiB 的 TaskTiming 条目
const MAX_TASK_TIMINGS: usize = (16 * 1024 * 1024) / core::mem::size_of::<TaskTiming>();
```

超出容量时最早的条目被淘汰。这限制了 profiler 的内存开销。

### 线程局部存储

```rust
thread_local! {
    static THREAD_TIMINGS: RefCell<Vec<TaskTiming>> = RefCell::new(Vec::new());
}
```

每个线程独立记录，无锁写入。

## 63.5 序列化

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

序列化后的数据可以：
- 导入外部分析工具
- 上传到性能仪表板
- 保存为文件用于回归对比

## 63.6 帧时间分析

```rust
// 追踪每帧渲染时间
struct FrameTimer {
    frame_start: Instant,
}

impl FrameTimer {
    fn begin_frame(&mut self) {
        self.frame_start = Instant::now();
    }

    fn end_frame(&self) -> Duration {
        self.frame_start.elapsed()
    }
}

// 目标：60fps = 每帧 16.67ms
// 如果帧时间超过阈值，记录警告
if frame_time > Duration::from_millis(16) {
    log::warn!("Frame took {:?} (target: 16ms)", frame_time);
}
```

### GPUI_MANIFEST_DIR

构建时嵌入 manifest 目录信息，用于关联性能数据与源码版本：

```rust
// 编译时嵌入
const GPUI_MANIFEST_DIR: &str = env!("CARGO_MANIFEST_DIR");
```

结合 git hash 和构建时间戳，可以精确追溯性能回归对应的代码版本。

## 63.7 性能分析工作流

1. **启用 profiler**：`profiler::set_enabled(true)`
2. **执行操作**：触发需要分析的用户操作
3. **收集数据**：`collector.collect_unseen(timings)`
4. **分析热点**：按耗时排序，找到最慢的 task
5. **定位源码**：通过 `location.file` + `location.line` 跳转到代码
6. **优化后对比**：再次收集数据确认改进效果
7. **导出记录**：序列化保存，用于后续回归对比

### 注意事项

- profiler 本身有少量开销，不建议在生产环境长期启用
- 环形缓冲区大小（16MiB）限制了大量短时 task 的保留
- 多线程数据可能存在读取时的竞态（旧条目被淘汰）
- `collect_unseen` 的 cursor 落后时数据不完整
