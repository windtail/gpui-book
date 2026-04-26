# 第31章：键绑定

## 31.1 `KeyBinding` 结构

### 31.1.1 keystroke 字符串格式

`KeyBinding` 将按键序列映射到 Action：

```rust
use gpui::KeyBinding;

// 基础格式：修饰符 + 键名
KeyBinding::new("a", MyAction, None);           // 单键 a
KeyBinding::new("ctrl-a", MyAction, None);      // Ctrl+A
KeyBinding::new("cmd-shift-p", MyAction, None); // Cmd+Shift+P (macOS)
KeyBinding::new("alt-enter", MyAction, None);   // Alt+Enter
```

修饰符关键字：
- `cmd` — macOS 的 Command 键
- `ctrl` — Control 键
- `alt` — Alt/Option 键
- `shift` — Shift 键

### 31.1.2 绑定到 Action

```rust
actions!(editor, [Save, Undo, Redo, Find]);

cx.bind_keys(vec![
    KeyBinding::new("ctrl-s", Save, None),
    KeyBinding::new("ctrl-z", Undo, None),
    KeyBinding::new("ctrl-shift-z", Redo, None),
    KeyBinding::new("ctrl-f", Find, None),
]);
```

带参数的 Action：

```rust
#[derive(Clone, PartialEq, Action)]
#[action(namespace = editor)]
pub struct GoToLine {
    pub line: usize,
}

cx.bind_keys(vec![
    // 注意：参数通过 JSON 传入
    // 实际使用中通常不带参数绑定，而是通过 Action handler 弹出输入框
]);
```

### 31.1.3 上下文谓词

第三个参数是上下文谓词字符串：

```rust
// 只在 Editor 上下文中生效
KeyBinding::new("ctrl-s", Save, Some("Editor"));

// 组合条件：Editor 且 mode == normal
KeyBinding::new("dd", DeleteLine, Some("Editor && mode == normal"));

// 在 Editor 子元素中生效
KeyBinding::new("ctrl-p", Find, Some("Editor > SearchBar"));
```

上下文谓词语法见第32章。

## 31.2 注册键绑定

### 31.2.1 `cx.bind_keys()` 添加绑定

```rust
use gpui::KeyBinding;

// 在应用启动或窗口创建时注册
cx.bind_keys(vec![
    KeyBinding::new("ctrl-n", NewFile, None),
    KeyBinding::new("ctrl-o", OpenFile, None),
    KeyBinding::new("ctrl-s", SaveFile, None),
]);
```

### 31.2.2 批量注册

```rust
fn register_editor_keybindings(cx: &mut App) {
    use gpui::KeyBinding;

    cx.bind_keys(vec![
        // 光标移动
        KeyBinding::new("up", MoveUp, Some("Editor")),
        KeyBinding::new("down", MoveDown, Some("Editor")),
        KeyBinding::new("left", MoveLeft, Some("Editor")),
        KeyBinding::new("right", MoveRight, Some("Editor")),

        // 编辑操作
        KeyBinding::new("enter", Newline, Some("Editor")),
        KeyBinding::new("backspace", Backspace, Some("Editor")),
        KeyBinding::new("delete", Delete, Some("Editor")),

        // 选择
        KeyBinding::new("shift-up", SelectUp, Some("Editor")),
        KeyBinding::new("shift-down", SelectDown, Some("Editor")),
        KeyBinding::new("ctrl-a", SelectAll, Some("Editor")),
    ]);
}
```

### 31.2.3 覆盖与优先级

后注册的绑定优先级更高：

```rust
// 基础绑定
cx.bind_keys(vec![
    KeyBinding::new("ctrl-p", Find, None),
]);

// 用户自定义覆盖
cx.bind_keys(vec![
    KeyBinding::new("ctrl-p", PreviousResult, Some("SearchActive")),
]);
```

`NoAction` 可以禁用特定上下文中的快捷键：

```rust
cx.bind_keys(vec![
    // 在 Terminal 中禁用 ctrl-f（避免与终端搜索冲突）
    KeyBinding::new("ctrl-f", NoAction, Some("Terminal")),
]);
```

## 31.3 键绑定语法

### 31.3.1 修饰符

| 修饰符 | 说明 |
|--------|------|
| `cmd` | macOS Command |
| `ctrl` | Control |
| `alt` | Alt/Option |
| `shift` | Shift |

```rust
KeyBinding::new("cmd-shift-k", DeleteLine, None);       // macOS: Cmd+Shift+K
KeyBinding::new("ctrl-shift-k", DeleteLine, None);      // Linux/Windows: Ctrl+Shift+K
KeyBinding::new("alt-shift-f", Format, None);           // Alt+Shift+F
```

### 31.3.2 键名

常用键名：

| 键名 | 说明 |
|------|------|
| `a`-`z` | 字母键 |
| `0`-`9` | 数字键 |
| `f1`-`f24` | 功能键 |
| `enter` | 回车 |
| `escape` | Esc |
| `space` | 空格 |
| `tab` | Tab |
| `backspace` | 退格 |
| `delete` | Delete |
| `left`/`right`/`up`/`down` | 方向键 |
| `home`/`end` | Home/End |
| `pageup`/`pagedown` | Page Up/Down |
| `[` `]` `/` `\` `;` `'` `,` `.` `-` `=` | 符号键 |

### 31.3.3 组合键与序列

多个键用空格分隔，表示键序列：

```rust
// Vim 风格：先按 g 再按 g 回到顶部
KeyBinding::new("g g", MoveToTop, Some("VimMode == normal"));

// 先按 leader 再按具体键
KeyBinding::new("ctrl-k s", SaveAll, None);
KeyBinding::new("ctrl-k ctrl-f", FormatFile, None);
```

序列匹配过程：
1. 用户按下第一个键，系统等待后续键
2. 用户在超时时间内按下后续键，匹配成功
3. 超时或按下不匹配的键，序列重置

## 31.4 动态键绑定

### 31.4.1 运行时修改绑定

`cx.bind_keys()` 可以在运行时多次调用：

```rust
// 应用启动时注册默认快捷键
fn init_default_keymap(cx: &mut App) {
    cx.bind_keys(vec![
        KeyBinding::new("ctrl-s", Save, None),
        KeyBinding::new("ctrl-z", Undo, None),
    ]);
}

// 用户修改快捷键后重新注册
fn update_keybinding(old: &str, new: &str, action: impl Action, cx: &mut App) {
    // 旧绑定不会被移除，新绑定因优先级更高而覆盖旧绑定
    cx.bind_keys(vec![
        KeyBinding::new(&new, action, None),
    ]);
}
```

### 31.4.2 用户自定义快捷键

从配置文件加载快捷键：

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct UserKeyBinding {
    keystrokes: String,
    action: String,
    context: Option<String>,
}

fn load_user_keymap(path: &str, cx: &mut App) -> anyhow::Result<()> {
    let config = std::fs::read_to_string(path)?;
    let bindings: Vec<UserKeyBinding> = serde_json::from_str(&config)?;

    let keybindings: Vec<KeyBinding> = bindings
        .into_iter()
        .filter_map(|b| {
            // 根据 action 名称查找并构建 Action
            let action = build_action_by_name(&b.action)?;
            Some(KeyBinding::new(&b.keystrokes, action, b.context.as_deref()))
        })
        .collect();

    cx.bind_keys(keybindings);
    Ok(())
}
```

配置文件示例 (`keymap.json`)：

```json
[
  { "keystrokes": "ctrl-shift-s", "action": "editor::SaveAll" },
  { "keystrokes": "ctrl-shift-p", "action": "workspace::CommandPalette" },
  { "keystrokes": "ctrl-k ctrl-s", "action": "editor::SaveAll", "context": "Editor" }
]
```

## 31.5 实战：完整的快捷键系统

```rust
use gpui::{actions, KeyBinding, Context, Window, div, InteractiveElement, IntoElement};

// 1. 定义 Action
actions!(app, [
    NewFile, OpenFile, SaveFile, SaveAll,
    Undo, Redo,
    Quit,
    ToggleSidebar,
    ToggleTerminal,
]);

actions!(editor, [
    MoveUp, MoveDown, MoveLeft, MoveRight,
    SelectUp, SelectDown,
    SelectAll, DeleteLine, DeleteWord,
    Newline, Backspace, Delete,
    Find, Replace,
]);

// 2. 注册全局快捷键
fn register_keybindings(cx: &mut gpui::App) {
    cx.bind_keys(vec![
        // 文件操作
        KeyBinding::new("ctrl-n", NewFile, None),
        KeyBinding::new("ctrl-o", OpenFile, None),
        KeyBinding::new("ctrl-s", SaveFile, Some("Editor")),
        KeyBinding::new("ctrl-shift-s", SaveAll, None),

        // 编辑
        KeyBinding::new("ctrl-z", Undo, Some("Editor")),
        KeyBinding::new("ctrl-shift-z", Redo, Some("Editor")),

        // 编辑器内
        KeyBinding::new("up", MoveUp, Some("Editor")),
        KeyBinding::new("down", MoveDown, Some("Editor")),
        KeyBinding::new("left", MoveLeft, Some("Editor")),
        KeyBinding::new("right", MoveRight, Some("Editor")),
        KeyBinding::new("shift-up", SelectUp, Some("Editor")),
        KeyBinding::new("shift-down", SelectDown, Some("Editor")),
        KeyBinding::new("ctrl-a", SelectAll, Some("Editor")),
        KeyBinding::new("ctrl-f", Find, Some("Editor")),

        // 窗口
        KeyBinding::new("ctrl-b", ToggleSidebar, None),
        KeyBinding::new("ctrl-j", ToggleTerminal, None),

        // 退出
        KeyBinding::new("ctrl-q", Quit, None),
    ]);
}

// 3. 在 View 中处理 Action
struct EditorView {
    content: String,
    sidebar_visible: bool,
}

impl Render for EditorView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("editor-view")
            .size_full()
            .key_context("Editor")
            // 注册 Action handler
            .on_action(cx.listener(|this, _: &SaveFile, _, cx| {
                this.save();
                cx.notify();
            }))
            .on_action(cx.listener(|this, _: &Undo, _, cx| {
                this.undo();
                cx.notify();
            }))
            .on_action(cx.listener(|this, _: &MoveUp, _, cx| {
                this.move_cursor_up();
                cx.notify();
            }))
            .child(&this.content)
    }
}
```
