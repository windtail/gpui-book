# 第32章：Key Context

## 32.1 `KeyContext` 是什么

### 32.1.1 key-value 对

`KeyContext` 是元素在键匹配时提供的上下文信息。它由一组标识符和键值对组成：

```rust
use gpui::KeyContext;

let mut context = KeyContext::new_with_defaults();
context.add("Editor");           // 标识符
context.set("mode", "normal");   // 键值对
context.set("language", "rust");
```

打印结果：`Editor mode=normal language=rust`

### 32.1.2 上下文栈

每个元素可以设置自己的 `KeyContext`。GPUI 从根元素到当前元素收集所有上下文，形成上下文栈。键匹配时，系统使用整个上下文栈来判断哪个绑定应该生效。

```
上下文栈（从根到叶子）：
[
  "os=macos Workspace",          // 窗口根
  "Pane",                         // 面板
  "Editor mode=normal language=rust",  // 编辑器
]
```

## 32.2 设置上下文

### 32.2.1 `.key_context()` 在元素上

```rust
use gpui::div, InteractiveElement;

// 添加单个标识符
div()
    .id("editor")
    .key_context("Editor")
    .child("编辑区域")

// 添加键值对
div()
    .id("vim-editor")
    .key_context("Editor mode=normal")
    .child("Vim 编辑器")
```

`.key_context()` 接受任何可转换为 `KeyContext` 的类型：

```rust
// 字符串字面量
.key_context("Editor")

// 多标识符和键值对
.key_context("Editor mode=normal language=rust")

// 手动构建
.key_context({
    let mut ctx = KeyContext::new_with_defaults();
    ctx.add("Editor");
    ctx.set("mode", "normal");
    ctx.set("has_selection", "true");
    ctx
})
```

### 32.2.2 方法链

```rust
struct VimEditor {
    mode: String,
    language: String,
}

impl Render for VimEditor {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let context = format!("Editor mode={} language={}", self.mode, self.language);

        div()
            .id("vim-editor")
            .size_full()
            .key_context(context)
            .child("...")
    }
}
```

### 32.2.3 动态更新

上下文在每次渲染时重建，状态变化后自动更新：

```rust
struct VimEditor {
    mode: VimMode,
}

enum VimMode {
    Normal,
    Insert,
    Visual,
}

impl Render for VimEditor {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let mode_str = match self.mode {
            VimMode::Normal => "normal",
            VimMode::Insert => "insert",
            VimMode::Visual => "visual",
        };

        div()
            .id("vim-editor")
            .key_context(format!("Editor mode={}", mode_str))
            .child(format!("Vim mode: {}", mode_str))
    }
}
```

模式切换时，上下文自动变化，键绑定也随之切换。

## 32.3 上下文谓词

### 32.3.1 `KeyBindingContextPredicate` 枚举

上下文谓词用于决定 `KeyBinding` 在当前上下文中是否生效：

```rust
pub enum KeyBindingContextPredicate {
    Identifier(SharedString),                  // 匹配标识符
    Equal(SharedString, SharedString),         // 匹配键值对
    NotEqual(SharedString, SharedString),      // 不匹配键值对
    Descendant(Box<Self>, Box<Self>),          // 父 > 子
    Not(Box<Self>),                            // 非
    And(Box<Self>, Box<Self>),                 // 与
    Or(Box<Self>, Box<Self>),                  // 或
}
```

### 32.3.2 `Identifier`、`Equal`、`NotEqual`

| 谓词字符串 | 匹配规则 |
|-----------|---------|
| `Editor` | 上下文中包含标识符 `Editor` |
| `mode == normal` | 上下文中 `mode` 键的值为 `normal` |
| `mode != insert` | 上下文中 `mode` 键的值不为 `insert`，或 `mode` 不存在 |
| `!Editor` | 上下文中不包含标识符 `Editor` |

```rust
use gpui::KeyBindingContextPredicate;

let p = KeyBindingContextPredicate::parse("Editor").unwrap();
// 等价于 Identifier("Editor")

let p = KeyBindingContextPredicate::parse("mode == normal").unwrap();
// 等价于 Equal("mode", "normal")

let p = KeyBindingContextPredicate::parse("mode != insert").unwrap();
// 等价于 NotEqual("mode", "insert")
```

### 32.3.3 `Descendant`、`And`、`Or`、`Not`

| 谓词字符串 | 含义 |
|-----------|------|
| `Workspace > Editor` | Editor 是 Workspace 的子上下文 |
| `Editor && mode == normal` | Editor 且 mode 为 normal |
| `Editor \|\| Terminal` | Editor 或 Terminal |
| `!Editor` | 非 Editor |
| `Editor && mode != insert` | Editor 且 mode 不为 insert |

```rust
// 组合谓词
KeyBinding::new("dd", DeleteLine, Some("Editor && mode == normal"));
KeyBinding::new("v", ToggleVisual, Some("Editor && (mode == normal || mode == insert)"));
KeyBinding::new("escape", ExitMode, Some("Editor && mode != normal"));

//  descendant（父 > 子）
KeyBinding::new("ctrl-f", Find, Some("Workspace > Editor"));
```

### 32.3.4 谓词匹配规则

谓词在上下文栈上求值：

```rust
let predicate = KeyBindingContextPredicate::parse("Editor && mode == normal").unwrap();

// 上下文栈
let contexts = vec![
    KeyContext::parse("os=macos Workspace").unwrap(),
    KeyContext::parse("Pane").unwrap(),
    KeyContext::parse("Editor mode=normal").unwrap(),
];

assert!(predicate.eval(&contexts));  // true
```

`depth_of()` 方法返回匹配的最深深度，用于确定优先级：

```rust
let depth = predicate.depth_of(&contexts);
// Some(3) — 在第3层（Editor）匹配
```

深层上下文优先于浅层。同一层后注册的优先于先注册的。

## 32.4 实战：编辑器模式切换

### 32.4.1 normal 模式 vs insert 模式

```rust
use gpui::{actions, KeyBinding, KeyContext, div, InteractiveElement, Context, Window, IntoElement};

actions!(vim, [
    InsertAtCursor, ExitInsertMode,
    DeleteCharUnderCursor, DeleteCharBeforeCursor,
    MoveCursorUp, MoveCursorDown, MoveCursorLeft, MoveCursorRight,
    EnterVisualMode, Yank, DeleteLine,
]);

struct VimEditor {
    mode: VimMode,
    content: String,
    cursor: usize,
}

enum VimMode {
    Normal,
    Insert,
    Visual,
}

impl Render for VimEditor {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let mode_str = match self.mode {
            VimMode::Normal => "normal",
            VimMode::Insert => "insert",
            VimMode::Visual => "visual",
        };

        div()
            .id("vim-editor")
            .size_full()
            .key_context(format!("Editor mode={}", mode_str))
            // Normal 模式绑定
            .on_action(cx.listener(|this, _: &InsertAtCursor, _, cx| {
                this.mode = VimMode::Insert;
                cx.notify();
            }))
            .on_action(cx.listener(|this, _: &EnterVisualMode, _, cx| {
                this.mode = VimMode::Visual;
                cx.notify();
            }))
            // Insert 模式绑定（通过上下文过滤）
            .on_action(cx.listener(|this, _: &ExitInsertMode, _, cx| {
                this.mode = VimMode::Normal;
                cx.notify();
            }))
            .child(format!("-- {} MODE --", mode_str))
            .child(&this.content)
    }
}

fn register_vim_keybindings(cx: &mut gpui::App) {
    cx.bind_keys(vec![
        // Normal 模式
        KeyBinding::new("i", InsertAtCursor, Some("Editor && mode == normal")),
        KeyBinding::new("v", EnterVisualMode, Some("Editor && mode == normal")),
        KeyBinding::new("x", DeleteCharUnderCursor, Some("Editor && mode == normal")),
        KeyBinding::new("dd", DeleteLine, Some("Editor && mode == normal")),
        KeyBinding::new("y", Yank, Some("Editor && mode == normal")),

        // Insert 模式
        KeyBinding::new("escape", ExitInsertMode, Some("Editor && mode == insert")),

        // 两个模式都有效的绑定
        KeyBinding::new("up", MoveCursorUp, Some("Editor")),
        KeyBinding::new("down", MoveCursorDown, Some("Editor")),
    ]);
}
```

### 32.4.2 不同模式的快捷键覆盖

同一快捷键在不同模式下绑定不同操作：

```rust
cx.bind_keys(vec![
    // normal: d 开始删除命令（等待后续键）
    KeyBinding::new("d", StartDeleteCommand, Some("Editor && mode == normal")),

    // insert: d 就是输入字符 d
    // 不需要绑定，因为没有 Editor && mode == insert 的 d 绑定

    // visual: d 执行删除
    KeyBinding::new("d", ExecuteDelete, Some("Editor && mode == visual")),
]);
```

### 32.4.3 上下文动态切换

```rust
struct MultiPaneEditor {
    panes: Vec<Pane>,
    active_pane: usize,
}

struct Pane {
    buffers: Vec<Buffer>,
    active_buffer: usize,
    mode: String,  // "normal" / "insert" / "visual"
}

impl Render for MultiPaneEditor {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("multi-pane")
            .key_context("Workspace")
            .child(
                // 每个 Pane 有自己的上下文
                div()
                    .id(("pane", self.active_pane))
                    .key_context("Pane")
                    .child(
                        // 每个 Buffer 有自己的上下文
                        div()
                            .id(("buffer", self.active_buffer()))
                            .key_context(format!(
                                "Editor mode={}",
                                self.active_pane_mode()
                            ))
                    )
            )
    }
}
```

上下文栈自动反映了当前焦点元素的层级关系：

```
焦点在 normal 模式的编辑器中：
["os=macos Workspace", "Pane", "Editor mode=normal"]

焦点在 insert 模式的编辑器中：
["os=macos Workspace", "Pane", "Editor mode=insert"]

焦点在搜索栏中：
["os=macos Workspace", "SearchBar"]
```
