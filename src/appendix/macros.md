# 附录 A：GPUI 宏一览

本附录汇总 GPUI 提供的所有宏，包括语法、参数和示例。

## `actions!` 宏

### 语法

```rust
// 带命名空间
actions!(namespace_name, [ActionA, ActionB, ActionC]);

// 不带命名空间
actions!([ActionA, ActionB]);
```

### 作用

为每个名称生成一个 unit struct，自动实现 `Action` trait 并注册到全局 action 注册表。

### 生成的代码

```rust
// actions!(editor, [MoveUp, MoveDown]);
// 等价于：

#[derive(Clone, PartialEq, Default, Debug, gpui::Action)]
#[action(namespace = editor)]
pub struct MoveUp;

#[derive(Clone, PartialEq, Default, Debug, gpui::Action)]
#[action(namespace = editor)]
pub struct MoveDown;
```

### 示例

```rust
use gpui::actions;

// Zed 风格的 action 定义
actions!(editor, [
    MoveUp,
    MoveDown,
    MoveLeft,
    MoveRight,
    SelectUp,
    SelectDown,
    Newline,
    Backspace,
    Delete,
    SelectAll,
]);

actions!(workspace, [
    CloseWindow,
    Save,
    SaveAll,
]);
```

### 命名约定

- 使用 `PascalCase` 命名 action 类型
- 命名空间使用 `snake_case`
- 动作名使用动词或动词短语：`Save`、`MoveUp`、`OpenFile`

---

## `#[derive(Action)]`

### 语法

```rust
#[derive(Action)]
#[action(namespace = editor)]
pub struct MyAction {
    pub field: Type,
}
```

### 属性参数

| 属性 | 说明 |
|------|------|
| `namespace = some_path` | 设置 action 的命名空间 |
| `name = "CustomName"` | 覆盖 action 的名称（不能包含 `::`） |
| `no_json` | 不生成 JSON 序列化/反序列化，不需要 `Serialize`/`JsonSchema` |
| `no_register` | 跳过自动注册，不支持按名称调用 |
| `deprecated = "message"` | 标记为废弃 |
| `deprecated_aliases = ["old::Name"]` | 指定废弃的旧名称 |

### 示例：带参数 Action

```rust
use gpui::Action;
use serde::{Deserialize, Serialize};
use schemars::JsonSchema;

#[derive(Clone, PartialEq, Serialize, Deserialize, JsonSchema, Action)]
#[action(namespace = editor)]
pub struct SelectNext {
    pub replace_newest: bool,
}

#[derive(Clone, PartialEq, Serialize, Deserialize, JsonSchema, Action)]
#[action(namespace = workspace)]
pub struct ActivatePane {
    pub direction: Direction,
}

#[derive(Clone, PartialEq, Serialize, Deserialize, JsonSchema, Action)]
#[action(namespace = workspace)]
#[action(name = "GoToLine")]
pub struct GoToLineAction {
    pub line: usize,
}
```

### 示例：no_json

```rust
#[derive(Clone, PartialEq, Action)]
#[action(namespace = editor)]
#[action(no_json)]
pub struct InternalAction {
    // 不需要 serde 的内部字段
    pub handle: SomeNonSerializableType,
}
```

### 手动实现 Action trait

当 `actions!` 宏和 `#[derive(Action)]` 都不满足时使用：

```rust
use gpui::{Action, SharedString, register_action};
use serde::{Deserialize, Serialize};
use schemars::JsonSchema;

#[derive(Clone, PartialEq, Serialize, Deserialize, JsonSchema)]
pub struct Paste {
    pub content: SharedString,
}

impl Action for Paste {
    fn boxed_clone(&self) -> Box<dyn Action> {
        Box::new(self.clone())
    }

    fn partial_eq(&self, other: &dyn Action) -> bool {
        other.downcast_ref::<Self>().map_or(false, |a| a == self)
    }

    fn name(&self) -> &'static str {
        "Paste"
    }

    fn name_for_type() -> &'static str {
        "Paste"
    }

    fn build(value: serde_json::Value) -> anyhow::Result<Box<dyn Action>> {
        let content = serde_json::from_value(value)?;
        Ok(Box::new(Paste { content }))
    }
}

register_action!(Paste);
```

---

## `#[register_action]` 属性宏

### 语法

```rust
#[register_action]
pub struct MyAction { ... }
```

### 作用

手动注册 action 到全局注册表。需要在 action struct 实现 `Action` trait 之后使用。

### 与 `#[derive(Action)]` 的关系

`#[derive(Action)]` 内部也会使用 `#[register_action]`。当你手动实现 `Action` trait 时，使用 `#[register_action]` 替代 derive。

### 示例

```rust
use gpui::{Action, register_action};

#[derive(Clone, PartialEq)]
pub struct CustomAction {
    pub value: i32,
}

impl Action for CustomAction {
    fn boxed_clone(&self) -> Box<dyn Action> {
        Box::new(CustomAction { value: self.value })
    }
    fn partial_eq(&self, other: &dyn Action) -> bool {
        other.downcast_ref::<Self>().map_or(false, |a| a == self)
    }
    fn name(&self) -> &'static str { "CustomAction" }
    fn name_for_type() -> &'static str { "CustomAction" }
    fn build(_value: serde_json::Value) -> anyhow::Result<Box<dyn Action>> {
        anyhow::bail!("not supported")
    }
}

register_action!(CustomAction);
```

---

## `#[gpui::test]` 宏

### 语法

```rust
#[gpui::test]
fn test_name(cx: &mut TestAppContext) {
    // 测试代码
}

#[gpui::test]
async fn test_async_name(cx: &mut TestAppContext) {
    // 异步测试代码
}

#[gpui::test]
fn test_with_views(cx: &mut TestAppContext) {
    // 需要多个测试上下文时使用
}
```

### 函数签名

| 签名 | 说明 |
|------|------|
| `fn test(cx: &mut TestAppContext)` | 基本测试，单上下文 |
| `async fn test(cx: &mut TestAppContext)` | 异步测试 |
| `fn test(cx: &mut TestAppContext, other_cx: &mut TestAppContext)` | 多上下文协作测试 |

### 环境变量

| 变量 | 说明 |
|------|------|
| `SEED` | 设置随机种子，用于复现失败的随机测试 |
| `RUST_LOG` | 控制测试日志级别 |

### 示例

```rust
use gpui::*;

#[gpui::test]
fn test_entity_creation(cx: &mut TestAppContext) {
    let entity = cx.new(|_cx| Counter { count: 0 });
    assert_eq!(entity.read(cx).count, 0);

    entity.update(cx, |this, _cx| {
        this.count += 1;
    });
    assert_eq!(entity.read(cx).count, 1);
}

#[gpui::test]
async fn test_async_spawn(cx: &mut TestAppContext) {
    let entity = cx.new(|_cx| Counter { count: 0 });

    let result = cx.spawn(|_, mut cx| async move {
        cx.background_executor().timer(std::time::Duration::from_millis(10)).await;
        42
    }).await;

    assert_eq!(result, 42);
}

#[gpui::test]
fn test_entity_observation(cx: &mut TestAppContext) {
    let entity = cx.new(|_cx| Counter { count: 0 });
    let observed = cx.new(|cx| {
        Observer {
            entity: entity.clone(),
            notification_count: 0,
        }
    });

    // 观察外部 entity
    observed.update(cx, |this, cx| {
        cx.observe(&entity, |_, _, cx| {
            cx.notify();
        }).detach();
    });

    entity.update(cx, |_, cx| {
        cx.notify();
    });

    // 触发渲染并验证
    cx.run_until_parked();
}
```

---

## `#[gpui::property_test]` 宏

### 语法

```rust
#[gpui::property_test]
fn test_property(cx: &mut TestAppContext, value: InputType) {
    // 属性测试代码
}
```

### 作用

集成 `proptest` 进行属性测试。自动生成随机输入并验证属性是否成立。

### 示例

```rust
use gpui::*;

#[gpui::property_test]
fn test_color_roundtrip(cx: &mut TestAppContext, hsla: Hsla) {
    // 验证颜色转换的往返一致性
    let serialized = format!("{:?}", hsla);
    // ... 验证解析后等于原始值
}

#[gpui::property_test]
fn test_bounds_containment(cx: &mut TestAppContext, bounds: Bounds<Pixels>) {
    // 验证 Bounds 的包含逻辑
    let center = bounds.center();
    assert!(bounds.contains(&center));
}
```

### 随机种子控制

```bash
# 复现特定失败案例
SEED=42 cargo test test_property
```

---

## `#[derive(IntoElement)]`

### 语法

```rust
#[derive(IntoElement)]
pub struct MyComponent {
    // 字段
}
```

### 作用

为 struct 生成 `IntoElement` trait 实现，使其可以作为 UI 元素使用。通常与 `RenderOnce` 配合实现无状态组件。

### 示例

```rust
use gpui::*;

#[derive(IntoElement)]
pub struct Avatar {
    src: ImageSource,
    size: Pixels,
}

impl RenderOnce for Avatar {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div()
            .w(self.size)
            .h(self.size)
            .rounded_full()
            .overflow_hidden()
            .child(img(self.src).size_full().object_fit(ObjectFit::Cover))
    }
}

// 使用
fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    div().child(
        Avatar {
            src: "https://...".into(), // From<&str> 转换
            size: px(32.),
        }
    )
}
```

### 与 `RenderOnce` 的关系

- `#[derive(IntoElement)]` 使 struct 可以作为元素使用
- `RenderOnce` 定义如何渲染该元素
- 两者配合实现可复用的无状态组件
- 组件参数通过 struct 字段传递（prop drilling）

### 带 builder 模式的组件

```rust
#[derive(IntoElement)]
pub struct Button {
    id: ElementId,
    label: SharedString,
    on_click: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App)>>,
}

impl Button {
    pub fn new(id: impl Into<ElementId>, label: impl Into<SharedString>) -> Self {
        Self {
            id: id.into(),
            label: label.into(),
            on_click: None,
        }
    }

    pub fn on_click(mut self, handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static) -> Self {
        self.on_click = Some(Box::new(handler));
        self
    }
}

impl RenderOnce for Button {
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement {
        div()
            .id(self.id)
            .child(self.label)
            .when_some(self.on_click, |div, handler| {
                div.on_click(handler)
            })
    }
}
```

---

## 宏使用决策树

```
需要定义 action？
├── 无参数，快速定义 → actions!(namespace, [Name])
├── 带参数，需序列化 → #[derive(Action)] + #[action(namespace = ...)]
├── 带参数，不需序列化 → #[derive(Action)] + #[action(no_json)]
└── 完全自定义实现 → 手动 impl Action + register_action!()

需要定义组件？
├── 无状态组件 → #[derive(IntoElement)] + impl RenderOnce
└── 有状态组件 → struct + impl Render

需要测试？
├── 单元测试 → #[gpui::test]
├── 异步测试 → #[gpui::test] + async fn
├── 属性测试 → #[gpui::property_test]
└── 多上下文 → #[gpui::test] + 多个 cx 参数
```
