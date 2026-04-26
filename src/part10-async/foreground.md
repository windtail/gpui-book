# 第 36 章：前台执行器

## 36.1 `ForegroundExecutor`

### 36.1.1 主线程异步

GPUI 的 `ForegroundExecutor` 运行在主线程上。所有 UI 操作只能在主线程执行，前台执行器保证了这一点。

```rust
use gpui::{ForegroundExecutor, Task};

// 从 App 获取前台执行器
let executor: &ForegroundExecutor = cx.foreground_executor();

// 在主线程上运行 future
executor.spawn(async {
    // 这里可以直接访问 UI 相关资源
    // 不需要 Send 约束
}).detach();
```

### 36.1.2 为什么需要前台执行器

`ForegroundExecutor` 内部的 future 不需要 `Send`。这允许捕获 `Rc`、`&mut Window` 等不可跨线程的类型。

```rust
use std::rc::Rc;

// 编译通过 —— Rc 不需要 Send
let shared_data: Rc<String> = Rc::new("hello".into());
cx.foreground_executor().spawn(async move {
    println!("{}", *shared_data);
}).detach();

// 编译失败 —— BackgroundExecutor 需要 Send
cx.background_executor().spawn(async move {
    println!("{}", *shared_data); // error: `Rc<String>` cannot be sent between threads
}).detach();
```

GPUI 在 `ForegroundExecutor` 上用 `PhantomData<Rc<()>>` 标记 `!Send`，编译期阻止错误使用。

## 36.2 `cx.spawn()`

### 36.2.1 从 Entity 启动异步任务

`cx.spawn()` 是 View/Entity 上下文上最常用的异步启动方式。它接收一个闭包，闭包的第一个参数是 `WeakEntity<T>`，第二个参数是 `&mut AsyncApp`。

```rust
struct FileLoader {
    content: Option<String>,
    loading: bool,
}

impl FileLoader {
    fn load_file(&mut self, path: String, cx: &mut Context<Self>) {
        self.loading = true;
        cx.notify();

        // spawn 持有 WeakEntity，Entity 被销毁时任务自动取消
        cx.spawn(|this, mut cx| async move {
            // 在后台读取文件
            let content = cx.background_spawn(async move {
                std::fs::read_to_string(&path).unwrap_or_default()
            }).await;

            // 回到前台更新 UI
            this.update(&mut cx, |this, cx| {
                this.content = Some(content);
                this.loading = false;
                cx.notify();
            }).ok();
        }).detach();
    }
}
```

### 36.2.2 `WeakEntity` 检查防止悬垂

`this` 是 `WeakEntity<Self>` 类型。调用 `this.update(&mut cx, ...)` 时，如果 entity 已被 drop，返回 `Err`。

```rust
cx.spawn(|this, mut cx| async move {
    // 耗时操作
    cx.background_executor().timer(Duration::from_secs(5)).await;

    // entity 可能已被销毁
    match this.update(&mut cx, |this, cx| {
        this.status = "done".into();
        cx.notify();
    }) {
        Ok(()) => println!("更新成功"),
        Err(_) => println!("entity 已被销毁，跳过更新"),
    }
}).detach();
```

### 36.2.3 任务返回值

`spawn` 返回 `Task<R>`，可以 `.await` 获取结果。

```rust
let task: Task<String> = cx.spawn(|this, mut cx| async move {
    let result = some_async_work().await;
    result
});

// 在另一个异步上下文中等待
let value = task.await;
```

## 36.3 `Task<T>`

### 36.3.1 GPUI 的任务类型

`Task<T>` 是 GPUI 封装的异步任务类型，实现了 `Future<Output = T>`。

```rust
use gpui::Task;

// 创建一个立即可用的 Task
let ready_task = Task::ready(42);

// 检查任务是否已完成
let task = cx.spawn(|_this, _cx| async { compute() });
if task.is_ready() {
    // 已经完成
}
```

### 36.3.2 `Task::detach()` 分离任务

默认情况下，丢弃 `Task` 会取消它。调用 `detach()` 让任务继续运行到完成，但无法再获取返回值。

```rust
// 不 detach —— drop task 时取消
let task = cx.spawn(|this, mut cx| async move {
    do_work().await;
    this.update(&mut cx, |this, cx| this.done = true).ok();
});
drop(task); // 任务被取消

// detach —— 任务继续运行
cx.spawn(|this, mut cx| async move {
    do_work().await;
    this.update(&mut cx, |this, cx| this.done = true).ok();
}).detach(); // 任务在后台继续
```

### 36.3.3 共享任务

多个消费者需要等待同一个任务的结果时，使用 `Task::shared()`。

```rust
// 注意：当前版本的 GPUI 中，shared 方法位于 scheduler::Task 层
// 典型使用模式是将同一个任务存入多个位置
```

## 36.4 View 生命周期与任务

### 36.4.1 View 销毁时任务自动取消

通过 `cx.spawn()` 启动的任务绑定到 entity 的弱引用。Entity 被 drop 后，`this.update()` 返回 `Err`，任务逻辑应优雅退出。

```rust
struct SearchView {
    results: Vec<String>,
}

impl SearchView {
    fn search(&mut self, query: String, cx: &mut Context<Self>) {
        cx.spawn(|this, mut cx| async move {
            let results = fetch_results(&query).await;

            // View 可能已被切换或关闭
            if this.update(&mut cx, |this, cx| {
                this.results = results;
                cx.notify();
            }).is_err() {
                return; // View 已销毁，直接返回
            }
        }).detach();
    }
}
```

### 36.4.2 任务中的 `cx.notify()`

任务完成后必须调用 `cx.notify()` 才能触发重新渲染。

```rust
this.update(&mut cx, |this, cx| {
    this.data = new_data;
    cx.notify(); // 不写这行，UI 不会更新
}).ok();
```

### 36.4.3 任务冲突与取消策略

同一操作不应同时运行多个实例。常见模式是在启动新任务前取消旧任务，或将任务 handle 存入 struct 字段。

```rust
struct AutoComplete {
    current_task: Option<Task<Vec<String>>>,
    query: String,
}

impl AutoComplete {
    fn update_query(&mut self, query: String, cx: &mut Context<Self>) {
        self.query = query;
        cx.notify();

        // 取消之前的搜索
        self.current_task.take();

        let query = self.query.clone();
        let task = cx.spawn(|this, mut cx| async move {
            let results = search(&query).await;
            this.update(&mut cx, |this, cx| {
                this.results = results;
                cx.notify();
            }).ok();
        });
        self.current_task = Some(task);
    }
}
```

## 36.5 实战：异步加载数据并更新 UI

完整的异步数据加载模式：

```rust
use gpui::{div, Context, Entity, Render, Styled, Task, prelude::*};
use std::time::Duration;

struct UserProfile {
    name: String,
    avatar_url: String,
}

struct UserView {
    user: Option<UserProfile>,
    loading: bool,
    error: Option<String>,
}

impl UserView {
    fn fetch_user(&mut self, user_id: u64, cx: &mut Context<Self>) {
        self.loading = true;
        self.error = None;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            // 后台网络请求
            let result = cx.background_spawn(async move {
                // 模拟网络延迟
                cx.background_executor().timer(Duration::from_millis(500)).await;
                if user_id == 0 {
                    Err("用户不存在".to_string())
                } else {
                    Ok(UserProfile {
                        name: format!("User {}", user_id),
                        avatar_url: format!("/avatar/{}", user_id),
                    })
                }
            }).await;

            // 前台更新
            match result {
                Ok(profile) => {
                    this.update(&mut cx, |this, cx| {
                        this.user = Some(profile);
                        this.loading = false;
                        cx.notify();
                    }).ok();
                }
                Err(e) => {
                    this.update(&mut cx, |this, cx| {
                        this.error = Some(e);
                        this.loading = false;
                        cx.notify();
                    }).ok();
                }
            }
        }).detach();
    }
}

impl Render for UserView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        if self.loading {
            return div().child("Loading...");
        }
        if let Some(ref e) = self.error {
            return div().child(format!("Error: {}", e));
        }
        if let Some(ref user) = self.user {
            return div()
                .flex()
                .gap_2()
                .child(format!("Name: {}", user.name))
                .child(format!("Avatar: {}", user.avatar_url));
        }
        div().child("No data")
    }
}
```

关键点：
- `spawn` 使用 `WeakEntity`，View 销毁时安全退出
- 后台操作用 `background_spawn`，不阻塞前台执行器
- 每个分支都调用 `cx.notify()` 触发重绘
- loading/error/data 三种状态在 render 中分别处理
