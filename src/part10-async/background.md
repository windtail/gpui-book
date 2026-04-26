# 第 37 章：后台执行器

## 37.1 `BackgroundExecutor`

### 37.1.1 后台线程池

`BackgroundExecutor` 运行在后台线程池上，用于执行 CPU 密集或阻塞操作，避免阻塞主线程。

```rust
use gpui::{BackgroundExecutor, Task};

// 从 App 获取后台执行器
let executor: &BackgroundExecutor = cx.background_executor();

// 在线程池上运行 future
executor.spawn(async move {
    // CPU 密集型计算
    let result = heavy_computation();
    result
}).detach();
```

### 37.1.2 `cx.background_spawn()`

最直接的调用方式是通过上下文：

```rust
// 通过 App/Window/Entity 上下文
let task = cx.background_spawn(async move {
    let data = std::fs::read_to_string("config.json")?;
    let parsed: Config = serde_json::from_str(&data)?;
    Ok(parsed)
});

// 返回 Task<Result<Config, Error>>
```

`background_spawn` 要求 future 是 `Send + 'static`，返回值也必须是 `Send + 'static`。

## 37.2 线程安全

### 37.2.1 `Send + Sync` 要求

后台任务在独立线程上运行，所有被捕获的变量必须实现 `Send`。

```rust
use std::rc::Rc;

let rc_data = Rc::new(42);

// 编译失败 —— Rc 不是 Send
cx.background_spawn(async move {
    println!("{}", *rc_data); // error: `Rc<i32>` cannot be sent
}).detach();

// 改用 Arc
use std::sync::Arc;
let arc_data = Arc::new(42);
cx.background_spawn(async move {
    println!("{}", *arc_data); // 编译通过
}).detach();
```

### 37.2.2 不能直接操作 UI

后台任务中不能访问 UI 相关资源。需要通过 `Task` 返回值将结果传回前台。

```rust
// 错误：后台任务中不能访问 UI
cx.background_spawn(async move {
    cx.notify(); // 编译错误：UI 操作只能在前台
}).detach();

// 正确：返回值传回前台
let result = cx.background_spawn(async move {
    compute_data()
}).await;

// 结果回到前台后更新 UI
cx.update(|this, cx| {
    this.data = result;
    cx.notify();
});
```

### 37.2.3 数据传递模式

典型的前后台协作模式：

```rust
struct DataProcessor {
    processed: Option<Vec<String>>,
}

impl DataProcessor {
    fn process(&mut self, input: Vec<String>, cx: &mut Context<Self>) {
        cx.spawn(|this, mut cx| async move {
            // 后台处理
            let output = cx.background_spawn(async move {
                input.into_iter()
                    .filter(|s| !s.is_empty())
                    .map(|s| s.to_uppercase())
                    .collect::<Vec<_>>()
            }).await;

            // 前台更新
            this.update(&mut cx, |this, cx| {
                this.processed = Some(output);
                cx.notify();
            }).ok();
        }).detach();
    }
}
```

## 37.3 工具方法

### 37.3.1 `timer()` 定时

`timer()` 返回一个 `Task<()>`，在指定时间后完成。

```rust
use std::time::Duration;

// 延迟执行
cx.background_executor().spawn(async move {
    cx.background_executor().timer(Duration::from_secs(1)).await;
    println!("1 秒后执行");
}).detach();

// 周期性执行
fn schedule_periodic(cx: &BackgroundExecutor, interval: Duration, f: impl Fn()) {
    cx.spawn({
        let cx = cx.clone();
        async move {
            loop {
                cx.timer(interval).await;
                f();
            }
        }
    }).detach();
}
```

### 37.3.2 `block_on()` 阻塞等待

`block_on()` 阻塞当前线程直到 future 完成。谨慎使用，不要在前台线程调用。

```rust
// 在后台任务中同步等待另一个后台任务
let executor = cx.background_executor().clone();
let result = executor.block_on(async {
    some_async_computation().await
});
```

### 37.3.3 `scoped()` 作用域任务

`scoped()` 创建一组并行任务，等待全部完成后返回。

```rust
let files = vec!["a.txt", "b.txt", "c.txt"];
let results = std::sync::Arc::new(std::sync::Mutex::new(Vec::new()));

cx.background_executor().scoped(|scope| {
    for file in &files {
        let file = file.to_string();
        let results = results.clone();
        scope.spawn(async move {
            let content = std::fs::read_to_string(&file).unwrap_or_default();
            results.lock().unwrap().push((file, content));
        });
    }
}).await;
// 所有任务完成后才继续
```

## 37.4 `FallibleTask`

### 37.4.1 优雅取消

`Task<T>` 在被 drop 时直接取消。`FallibleTask<T>` 通过 `Task::fallible()` 创建，被取消时返回 `None` 而非 panic。

```rust
let task = cx.spawn(|this, mut cx| async move {
    let result = long_running_operation().await;
    result
});

// 转为 FallibleTask
let fallible = task.fallible();

// await 返回 Option<T>
match fallible.await {
    Some(value) => println!("任务完成: {:?}", value),
    None => println!("任务被取消"),
}
```

### 37.4.2 错误传播

对于 `Task<Result<T, E>>`，结合 `fallible()` 和 `Result` 处理：

```rust
let task = cx.spawn(|this, mut cx| async move {
    let result = risky_operation().await;
    result
});

let fallible = task.fallible();

match fallible.await {
    Some(Ok(value)) => println!("成功: {:?}", value),
    Some(Err(e)) => println!("失败: {}", e),
    None => println!("任务被取消"),
}
```

## 37.5 前台与后台的协作模式

常见的前后台协作架构：

```rust
struct ImageGallery {
    images: Vec<Thumbnail>,
    loading: bool,
}

impl ImageGallery {
    fn load_directory(&mut self, dir_path: String, cx: &mut Context<Self>) {
        self.loading = true;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            // 第一层：后台扫描文件
            let file_paths = cx.background_spawn(async move {
                std::fs::read_dir(&dir_path)
                    .into_iter()
                    .flatten()
                    .filter_map(|e| e.ok())
                    .filter(|e| {
                        e.path().extension()
                            .is_some_and(|ext| matches!(ext.to_str(), Some("png" | "jpg" | "webp")))
                    })
                    .map(|e| e.path())
                    .collect::<Vec<_>>()
            }).await;

            // 第二层：后台加载缩略图（并行）
            let thumbnails = cx.background_spawn({
                async move {
                    let mut thumbs = Vec::new();
                    for path in file_paths {
                        if let Some(thumb) = load_thumbnail(&path) {
                            thumbs.push(thumb);
                        }
                    }
                    thumbs
                }
            }).await;

            // 前台更新 UI
            this.update(&mut cx, |this, cx| {
                this.images = thumbnails;
                this.loading = false;
                cx.notify();
            }).ok();
        }).detach();
    }
}
```

协作要点：
- 前台负责 UI 状态管理和渲染触发
- 后台负责 I/O、计算、网络请求
- 通过 `spawn` 的 `WeakEntity` 检查保证 View 生命周期安全
- `cx.notify()` 只在 `this.update()` 闭包中调用
