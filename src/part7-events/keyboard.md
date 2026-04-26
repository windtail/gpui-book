# 第27章：键盘事件

## 27.1 `KeyDownEvent` 与 `KeyUpEvent`

```rust
use gpui::{
    div, KeyDownEvent, KeyUpEvent,
    InteractiveElement, ParentElement, IntoElement, Context, Window,
};

struct KeyEventLogger {
    last_key: String,
    key_down_count: usize,
    key_up_count: usize,
}

impl Render for KeyEventLogger {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("key-logger")
            .size_full()
            .on_key_down(cx.listener(|this, event: &KeyDownEvent, _, cx| {
                // event.keystroke: Keystroke — 解析后的按键
                // event.is_held: bool — 是否长按
                // event.prefer_character_input: bool — 是否优先字符输入

                this.last_key = event.keystroke.to_string();
                this.key_down_count += 1;
                cx.notify();
            }))
            .on_key_up(cx.listener(|this, event: &KeyUpEvent, _, cx| {
                // event.keystroke: Keystroke
                this.key_up_count += 1;
                cx.notify();
            }))
            .child(format!(
                "last: {}, down: {}, up: {}",
                this.last_key, this.key_down_count, this.key_up_count
            ))
    }
}
```

`KeyDownEvent` 包含三个字段：

```rust
pub struct KeyDownEvent {
    pub keystroke: Keystroke,
    pub is_held: bool,
    pub prefer_character_input: bool,
}

pub struct KeyUpEvent {
    pub keystroke: Keystroke,
}
```

`is_held` 在长按时为 `true`。`prefer_character_input` 用于 AltGr 等场景，表示应优先处理字符输入而非快捷键。

## 27.2 `Keystroke` 解析

### 27.2.1 键码表示

`Keystroke` 是键盘事件的统一表示：

```rust
use gpui::Keystroke;

// 解析字符串
let keystroke = Keystroke::parse("ctrl-a").unwrap();
let keystroke = Keystroke::parse("cmd-shift-p").unwrap();
let keystroke = Keystroke::parse("alt-enter").unwrap();

// Keystroke 显示
println!("{}", keystroke); // "ctrl-a"
```

`Keystroke` 由修饰键和主键组成。主键可以是：
- 字母：`a` ~ `z`
- 数字：`0` ~ `9`
- 功能键：`f1` ~ `f24`
- 特殊键：`enter`, `escape`, `space`, `tab`, `backspace`, `delete`, `insert`
- 导航：`left`, `right`, `up`, `down`, `home`, `end`, `pageup`, `pagedown`
- 其他：`shift`, `ctrl`, `alt`, `cmd`, `capslock`

### 27.2.2 修饰符组合

修饰键通过 `Modifiers` 结构表示：

```rust
pub struct Modifiers {
    pub control: bool,
    pub shift: bool,
    pub alt: bool,
    pub command: bool,    // macOS 的 cmd，其他平台为 false
    pub function: bool,   // macOS 的 fn
    pub caps_lock: bool,
}
```

`Keystroke` 内部持有 `Modifiers` 和主键：

```rust
// 手动构建 Keystroke
use gpui::{Keystroke, Modifiers};

let keystroke = Keystroke {
    modifiers: Modifiers {
        control: true,
        shift: false,
        alt: false,
        command: false,
        function: false,
        caps_lock: false,
    },
    key: "a".into(),
    ime_key: None,
};

// 通常使用 parse
let keystroke = Keystroke::parse("ctrl-a").unwrap();
```

`Keystroke` 的 `Display` 实现会格式化为可读字符串：

```rust
let k = Keystroke::parse("ctrl-shift-enter").unwrap();
assert_eq!(k.to_string(), "ctrl-shift-enter");
```

## 27.3 `ModifiersChangedEvent`

### 27.3.1 `Modifiers` 状态

修饰键变化时触发 `ModifiersChangedEvent`：

```rust
use gpui::ModifiersChangedEvent;

div()
    .id("modifier-tracker")
    .on_modifiers_changed(cx.listener(|this, event: &ModifiersChangedEvent, _, cx| {
        // event.modifiers: Modifiers
        // event.capslock: Capslock

        if event.modifiers.control {
            this.ctrl_held = true;
        } else {
            this.ctrl_held = false;
        }

        if event.modifiers.shift {
            this.shift_held = true;
        } else {
            this.shift_held = false;
        }

        cx.notify();
    }))
```

### 27.3.2 修饰符追踪

`ModifiersChangedEvent` 实现了 `Deref<Target = Modifiers>`，可以直接访问修饰符：

```rust
// event 可以直接当作 Modifiers 使用
div()
    .id("mod-tracker")
    .on_modifiers_changed(cx.listener(|this, event: &ModifiersChangedEvent, _, cx| {
        // 通过 Deref 直接访问
        let ctrl = event.control;
        let shift = event.shift;
        let alt = event.alt;
        let cmd = event.command;

        this.modifier_state = format!(
            "ctrl={} shift={} alt={} cmd={}",
            ctrl, shift, alt, cmd
        );
        cx.notify();
    }))
```

`ModifiersChangedEvent` 也包含 `capslock` 字段：

```rust
pub struct ModifiersChangedEvent {
    pub modifiers: Modifiers,
    pub capslock: Capslock,
}
```

## 27.4 键盘事件注册

### 27.4.1 `.on_key_down()` / `.on_key_up()`

```rust
struct Editor {
    content: String,
}

impl Render for Editor {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("editor")
            .size_full()
            .on_key_down(cx.listener(|this, event: &KeyDownEvent, _, cx| {
                let k = &event.keystroke;

                // 直接处理某些快捷键
                if k.to_string() == "ctrl-s" {
                    this.save();
                    cx.stop_propagation();
                    cx.prevent_default();
                    return;
                }

                if k.to_string() == "escape" {
                    this.clear_selection();
                    cx.stop_propagation();
                    cx.prevent_default();
                    return;
                }

                // 其他键交给 Action 系统处理
            }))
            .child(&this.content)
    }
}
```

### 27.4.2 `.on_modifiers_changed()`

```rust
struct StatusBar {
    modifier_text: String,
}

impl Render for StatusBar {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("status-bar")
            .on_modifiers_changed(cx.listener(|this, event: &ModifiersChangedEvent, _, cx| {
                let mods = &event.modifiers;
                let mut parts = vec![];
                if mods.control { parts.push("ctrl"); }
                if mods.shift { parts.push("shift"); }
                if mods.alt { parts.push("alt"); }
                if mods.command { parts.push("cmd"); }
                this.modifier_text = parts.join(" + ");
                cx.notify();
            }))
            .child(&this.modifier_text)
    }
}
```

## 27.5 键盘事件与 Action 的关系

键盘事件有两个处理层级：

1. **低级层**：`.on_key_down()` 直接接收 `KeyDownEvent`，适合需要精确控制键行为的场景。
2. **高级层**：Action + Keymap 系统，将按键映射为语义化操作。

推荐使用 Action 系统处理快捷键。直接在 `.on_key_down()` 中处理只适用于：
- 全局快捷键拦截
- 特殊输入处理（如 IME 相关）
- 需要 `stop_propagation()` 阻止事件传播的场景

```rust
// 推荐方式：用 Action 系统处理快捷键
// 1. 定义 Action
actions!(editor, [Save, Undo, Redo]);

// 2. 注册 KeyBinding
cx.bind_keys(vec![
    KeyBinding::new("ctrl-s", Save, None),
    KeyBinding::new("ctrl-z", Undo, None),
    KeyBinding::new("ctrl-shift-z", Redo, None),
]);

// 3. 在元素上注册 action handler
div()
    .id("editor")
    .key_context("Editor")
    .on_action(cx.listener(|this, _: &Save, _, cx| {
        this.save();
        cx.notify();
    }))
```

Action 系统的优势：
- 用户可自定义快捷键
- 上下文感知的键绑定
- 统一的快捷键显示（菜单、帮助界面）
- 跨平台键等效处理
