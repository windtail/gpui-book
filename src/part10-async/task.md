# 第 38 章：Task 管理

## 38.1 Task 生命周期

### 38.1.1 创建 → 运行 → 完成/取消

`Task<T>` 的生命周期有三种状态：

```
创建 → 运行中 → 完成(返回值)
              → 取消(drop 时)
```

```rust
use gpui::Task;

// 创建：立即进入运行队列
let task: Task<String> = cx.spawn(|this, mut cx| async move {
    // 异步工作
    let result = do_work().await;

    // 更新状态并返回
    this.update(&mut cx, |this, cx| {
        this.done = true;
        cx.notify();
    }).ok();

    result
});

// 完成：通过 .await 获取返回值
let value = task.await;

// 取消：drop task 即取消
drop(task); // 异步工作被中断
```

### 38.1.2 Task drop 时的行为

`Task` 实现为 `Future` 的包装。drop 时，底层的异步任务被取消，不会继续执行。

```rust
{
    let task = cx.spawn(|_this, mut cx| async move {
        println!("开始");
        cx.background_executor().timer(Duration::from_secs(10)).await;
        println!("结束"); // 不会执行 —— task 在 10 秒前被 drop
    });
    // task 在此处被 drop
}
// 输出：无（任务可能还没来得及执行就被取消）
```

使用 `detach()` 可以让任务脱离 `Task` 的生命周期管理：

```rust
cx.spawn(|_this, mut cx| async move {
    println!("开始");
    cx.background_executor().timer(Duration::from_secs(1)).await;
    println!("结束"); // 会执行 —— 已 detach
}).detach();
```

## 38.2 多任务协调

### 38.2.1 任务串行化

同一类型的操作一次只运行一个实例，新操作取消旧操作。

```rust
struct SearchBox {
    current_task: Option<Task<Vec<SearchResult>>>,
    results: Vec<SearchResult>,
}

impl SearchBox {
    fn search(&mut self, query: String, cx: &mut Context<Self>) {
        // 取消之前的搜索
        self.current_task.take();

        let task = cx.spawn(|this, mut cx| async move {
            let results = search_api(&query).await;
            this.update(&mut cx, |this, cx| {
                this.results = results;
                cx.notify();
            }).ok();
        });

        self.current_task = Some(task);
    }
}
```

旧任务通过 `self.current_task.take()` 被 drop 取消，新任务接管字段。

### 38.2.2 任务并行化

多个独立任务同时运行：

```rust
struct Dashboard {
    user_task: Option<Task<UserInfo>>,
    stats_task: Option<Task<Stats>>,
    notifications_task: Option<Task<Vec<Notification>>>,
}

impl Dashboard {
    fn load_all(&mut self, cx: &mut Context<Self>) {
        self.user_task = Some(cx.spawn(|this, mut cx| async move {
            let user = fetch_user().await;
            this.update(&mut cx, |this, cx| {
                this.user_info = Some(user);
                cx.notify();
            }).ok();
        }));

        self.stats_task = Some(cx.spawn(|this, mut cx| async move {
            let stats = fetch_stats().await;
            this.update(&mut cx, |this, cx| {
                this.stats = Some(stats);
                cx.notify();
            }).ok();
        }));

        // 两个任务并行运行，各自完成后独立更新 UI
    }
}
```

### 38.2.3 任务队列

用 `Vec<Task>` 管理批量任务，等待全部完成：

```rust
struct BatchProcessor {
    tasks: Vec<Task<()>>,
}

impl BatchProcessor {
    fn process_batch(&mut self, items: Vec<Item>, cx: &mut Context<Self>) {
        for item in items {
            let task = cx.spawn(|this, mut cx| async move {
                process_item(item).await;
                this.update(&mut cx, |_, cx| cx.notify()).ok();
            });
            self.tasks.push(task);
        }
    }

    fn wait_all(&mut self, cx: &mut Context<Self>) {
        let tasks = std::mem::take(&mut self.tasks);
        cx.spawn(|this, mut cx| async move {
            for task in tasks {
                task.await; // 串行等待每个任务完成
            }
            this.update(&mut cx, |_, cx| cx.notify()).ok();
        }).detach();
    }
}
```

## 38.3 `Priority` 枚举

### 38.3.1 任务优先级

`Priority` 控制任务在线程池中的调度顺序：

```rust
use gpui::Priority;

// 后台执行器支持优先级
cx.background_executor().spawn_with_priority(Priority::Low, async move {
    // 低优先级：空闲时执行
    cleanup_temp_files().await
}).detach();

cx.background_executor().spawn_with_priority(Priority::High, async move {
    // 高优先级：立即执行
    save_critical_data().await
}).detach();
```

`Priority::RealtimeAudio` 用于音频处理的实时线程。

### 38.3.2 调度策略

前台任务的优先级目前被忽略（按顺序执行），后台任务的优先级影响线程池调度顺序。

```rust
// 前台：优先级无效，按入队顺序执行
cx.foreground_executor().spawn_with_priority(Priority::High, async {
    // 和低优先级任务一样排队
}).detach();

// 后台：高优先级先于低优先级执行
cx.background_executor().spawn_with_priority(Priority::High, async {
    first()
}).detach();
```

## 38.4 错误处理

### 38.4.1 `Result` 在 Task 中的传播

`Task<Result<T, E>>` 是常见的错误承载方式：

```rust
fn load_config(cx: &mut Context<App>) -> Task<Result<Config, anyhow::Error>> {
    cx.spawn(|this, mut cx| async move {
        let content = cx.background_spawn(async {
            std::fs::read_to_string("config.toml")
        }).await?;

        let config: Config = toml::from_str(&content)?;
        Ok(config)
    })
}
```

### 38.4.2 UI 中的错误展示

将错误映射到 UI 状态：

```rust
struct DataPanel {
    data: Option<Data>,
    error: Option<String>,
    loading: bool,
}

impl DataPanel {
    fn load(&mut self, cx: &mut Context<Self>) {
        self.loading = true;
        self.error = None;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            match fetch_data().await {
                Ok(data) => {
                    this.update(&mut cx, |this, cx| {
                        this.data = Some(data);
                        this.loading = false;
                        this.error = None;
                        cx.notify();
                    }).ok();
                }
                Err(e) => {
                    this.update(&mut cx, |this, cx| {
                        this.data = None;
                        this.loading = false;
                        this.error = Some(e.to_string());
                        cx.notify();
                    }).ok();
                }
            }
        }).detach();
    }
}
```

## 38.5 实战：可取消的搜索请求

自动补全场景中，快速输入会触发多个请求，只保留最后一个的结果：

```rust
use gpui::{div, Context, Render, Styled, Task, prelude::*};
use std::time::Duration;

struct AutoComplete {
    query: String,
    suggestions: Vec<String>,
    loading: bool,
    current_request: Option<Task<()>>,
    request_id: u64,
}

impl AutoComplete {
    fn on_input(&mut self, new_query: String, cx: &mut Context<Self>) {
        self.query = new_query.clone();
        self.loading = !new_query.is_empty();

        // 取消上一个请求
        self.current_request.take();

        if new_query.is_empty() {
            self.suggestions.clear();
            cx.notify();
            return;
        }

        let request_id = self.request_id;
        self.request_id += 1;

        let task = cx.spawn(move |this, mut cx| async move {
            // 延迟 300ms 防抖
            cx.background_executor().timer(Duration::from_millis(300)).await;

            let results = search_suggestions(&new_query).await;

            this.update(&mut cx, |this, cx| {
                // 只在查询未被更新时才应用结果
                if this.query == new_query {
                    this.suggestions = results;
                    this.loading = false;
                }
                cx.notify();
            }).ok();
        });

        self.current_request = Some(task);
    }
}

impl Render for AutoComplete {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .gap_1()
            .child(div().child(format!("Query: {}", self.query)))
            .when(self.loading, |this| {
                this.child(div().child("Searching..."))
            })
            .children(self.suggestions.iter().map(|s| {
                div().child(s.clone()).p_2()
            }))
    }
}
```

关键设计：
- `current_request.take()` 取消旧任务
- 查询字符串比对防止过期结果覆盖新结果
- 防抖延迟减少无效请求
- `FallibleTask` 可用于更优雅地处理取消
