# 第34章：焦点系统

## 34.1 `FocusHandle`

### 34.1.1 创建：`cx.focus_handle()`

`FocusHandle` 是 GPUI 中表示焦点的句柄。每个需要接收焦点的 View 都应持有至少一个 `FocusHandle`。

```rust
use gpui::{FocusHandle, Context, Window, div, InteractiveElement, IntoElement};

struct InputField {
    focus_handle: FocusHandle,
    value: String,
}

impl InputField {
    fn new(cx: &mut Context<Self>) -> Self {
        InputField {
            focus_handle: cx.focus_handle(),
            value: String::new(),
        }
    }
}

impl Render for InputField {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("input-field")
            .track_focus(&self.focus_handle)
            .child(&self.value)
    }
}
```

### 34.1.2 `FocusId` 唯一标识

`FocusHandle` 内部持有 `FocusId`，用于唯一标识焦点目标：

```rust
pub struct FocusHandle {
    id: FocusId,
    handles: Arc<Mutex<...>>,
    tab_index: Cell<Option<usize>>,
    tab_stop: Cell<bool>,
    // ...
}
```

`FocusHandle` 除了持有 `FocusId`，还包含以下字段：

- **`id`**: `FocusId` 唯一标识符
- **`handles`**: 内部共享状态，用于追踪同一焦点目标的所有句柄
- **`tab_index`**: 可通过 `clone()` 方法设置的 Tab 索引，控制元素在 Tab 导航中的顺序
- **`tab_stop`**: 可通过 `clone()` 方法设置，控制该句柄是否参与 Tab 导航（`false` 时可聚焦但 Tab 跳过）

多个 `FocusHandle` 可以有相同的 `FocusId`（通过克隆获得），表示同一个焦点目标。

### 34.1.3 弱引用与生命周期

`WeakFocusHandle` 是 `FocusHandle` 的弱引用版本，不阻止焦点目标被销毁：

```rust
// 在回调中捕获弱引用
let weak_handle = self.focus_handle.downgrade();

// 使用时尝试升级
if let Some(handle) = weak_handle.upgrade() {
    // handle 仍然有效
}
```

## 34.2 焦点追踪

### 34.2.1 `track_focus()` 在元素上

`track_focus()` 将元素与焦点状态关联：

```rust
struct InputField {
    focus_handle: FocusHandle,
    focused: bool,
    value: String,
}

impl Render for InputField {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("input-field")
            .track_focus(&self.focus_handle)
            .border_1()
            .border_color(if self.is_focused(cx) { blue_500 } else { gray_300 })
            .child(&self.value)
    }

    fn is_focused(&self, window: &Window) -> bool {
        self.focus_handle.is_focused(window)
    }
}
```

`.focusable()` 快捷方法标记元素为可聚焦（但不追踪特定 handle）：

```rust
// 仅使元素可聚焦，不追踪特定 handle
div()
    .id("input")
    .focusable()
```

如需关联特定 `FocusHandle` 以应用样式，使用 `.track_focus()`：

```rust
// 追踪 handle 并设置 focusable = true
div()
    .id("input")
    .track_focus(&self.focus_handle)
```

### 34.2.2 焦点变化回调

使用 `cx.on_focus()` 和 `cx.on_blur()` 监听焦点变化：

```rust
struct SearchBar {
    focus_handle: FocusHandle,
    expanded: bool,
}

impl SearchBar {
    fn new(cx: &mut Context<Self>, window: &mut Window) -> Self {
        let focus_handle = cx.focus_handle();

        // 监听获得焦点
        cx.on_focus(&focus_handle, window, |this, window, cx| {
            this.expanded = true;
            cx.notify();
        })
        .detach();

        // 监听失去焦点
        cx.on_blur(&focus_handle, window, |this, window, cx| {
            this.expanded = false;
            cx.notify();
        })
        .detach();

        SearchBar {
            focus_handle,
            expanded: false,
        }
    }
}
```

### 34.2.3 焦点变化回调

焦点离开元素时，通过 `cx.on_blur()` 在 `new()` 中注册回调：

```rust
struct AutoSaveInput {
    focus_handle: FocusHandle,
    value: String,
    saved_value: String,
}

impl AutoSaveInput {
    fn new(cx: &mut Context<Self>, window: &mut Window) -> Self {
        let focus_handle = cx.focus_handle();

        cx.on_blur(&focus_handle, window, |this, window, cx| {
            // 失去焦点时自动保存
            if this.value != this.saved_value {
                this.save();
                this.saved_value = this.value.clone();
            }
            cx.notify();
        })
        .detach();

        AutoSaveInput {
            focus_handle,
            value: String::new(),
            saved_value: String::new(),
        }
    }
}
```

## 34.3 焦点管理

### 34.3.1 `cx.focus()` 设置焦点

```rust
// 将焦点设置到指定元素
window.focus(&self.focus_handle, cx);

// 通常在按钮点击中设置焦点
div()
    .id("edit-button")
    .on_click(cx.listener(|this, _, window, cx| {
        window.focus(&this.input_focus_handle, cx);
        cx.notify();
    }))
    .child("编辑")
```

### 34.3.2 `window.blur()` 移除焦点

```rust
// 移除焦点，焦点回到窗口
window.blur();

// 在 Escape 键处理中
.on_key_down(cx.listener(|this, event: &KeyDownEvent, window, cx| {
    if event.keystroke.key == "escape" {
        window.blur();
        cx.stop_propagation();
        cx.prevent_default();
    }
}))
```

### 34.3.3 `is_focused()` 检查焦点

```rust
impl InputField {
    fn is_focused(&self, window: &Window) -> bool {
        self.focus_handle.is_focused(window)
    }

    fn contains_focused(&self, window: &Window, cx: &App) -> bool {
        self.focus_handle.contains_focused(window, cx)
    }
}
```

- `focus_handle.is_focused(window)`：该 handle 是否是当前焦点元素
- `focus_handle.contains_focused(window, cx)`：当前焦点是否在该 handle 或其子元素中

## 34.4 Tab 导航

### 34.4.1 `TabStopMap` 焦点顺序

Tab 键在可聚焦元素间按顺序移动焦点。默认顺序是渲染顺序。

自定义 Tab 顺序使用 `tab_index()`：

```rust
struct LoginForm {
    username_focus: FocusHandle,
    password_focus: FocusHandle,
    submit_focus: FocusHandle,
}

impl Render for LoginForm {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("login-form")
            .child(
                div()
                    .id("username-input")
                    .track_focus(&self.username_focus)
                    .tab_index(0)  // 第一个 Tab 停止
                    .child("用户名")
            )
            .child(
                div()
                    .id("password-input")
                    .track_focus(&self.password_focus)
                    .tab_index(1)  // 第二个 Tab 停止
                    .child("密码")
            )
            .child(
                div()
                    .id("submit-button")
                    .track_focus(&self.submit_focus)
                    .tab_index(2)  // 第三个 Tab 停止
                    .child("登录")
            )
    }
}
```

### 34.4.2 焦点分组

`tab_group()` 创建 Tab 分组，组内 Tab 索引相对于组重置：

```rust
div()
    .id("sidebar")
    .tab_group()  // 创建一个 Tab 组
    .tab_index(0)  // 组本身的 Tab 索引
    .child(
        div()
            .id("sidebar-item-1")
            .tab_index(0)  // 相对于组的索引
            .child("项目 1")
    )
    .child(
        div()
            .id("sidebar-item-2")
            .tab_index(1)
            .child("项目 2")
    )

// Tab 顺序：sidebar 组 → sidebar-item-1 → sidebar-item-2 → 下一个组的元素
```

嵌套分组：

```rust
div()
    .id("panel-1")
    .tab_index(0)
    .child(
        div()
            .id("panel-1-content")
            .tab_group()    // 子组
            .tab_index(0)
            .child(div().id("p1-item-1").tab_index(0))
            .child(div().id("p1-item-2").tab_index(1))
    )

// Tab 顺序：panel-1 → p1-item-1 → p1-item-2 → 下一个元素
```

### 34.4.3 自定义 Tab 顺序

`tab_stop(false)` 将元素从 Tab 导航中排除（但保持可聚焦）：

```rust
div()
    .id("clickable-but-not-tab-stop")
    .track_focus(&self.focus_handle)
    .tab_stop(false)  // Tab 跳过，但仍可点击聚焦
    .child("点击我但 Tab 不会到这里")
```

## 34.5 实战：表单焦点管理

```rust
use gpui::{
    FocusHandle, Context, Window, div, InteractiveElement, IntoElement,
    ParentElement, px,
};

struct LoginForm {
    username_focus: FocusHandle,
    password_focus: FocusHandle,
    submit_focus: FocusHandle,

    username: String,
    password: String,
    error: Option<String>,
}

impl LoginForm {
    fn new(cx: &mut Context<Self>) -> Self {
        LoginForm {
            username_focus: cx.focus_handle(),
            password_focus: cx.focus_handle(),
            submit_focus: cx.focus_handle(),
            username: String::new(),
            password: String::new(),
            error: None,
        }
    }

    fn submit(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        if self.username.is_empty() {
            self.error = Some("请输入用户名".into());
            window.focus(&self.username_focus, cx);
        } else if self.password.is_empty() {
            self.error = Some("请输入密码".into());
            window.focus(&self.password_focus, cx);
        } else {
            self.do_login();
        }
        cx.notify();
    }
}

impl Render for LoginForm {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("login-form")
            .w(px(400.0))
            .p_4()
            .child(
                div()
                    .id("username-field")
                    .track_focus(&self.username_focus)
                    .on_action(cx.listener(|this, _: &gpui::Submit, window, cx| {
                        this.submit(window, cx);
                    }))
                    .child("用户名：")
                    .child(&this.username)
            )
            .child(
                div()
                    .id("password-field")
                    .track_focus(&self.password_focus)
                    .child("密码：")
                    .child(&this.password)
            )
            .child(
                div()
                    .id("submit-button")
                    .track_focus(&self.submit_focus)
                    .on_click(cx.listener(|this, _, window, cx| {
                        this.submit(window, cx);
                    }))
                    .on_action(cx.listener(|this, _: &gpui::Submit, window, cx| {
                        this.submit(window, cx);
                    }))
                    .child("登录")
            )
            .when_some(&self.error, |d, e| {
                d.child(div().text_color(red_500).child(e))
            })
    }
}
```

表单提交时验证字段，验证失败将焦点设置到对应字段。Enter 键通过 `Submit` Action 触发提交。
