# 第10章：View 组合模式

## 10.1 视图组合基础

### 10.1.1 父 View 引用子 Entity

```rust
struct ParentView {
    child: Entity<ChildView>,
}

struct ChildView {
    name: String,
}

impl Render for ChildView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(self.name.as_str())
    }
}

impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(div().child("Parent content"))
            .child(self.child.clone())  // 子 Entity 直接嵌入元素树
    }
}

impl ParentView {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            child: cx.new(|cx| ChildView { name: "Child".into() }),
        }
    }
}
```

### 10.1.2 子 View 通过回调通知父 View

```rust
struct ParentView {
    child: Entity<ChildView>,
    message: Option<String>,
    _subscription: gpui::Subscription,
}

// 子 View 定义事件类型
enum ChildEvent {
    Completed(String),
    Deleted,
}

impl EventEmitter<ChildEvent> for ChildView {}

struct ChildView {
    done: bool,
}

impl Render for ChildView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(button().label("Done").on_click(cx.listener(|this, _, _, cx| {
                this.done = true;
                cx.emit(ChildEvent::Completed("finished".into()));
            })))
    }
}

impl ParentView {
    fn new(cx: &mut Context<Self>) -> Self {
        let child = cx.new(|cx| ChildView { done: false });

        Self {
            child: child.clone(),
            message: None,
            _subscription: cx.subscribe(&child, |this, _, event, cx| {
                match event {
                    ChildEvent::Completed(msg) => {
                        this.message = Some(format!("Child done: {}", msg));
                    }
                    ChildEvent::Deleted => {
                        this.message = Some("Child deleted".into());
                    }
                }
                cx.notify();
            }),
        }
    }
}
```

### 10.1.3 深层嵌套与 prop drilling

```rust
// 三层嵌套：GrandParent → Parent → Child
struct GrandParent { data: SharedString }
struct Parent { data: SharedString }
struct Child { data: SharedString }

// 每层都需要传递 data，这就是 prop drilling
impl Render for GrandParent {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(self.data.as_str())
            .child(Parent { data: self.data.clone() })
    }
}

impl RenderOnce for Parent {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div().child(self.data.as_str())
            .child(Child { data: self.data.clone() })
    }
}

impl RenderOnce for Child {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div().child(self.data.as_str())
    }
}
```

避免深层 prop drilling 的两种方式：

**方式一：共享 Entity**

```rust
struct SharedData { value: SharedString }

struct DeepChild {
    shared: Entity<SharedData>,
}

impl Render for DeepChild {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let value = self.shared.read(cx).value.clone();
        div().child(value.as_str())
    }
}
// 所有层级直接读取 shared，无需传递
```

**方式二：全局状态**

```rust
struct AppState { theme: String }
impl Global for AppState {}

struct DeepChild {}

impl Render for DeepChild {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let theme = cx.global::<AppState>().theme.clone();
        div().child(theme.as_str())
    }
}
```

## 10.2 组件化模式

### 10.2.1 可复用 UI 组件的定义

使用 `RenderOnce` + `#[derive(IntoElement)]` 定义无状态组件：

```rust
#[derive(IntoElement)]
struct Card {
    title: SharedString,
    body: SharedString,
    footer: Option<SharedString>,
}

impl RenderOnce for Card {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div()
            .border_1()
            .rounded_lg()
            .shadow_md()
            .overflow_hidden()
            .child(
                div()
                    .p_4()
                    .border_b_1()
                    .child(div().font_semibold().child(self.title)),
            )
            .child(
                div()
                    .p_4()
                    .child(div().child(self.body)),
            )
            .when_some(self.footer, |this, footer| {
                this.child(
                    div()
                        .p_4()
                        .border_t_1()
                        .bg_gray_50()
                        .child(div().text_sm().text_gray_500().child(footer)),
                )
            })
    }
}
```

### 10.2.2 组件参数传递

```rust
#[derive(IntoElement)]
struct ButtonGroup {
    buttons: Vec<ButtonDef>,
    variant: ButtonVariant,
}

enum ButtonVariant { Primary, Secondary, Danger }

struct ButtonDef {
    label: SharedString,
    on_click: Box<dyn Fn(&mut Window, &mut App)>,
}

impl RenderOnce for ButtonGroup {
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement {
        div().flex().gap_2().children(
            self.buttons.into_iter().map(|def| {
                let btn = match self.variant {
                    ButtonVariant::Primary => button().label(def.label).primary(),
                    ButtonVariant::Secondary => button().label(def.label).secondary(),
                    ButtonVariant::Danger => button().label(def.label).danger(),
                };
                // 注意：RenderOnce 中需要特殊处理事件绑定
                btn
            })
        )
    }
}
```

对于需要事件回调的组件，传递闭包参数：

```rust
#[derive(IntoElement)]
struct InputField {
    value: SharedString,
    placeholder: SharedString,
    on_change: Box<dyn FnMut(SharedString, &mut Window, &mut App)>,
}
```

### 10.2.3 槽模式 (slot pattern)

```rust
#[derive(IntoElement)]
struct Panel {
    header: Option<AnyElement>,
    content: AnyElement,
    sidebar: Option<AnyElement>,
}

impl Panel {
    pub fn new(content: impl IntoElement) -> Self {
        Self {
            header: None,
            content: content.into_any_element(),
            sidebar: None,
        }
    }

    pub fn header(mut self, header: impl IntoElement) -> Self {
        self.header = Some(header.into_any_element());
        self
    }

    pub fn sidebar(mut self, sidebar: impl IntoElement) -> Self {
        self.sidebar = Some(sidebar.into_any_element());
        self
    }
}

impl RenderOnce for Panel {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div().flex().h_full().children(
            self.sidebar.map(|sidebar| {
                div().w(px(200.0)).border_r_1().child(sidebar)
            })
        ).child(
            div().flex_col().flex_1().children(
                self.header.map(|header| {
                    div().p_3().border_b_1().child(header)
                })
            ).child(
                div().p_4().flex_1().child(self.content)
            )
        )
    }
}

// 使用
Panel::new(div().child("Main content"))
    .header(div().child("Header"))
    .sidebar(div().child("Sidebar"))
```

## 10.3 何时拆分视图

### 10.3.1 性能考量：独立更新

```rust
// 不拆分：父 View 中任何状态变化都会导致整个 UI 重新渲染
struct MonolithicView {
    header: HeaderState,
    items: Vec<Item>,
    footer: FooterState,
}

// 拆分：Header 和 Footer 不受 items 变化影响
struct SplitView {
    header: Entity<HeaderView>,
    items: Entity<ItemList>,
    footer: Entity<FooterView>,
}

impl Render for SplitView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().flex_col().h_full()
            .child(self.header.clone())       // header 只在自己 notify 时重绘
            .child(self.items.clone().flex_1()) // items 变化不影响 header/footer
            .child(self.footer.clone())       // footer 只在自己 notify 时重绘
    }
}
```

当子 View 是独立 Entity 时，父 View 的 `render()` 返回子 Entity 句柄，子 View 只有在自己的状态变化时才重新渲染。

### 10.3.2 关注点分离

```rust
// 拆分成独立模块
struct EditorView {
    buffer: Entity<Buffer>,
    toolbar: Entity<Toolbar>,
    status_bar: Entity<StatusBar>,
}

// Toolbar 专注于工具栏逻辑
struct Toolbar {
    actions: Vec<Action>,
    search_open: bool,
}

// StatusBar 专注于状态显示
struct StatusBar {
    cursor_position: (usize, usize),
    file_type: String,
    encoding: String,
}
```

### 10.3.3 过度拆分的代价

```rust
// 过度拆分：每个 Item 都是独立 Entity
struct OverSplitView {
    items: Vec<Entity<ItemView>>,  // 1000 个 Entity！
}

// 更好的做法：单个 View 管理列表
struct BetterView {
    items: Vec<ItemData>,
}

impl Render for BetterView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // 用 uniform_list 或 list 做虚拟列表
        uniform_list(cx.view().clone(), "items", self.items.len(),
            |this, range, window, cx| {
                this.items[range].iter().map(|item| {
                    div().child(item.name.as_str())
                }).collect()
            }
        )
    }
}
```

拆分原则：**状态变更频率不同、布局独立、逻辑清晰的边界才拆分**。

## 10.4 视图通信

### 10.4.1 直接 Entity::update

```rust
struct Parent {
    child: Entity<Child>,
}

impl Parent {
    fn update_child_name(&mut self, cx: &mut Context<Self>) {
        self.child.update(cx, |child, cx| {
            child.name = "Updated".into();
            cx.notify();
        });
    }
}

struct Child {
    name: SharedString,
}
```

适用于父子关系明确、路径直接的更新。

### 10.4.2 事件订阅

```rust
struct EventBus {}

enum AppEvent {
    UserLoggedIn(Entity<User>),
    UserLoggedOut,
    ThemeChanged(String),
}

impl EventEmitter<AppEvent> for EventBus {}

struct Listener {
    _sub: gpui::Subscription,
}

impl Listener {
    fn new(bus: Entity<EventBus>, cx: &mut Context<Self>) -> Self {
        Self {
            _sub: cx.subscribe(&bus, |this, _, event, cx| {
                match event {
                    AppEvent::ThemeChanged(theme) => {
                        this.apply_theme(theme);
                        cx.notify();
                    }
                    _ => {}
                }
            }),
        }
    }
}
```

适用于一对多、解耦的通信场景。

### 10.4.3 闭包回调链

```rust
#[derive(IntoElement)]
struct ConfirmDialog {
    message: SharedString,
    on_confirm: Box<dyn FnOnce(&mut Window, &mut App)>,
    on_cancel: Box<dyn FnOnce(&mut Window, &mut App)>,
}

impl RenderOnce for ConfirmDialog {
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement {
        div()
            .child(div().child(self.message))
            .child(
                div().flex().gap_2()
                    .child(button().label("Confirm").on_click(move |_, window, cx| {
                        (self.on_confirm)(window, cx);
                    }))
                    .child(button().label("Cancel").on_click(move |_, window, cx| {
                        (self.on_cancel)(window, cx);
                    })),
            )
    }
}
```

适用于一次性回调，用完即丢弃。

## 10.5 实战：构建一个可复用的模态框组件

```rust
use gpui::{
    div, button, IntoElement, Context, Window, App, RenderOnce,
    SharedString, Entity, ElementId, Pixels, rgba, rgb,
    div::Div, prelude::*, Styled,
};

// 模态框内容组件
#[derive(IntoElement)]
struct Modal {
    id: ElementId,
    title: SharedString,
    body: AnyElement,
    actions: Vec<AnyElement>,
    on_close: Box<dyn FnOnce(&mut Window, &mut App)>,
}

impl Modal {
    pub fn new(title: impl Into<SharedString>, body: impl IntoElement) -> Self {
        Self {
            id: ElementId::Name(SharedString::new_static("modal")),
            title: title.into(),
            body: body.into_any_element(),
            actions: Vec::new(),
            on_close: Box::new(|_, _| {}),
        }
    }

    pub fn action(mut self, action: impl IntoElement) -> Self {
        self.actions.push(action.into_any_element());
        self
    }

    pub fn on_close(mut self, handler: impl FnOnce(&mut Window, &mut App) + 'static) -> Self {
        self.on_close = Box::new(handler);
        self
    }
}

impl RenderOnce for Modal {
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement {
        let on_close = self.on_close;

        div()
            .id(self.id)
            .fixed()
            .inset_0()
            .bg(rgba(0x00000080))  // 半透明遮罩
            .flex()
            .items_center()
            .justify_center()
            .on_click(move |_, window, cx| {
                // 点击遮罩关闭
                on_close(window, cx);
            })
            .child(
                div()
                    .rounded_lg()
                    .bg(Colors::get(cx).background)
                    .shadow_xl()
                    .w(px(480.0))
                    .max_h(re(0.8))
                    .overflow_hidden()
                    .on_click(|e, _, cx| e.stop_propagation())  // 阻止事件冒泡到遮罩
                    .child(
                        div()
                            .flex()
                            .justify_between()
                            .items_center()
                            .p_4()
                            .border_b_1()
                            .child(div().font_semibold().child(self.title.as_str()))
                            .child(
                                button()
                                    .label("×")
                                    .small()
                                    .on_click(move |_, window, cx| {
                                        on_close(window, cx);
                                    }),
                            ),
                    )
                    .child(
                        div().p_4().max_h(re(0.6)).overflow_y_scroll()
                            .child(self.body),
                    )
                    .when(!self.actions.is_empty(), |this| {
                        this.child(
                            div()
                                .flex()
                                .justify_end()
                                .gap_2()
                                .p_4()
                                .border_t_1()
                                .children(self.actions),
                        )
                    }),
            )
    }
}

// 使用示例
struct AppView {
    showing_modal: bool,
}

impl Render for AppView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .p_4()
            .child(
                button()
                    .label("Show Modal")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.showing_modal = true;
                        cx.notify();
                    })),
            )
            .when(self.showing_modal, |this| {
                this.child(
                    Modal::new("Confirm Action", div().child("Are you sure?"))
                        .action(
                            button().label("Cancel").on_click(cx.listener(|this, _, _, cx| {
                                this.showing_modal = false;
                                cx.notify();
                            }))
                        )
                        .action(
                            button().label("Confirm").on_click(cx.listener(|this, _, _, cx| {
                                // 执行确认操作
                                this.showing_modal = false;
                                cx.notify();
                            }))
                        )
                        .on_click(cx.listener(|this, _, _, cx| {
                            this.showing_modal = false;
                            cx.notify();
                        }))
                )
            })
    }
}
```

模态框的关键点：
- 遮罩层用 `fixed().inset_0()` 覆盖整个窗口
- 内容区用 `on_click(e.stop_propagation())` 阻止事件冒泡
- 通过闭包传递 `on_close` 回调
- 使用 `.when()` 条件渲染控制显示/隐藏
