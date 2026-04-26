# 第 58 章：TestAppContext

`TestApp` 和 `TestAppWindow` 提供简洁的测试 API，比 `TestAppContext` 更易用。

## 58.1 TestApp

`TestApp` 封装了 GPUI 应用的测试环境：

```rust
pub struct TestApp {
    app: Rc<AppCell>,
    platform: Rc<TestPlatform>,
    background_executor: BackgroundExecutor,
    foreground_executor: ForegroundExecutor,
    dispatcher: TestDispatcher,
    text_system: Arc<TextSystem>,
}
```

### 创建 TestApp

```rust
// 默认创建
let mut app = TestApp::new();

// 带 seed 创建
let mut app = TestApp::with_seed(42);

// 自定义文本系统（用于真实字体渲染）
let mut app = TestApp::with_text_system(platform_text_system);
```

### 基本测试示例

```rust
#[test]
fn test_basic_usage() {
    let mut app = TestApp::new();

    let mut window = app.open_window(Counter::new);

    window.update(|counter, _window, cx| {
        counter.increment(cx);
    });

    window.read(|counter, _| {
        assert_eq!(counter.count, 1);
    });
}
```

## 58.2 update 与 read

```rust
// update：可变访问，自动 run_until_parked
pub fn update<R>(&mut self, f: impl FnOnce(&mut App) -> R) -> R

// read：只读访问
pub fn read<R>(&self, f: impl FnOnce(&App) -> R) -> R
```

`update()` 在闭包完成后自动调用 `run_until_parked()`，确保所有副作用刷新。

### Entity 快捷方法

```rust
// 创建 entity
let entity = app.new_entity(|cx| Counter {
    count: 42,
    focus_handle: cx.focus_handle(),
});

// 更新 entity
app.update_entity(&entity, |counter, _cx| {
    counter.count += 1;
});

// 读取 entity
app.read_entity(&entity, |counter, _| {
    assert_eq!(counter.count, 43);
});
```

## 58.3 窗口管理

### 打开窗口

```rust
pub fn open_window<V: Render + 'static>(
    &mut self,
    build_view: impl FnOnce(&mut Window, &mut Context<V>) -> V,
) -> TestAppWindow<V>
```

使用最大化边界创建窗口：

```rust
let mut window = app.open_window(|window, cx| {
    MyView::new(window, cx)
});
```

### 带选项打开窗口

```rust
let mut window = app.open_window_with_options(
    WindowOptions {
        window_bounds: Some(WindowBounds::Windowed(bounds)),
        ..Default::default()
    },
    |window, cx| MyView::new(window, cx),
);
```

## 58.4 输入模拟

`TestAppWindow` 提供完整的输入事件模拟：

### 键盘输入

```rust
// 单个按键
window.simulate_keystroke("ctrl-c");

// 多个按键（空格分隔）
window.simulate_keystrokes("ctrl-a ctrl-c");

// 模拟文本输入
window.simulate_input("hello world");
```

按键字符串格式：`modifier-key`，如 `ctrl-c`、`shift-tab`、`cmd-v`。

### 鼠标输入

```rust
let pos = Point::new(px(100.), px(200.));

// 移动
window.simulate_mouse_move(pos);

// 按下
window.simulate_mouse_down(pos, MouseButton::Left);

// 释放
window.simulate_mouse_up(pos, MouseButton::Left);

// 完整点击
window.simulate_click(pos, MouseButton::Left);

// 滚动
window.simulate_scroll(pos, Point::new(px(0.), px(-50.)));
```

### 窗口事件

```rust
// 模拟 resize
window.simulate_resize(Size::new(px(800.), px(600.)));

// 强制重绘
window.draw();
```

## 58.5 时间与任务控制

```rust
// 运行所有待处理任务
app.run_until_parked();

// 推进模拟时钟
app.advance_clock(Duration::from_secs(1));

// 前台 spawn
app.spawn(|cx| async move {
    // 在主线程执行的异步代码
});

// 后台 spawn
app.background_spawn(async {
    // 在后台线程执行的代码
});
```

### 模拟平台操作

```rust
// 剪贴板
app.write_to_clipboard(ClipboardItem::new_string("copied text"));
let text = app.read_from_clipboard().unwrap().text().unwrap();

// URL 打开
let url = app.opened_url();

// 文件对话框模拟
app.simulate_new_path_selection(|path| {
    Some(path.join("selected_file.txt"))
});

// 提示对话框
app.simulate_prompt_answer("OK");
```

## 58.6 全局状态

```rust
struct MyGlobal(String);
impl Global for MyGlobal {}

// 检查是否存在
assert!(!app.has_global::<MyGlobal>());

// 设置
app.set_global(MyGlobal("hello".into()));

// 读取
app.read_global::<MyGlobal, _>(|global, _| {
    assert_eq!(global.0, "hello");
});

// 更新
app.update_global::<MyGlobal, _>(|global, _| {
    global.0 = "world".into();
});
```

## 58.7 TestAppWindow API

```rust
pub struct TestAppWindow<V> {
    handle: WindowHandle<V>,
    app: Rc<AppCell>,
    platform: Rc<TestPlatform>,
    background_executor: BackgroundExecutor,
}
```

常用方法：

```rust
// 获取窗口句柄
let handle = window.handle();

// 获取根 view
let root = window.root();

// 更新根 view
window.update(|view, window, cx| {
    view.do_something(cx);
});

// 读取根 view
window.read(|view, _| {
    assert_eq!(view.some_field, expected);
});
```

## 58.8 多窗口测试

```rust
#[test]
fn test_multiple_windows() {
    let mut app = TestApp::new();

    let mut window_a = app.open_window(|_, cx| ViewA::new(cx));
    let mut window_b = app.open_window(|_, cx| ViewB::new(cx));

    // 两个窗口共享同一个 app
    // 可以测试跨窗口状态共享
}
```

## 58.9 清理

测试结束时关闭 app：

```rust
app.update(|cx| cx.shutdown());
```

这释放所有资源，确保没有泄漏。
