# 第9章：View 生命周期

## 9.1 视图创建

### 9.1.1 `cx.new()` 创建流程

```rust
use gpui::{App, Context, Window, div, IntoElement};

struct MyView {
    name: String,
}

impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(format!("Hello, {}!", self.name))
    }
}

fn main() {
    let app = gpui::Application::new();
    app.run(|cx: &mut App| {
        let view = cx.new(|cx| MyView {
            name: "GPUI".to_string(),
        });
        // view: Entity<MyView>
    });
}
```

`cx.new()` 的执行流程：
1. 在 `EntityMap` 中分配一个新的 `EntityId`
2. 调用闭包初始化数据
3. 注册 `SubscriberSet`（用于 observe/subscribe）
4. 返回 `Entity<T>` 句柄

### 9.1.2 `Entity<V>` 与 `AnyView` 的关系

```rust
// Entity<MyView> 是类型安全的
let typed: Entity<MyView> = cx.new(|cx| MyView { ... });

// AnyView 是类型擦除的视图句柄
let any: gpui::AnyView = typed.into();

// AnyView 可以用于窗口根视图
cx.open_window(WindowOptions::default(), |window, cx| {
    cx.new(|cx| MyView { ... }).into()
});
```

`AnyView` 用于存储不同类型的 View，典型场景是路由和页面切换。

### 9.1.3 窗口根视图设置

```rust
use gpui::{Application, App, Window, WindowOptions, AnyView, Context, IntoElement};

struct HomePage { title: String }
impl Render for HomePage {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(self.title.as_str())
    }
}

struct SettingsPage { theme: String }
impl Render for SettingsPage {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(format!("Theme: {}", self.theme))
    }
}

fn main() {
    let app = Application::new();
    app.run(|cx: &mut App| {
        cx.open_window(WindowOptions::default(), |window, cx| {
            // 根视图必须返回 AnyView
            cx.new(|cx| HomePage { title: "Home".into() }).into()
        });
    });
}
```

## 9.2 渲染触发

### 9.2.1 `cx.notify()` 手动通知

```rust
struct Counter { count: i32 }

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(format!("Count: {}", self.count))
            .child(
                button().label("Inc").on_click(cx.listener(|this, _, _, cx| {
                    this.count += 1;
                    cx.notify();  // 告诉 GPUI 此 View 需要重新渲染
                }))
            )
    }
}
```

`cx.notify()` 将当前 View 标记为 dirty。在当前帧的所有事件回调执行完毕后，GPUI 收集所有 dirty View 并重新调用它们的 `render()` 方法。

### 9.2.2 `cx.observe()` 自动通知

```rust
struct SharedCounter {
    value: i32,
}

struct Display {
    shared: Entity<SharedCounter>,
    _subscription: gpui::Subscription,
}

impl Display {
    fn new(shared: Entity<SharedCounter>, cx: &mut Context<Self>) -> Self {
        Self {
            shared: shared.clone(),
            _subscription: cx.observe(&shared, |this, _, cx| {
                // 当 shared 被 update 时，自动触发此回调
                cx.notify();  // 自动重新渲染
            }),
        }
    }
}

impl Render for Display {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let value = self.shared.read(cx).value;
        div().child(format!("Shared value: {}", value))
    }
}
```

`cx.observe()` 建立观察关系：当被观察的 Entity 被 `update()` 时，观察者自动收到通知。

### 9.2.3 帧调度与渲染时机

```
事件到达 → 执行回调 → 回调中修改状态 → 回调中 cx.notify()
                                               ↓
                              当前帧回调全部执行完毕
                                               ↓
                              收集所有 dirty View
                                               ↓
                              对每个 dirty View 调用 render()
                                               ↓
                              Element 树 → 布局 → 绘制 → 提交到 GPU
                                               ↓
                              等待下一个事件
```

渲染是批量的：一帧内多次 `cx.notify()` 只触发一次重新渲染。

```rust
struct BatchUpdate {
    a: i32,
    b: i32,
    c: i32,
}

impl Render for BatchUpdate {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(format!("a={} b={} c={}", self.a, self.b, self.c))
    }
}

fn update_all(this: &mut BatchUpdate, cx: &mut Context<BatchUpdate>) {
    this.a += 1;
    cx.notify();  // 第1次通知
    this.b += 1;
    cx.notify();  // 第2次通知
    this.c += 1;
    cx.notify();  // 第3次通知
    // 最终只触发一次 render() 调用
}
```

## 9.3 视图缓存

### 9.3.1 `AnyView::cached()` 的作用

```rust
use gpui::AnyView;

struct ExpensiveView {
    items: Vec<String>,
}

impl Render for ExpensiveView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().children(self.items.iter().map(|item| {
            div().child(item.as_str())
        }))
    }
}

// 在父 View 中使用缓存
struct Parent {
    child: AnyView,
}

impl Render for Parent {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(self.child.clone())  // 如果 child 没变，不会重新渲染
    }
}
```

GPUI 会追踪 View 的 EntityId。如果同一个 Entity 在连续两帧中被渲染，且没有被 `notify()` 标记为 dirty，GPUI 会复用上一帧的 Element 树。

### 9.3.2 缓存失效的条件

以下情况会导致缓存失效：

1. View 被 `cx.notify()` 标记为 dirty
2. View 被 `cx.observe()` 的观察者触发回调
3. 窗口尺寸变化（布局约束改变）
4. 主题/外观变化

```rust
struct CachedChild {
    data: String,
}

impl Render for CachedChild {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(self.data.as_str())
    }
}

struct Parent {
    child: Entity<CachedChild>,
}

impl Parent {
    fn update_child(&mut self, cx: &mut Context<Self>) {
        self.child.update(cx, |child, cx| {
            child.data = "New data".into();
            cx.notify();  // 清除子 View 缓存
        });
    }
}
```

### 9.3.3 何时使用缓存，何时不用

**应该缓存**：
- 大型列表（成百上千项）
- 复杂的布局计算结果
- 不频繁变化的 UI 片段

**不应缓存**：
- 频繁变化的动画
- 实时更新的进度条
- 依赖于父状态的轻量级组件

## 9.4 视图无效化

### 9.4.1 `refresh()` 强制刷新

```rust
use gpui::AnyView;

struct AppView {
    current_page: AnyView,
}

impl AppView {
    fn force_refresh(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        // 刷新整个窗口
        window.refresh();
    }

    fn navigate_to(&mut self, page: AnyView, window: &mut Window, cx: &mut Context<Self>) {
        self.current_page = page;
        window.refresh();  // 强制下一帧重新渲染
    }
}
```

### 9.4.2 `request_animation_frame()` 安排下一帧

```rust
struct AnimationView {
    progress: f32,
    animating: bool,
}

impl Render for AnimationView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        if self.animating {
            // 请求下一帧，形成动画循环
            window.request_animation_frame();
        }

        div()
            .w_full()
            .bg_blue_500()
            .h(px(20.0))
            .w(re(self.progress))  // 宽度随 progress 变化
    }
}

impl AnimationView {
    fn start_animation(&mut self, cx: &mut Context<Self>) {
        self.animating = true;
        self.progress = 0.0;
        cx.notify();
    }

    fn tick(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        self.progress += 0.01;
        if self.progress >= 1.0 {
            self.animating = false;
            self.progress = 1.0;
        }
        window.request_animation_frame();  // 继续请求下一帧
        cx.notify();
    }
}
```

### 9.4.3 局部更新与全量刷新

```rust
struct SplitView {
    left: Entity<LeftPanel>,
    right: Entity<RightPanel>,
}

impl SplitView {
    // 只更新左侧面板
    fn update_left(&mut self, cx: &mut Context<Self>) {
        self.left.update(cx, |left, cx| {
            left.refresh_data();
            cx.notify();  // 只标记 left 为 dirty
        });
        // right 不会重新渲染
    }

    // 全量刷新（不推荐）
    fn refresh_all(&mut self, cx: &mut Context<Self>) {
        cx.notify();  // 标记 SplitView 本身为 dirty
    }
}
```

经验法则：**精确通知需要更新的 View，而不是刷新整个父级**。

## 9.5 视图导航

### 9.5.1 `replace_root_view()` 切换页面

```rust
struct Router {
    history: Vec<AnyView>,
}

impl Router {
    fn navigate(&mut self, new_page: AnyView, window: &mut Window, cx: &mut Context<Self>) {
        self.history.push(window.root_view(cx).unwrap());
        window.replace_root_view(new_page);
    }

    fn go_back(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        if let Some(previous) = self.history.pop() {
            window.replace_root_view(previous);
        }
    }
}
```

### 9.5.2 路由模式实现

```rust
enum Route {
    Home,
    Settings,
    Profile(Entity<User>),
}

struct RouterView {
    current_route: Route,
}

impl RouterView {
    fn to_page(&self, cx: &mut Context<Self>) -> AnyView {
        match &self.current_route {
            Route::Home => cx.new(|cx| HomePage { ... }).into(),
            Route::Settings => cx.new(|cx| SettingsPage { ... }).into(),
            Route::Profile(user) => cx.new(|cx| ProfilePage { user: user.clone() }).into(),
        }
    }
}

impl Render for RouterView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(
                div().flex().gap_2()
                    .child(button().label("Home").on_click(cx.listener(|this, _, window, cx| {
                        this.current_route = Route::Home;
                        window.replace_root_view(this.to_page(cx));
                    })))
                    .child(button().label("Settings").on_click(cx.listener(|this, _, window, cx| {
                        this.current_route = Route::Settings;
                        window.replace_root_view(this.to_page(cx));
                    }))),
            )
    }
}
```

### 9.5.3 多页面状态保持

`replace_root_view()` 会替换根视图，旧 View 的 Entity 如果还有引用则存活，否则被销毁。要保存页面状态：

```rust
struct App {
    home: Option<Entity<HomePage>>,
    settings: Option<Entity<SettingsPage>>,
    current_page: Page,
}

enum Page { Home, Settings }

impl App {
    fn navigate(&mut self, page: Page, window: &mut Window, cx: &mut Context<Self>) {
        let view = match page {
            Page::Home => {
                // 复用已有的 Entity，保留状态
                self.home.get_or_insert_with(|| {
                    cx.new(|cx| HomePage { data: Vec::new() })
                }).clone().into()
            }
            Page::Settings => {
                self.settings.get_or_insert_with(|| {
                    cx.new(|cx| SettingsPage { theme: "dark".into() })
                }).clone().into()
            }
        };
        self.current_page = page;
        window.replace_root_view(view);
    }
}
```

通过保留 `Entity` 引用而不是每次创建新的 View，页面切换时可以保持状态。
