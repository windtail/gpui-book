# 第30章：定义 Action

## 30.1 `Action` trait

### 30.1.1 trait 定义与核心方法

`Action` trait 是 GPUI 键盘驱动 UI 的核心。每个 Action 代表一个语义化操作。

```rust
pub trait Action: Any + Send {
    fn boxed_clone(&self) -> Box<dyn Action>;
    fn partial_eq(&self, action: &dyn Action) -> bool;
    fn name(&self) -> &'static str;
    fn name_for_type() -> &'static str where Self: Sized;
    fn build(value: serde_json::Value) -> anyhow::Result<Box<dyn Action>> where Self: Sized;
    fn action_json_schema(_: &mut schemars::SchemaGenerator) -> Option<schemars::Schema> where Self: Sized { None }
}
```

- `boxed_clone()`：克隆为 `Box<dyn Action>`
- `partial_eq()`：与另一个 Action 做动态相等比较
- `name()`：Action 实例的名称，用于 UI 显示
- `name_for_type()`：Action 类型的名称，用于键映射查找
- `build()`：从 JSON 值构建 Action，用于从配置文件加载快捷键

### 30.1.2 JSON 序列化与反序列化

Action 需要实现 `serde::Deserialize` 和 `schemars::JsonSchema` 才能从 JSON 加载：

```rust
use gpui::Action;
use serde::{Deserialize, Serialize};
use schemars::JsonSchema;

#[derive(Clone, PartialEq, Serialize, Deserialize, JsonSchema, Action)]
#[action(namespace = editor)]
pub struct GoToLine {
    pub line: usize,
}

// 从 JSON 构建
let json = serde_json::json!({"line": 42});
let action: Box<dyn Action> = GoToLine::build(json).unwrap();
```

## 30.2 `actions!()` 宏

### 30.2.1 快速定义无参数 Action

`actions!()` 宏为单元结构生成完整的 `Action` 实现：

```rust
use gpui::actions;

actions!(editor, [
    MoveUp,
    MoveDown,
    MoveLeft,
    MoveRight,
    SelectAll,
    DeleteLine,
]);
```

展开后等价于：

```rust
#[derive(Clone, PartialEq, Default, Debug, Action)]
#[action(namespace = editor)]
pub struct MoveUp;

#[derive(Clone, PartialEq, Default, Debug, Action)]
#[action(namespace = editor)]
pub struct MoveDown;

// ... 依此类推
```

每个 Action 的名称为 `editor::MoveUp`、`editor::MoveDown` 等。

### 30.2.2 命名空间约定

命名空间用于区分不同模块的同名 Action：

```rust
actions!(editor, [Copy, Cut, Paste]);
actions!(terminal, [Copy, Paste]);

// 名称分别为：
// "editor::Copy", "editor::Cut", "editor::Paste"
// "terminal::Copy", "terminal::Paste"
```

在 Zed 中，每个模块都应使用独立的命名空间。命名空间也用于 Key Context 的谓词匹配。

## 30.3 `#[derive(Action)]`

### 30.3.1 带参数的 Action

需要携带数据的 Action 使用 derive 宏：

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
#[action(namespace = editor)]
pub struct FindInSelection {
    pub pattern: String,
}

#[derive(Clone, PartialEq, Serialize, Deserialize, JsonSchema, Action)]
#[action(namespace = workspace)]
pub struct ActivatePane {
    pub direction: PaneDirection,
}

#[derive(Clone, PartialEq, Serialize, Deserialize, JsonSchema, Action)]
#[action(namespace = workspace)]
pub struct SwitchMode {
    pub mode: String,
}
```

### 30.3.2 `#[action(...)]` 属性

```rust
#[derive(Clone, PartialEq, Action)]
#[action(namespace = editor, name = "CustomActionName")]
pub struct CustomAction;
```

可用属性：

| 属性 | 说明 |
|------|------|
| `namespace = mod` | 设置命名空间 |
| `name = "str"` | 覆盖 Action 名称 |
| `no_json` | 跳过 JSON 序列化支持 |
| `no_register` | 跳过自动注册 |

`no_json` 允许不实现 `serde::Serialize` 和 `schemars::JsonSchema`：

```rust
#[derive(Clone, PartialEq, Action)]
#[action(namespace = app, no_json)]
pub struct InternalAction {
    // 包含不能序列化的字段
    pub callback: Box<dyn Fn()>,
}
```

## 30.4 `register_action!` 宏

### 30.4.1 手动注册

`register_action!()` 宏在 `main` 之前注册 Action 类型：

```rust
use gpui::register_action;
use serde::{Deserialize, Serialize};
use schemars::JsonSchema;

#[derive(Clone, PartialEq, Serialize, Deserialize, JsonSchema)]
pub struct Paste {
    pub content: gpui::SharedString,
}

impl gpui::Action for Paste {
    fn boxed_clone(&self) -> Box<dyn gpui::Action> {
        Box::new(self.clone())
    }

    fn partial_eq(&self, action: &dyn gpui::Action) -> bool {
        action.downcast_ref::<Self>().map_or(false, |a| a == self)
    }

    fn name(&self) -> &'static str {
        "Paste"
    }

    fn name_for_type() -> &'static str {
        "Paste"
    }

    fn build(value: serde_json::Value) -> anyhow::Result<Box<dyn gpui::Action>> {
        let content = serde_json::from_value(value)?;
        Ok(Box::new(Paste { content }))
    }
}

register_action!(Paste);
```

### 30.4.2 `register_action!` 与 `#[derive(Action)]` 的区别

`register_action!()` 宏用于手动实现了 `Action` trait 的结构体注册类型：

```rust
use gpui::register_action;

register_action!(Paste);
```

对于使用 `#[derive(Action)]` 的结构体，不需要手动注册——derive 宏已自动处理。`register_action!()` 主要用于无法使用 derive 的场景（如包含非序列化字段的手动实现）。

## 30.5 特殊 Action

### 30.5.1 `NoAction`

`NoAction` 表示"什么都不做"。用于禁用某个快捷键：

```rust
use gpui::NoAction;

// 在特定上下文中禁用 ctrl-p
cx.bind_keys(vec![
    KeyBinding::new("ctrl-p", NoAction, Some("Terminal")),
]);
```

### 30.5.2 `Unbind`

`Unbind` 解绑特定 Action 的某个快捷键：

`Unbind` 是元组结构体 `Unbind(pub SharedString)`，直接构造即可：

```rust
use gpui::Unbind;

// 在编辑器中解绑 tab 键的默认行为
cx.bind_keys(vec![
    KeyBinding::new("tab", Unbind("editor::Tab".into()), Some("Editor")),
]);
```

## 30.6 Action 设计原则

### 30.6.1 Action 应该是什么粒度

Action 应表达"做什么"，而非"怎么做"：

```rust
// 好的粒度：表达意图
actions!(editor, [DeleteLine, DeleteWord, MoveToBeginning]);

// 过细：模拟按键
// 不好的做法：不需要定义 KeyDown('d'), KeyDown('e'), ...

// 过粗：把所有操作合成一个
// 不好的做法：只有一个 EditorAction { kind: enum }
```

### 30.6.2 Action 与业务逻辑的边界

Action 只描述操作，具体逻辑在 handler 中：

```rust
// Action 定义 — 只描述操作
actions!(todo, [NewTodo, DeleteTodo, ToggleTodo]);

// Handler 实现 — 包含业务逻辑
div()
    .id("todo-list")
    .on_action(cx.listener(|this, _: &NewTodo, _, cx| {
        // 业务逻辑在这里
        this.items.push(Todo::new(this.next_id));
        this.next_id += 1;
        cx.notify();
    }))
```

### 30.6.3 Action 命名约定

- 使用动词短语：`Save`、`OpenFile`、`ToggleBold`
- 使用现在时：`Delete` 而非 `Deleted`
- 命名空间 + 动作：`editor::Save`、`terminal::Copy`
- 带方向的操作：`MoveUp`、`ScrollDown`、`ExpandLeft`
- 切换操作：`ToggleComment`、`ToggleWordWrap`
