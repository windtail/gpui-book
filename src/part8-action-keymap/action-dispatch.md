# 第33章：Action 分发

## 33.1 `on_action()` 注册

### 33.1.1 在元素上注册

`.on_action()` 在元素上注册 Action 处理器：

```rust
use gpui::{actions, div, InteractiveElement, Context, Window, IntoElement};

actions!(editor, [Save, Undo, Redo, DeleteLine]);

struct EditorView {
    content: String,
    saved: bool,
}

impl Render for EditorView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("editor")
            .size_full()
            .key_context("Editor")
            .on_action(cx.listener(|this, _: &Save, _, cx| {
                this.saved = true;
                println!("文件已保存");
                cx.notify();
            }))
            .on_action(cx.listener(|this, _: &Undo, _, cx| {
                this.undo();
                cx.notify();
            }))
            .on_action(cx.listener(|this, _: &DeleteLine, _, cx| {
                this.delete_current_line();
                cx.notify();
            }))
            .child(&this.content)
    }
}
```

### 33.1.2 Handler 签名

`on_action` 的 handler 签名：

```rust
fn on_action<A: Action>(
    &mut self,
    listener: impl Fn(&A, &mut Window, &mut App) + 'static,
)
```

- `&A` — Action 实例的引用（类型安全的）
- `&mut Window` — 窗口上下文
- `&mut App` — 应用上下文

使用 `cx.listener()` 获取对 View 的可变引用：

```rust
.on_action(cx.listener(|this: &mut EditorView, action: &Save, window: &mut Window, cx: &mut App| {
    // this: &mut EditorView — 可修改 View 状态
    // action: &Save — Action 实例
    // window: &mut Window — 窗口操作
    // cx: &mut App — 应用操作
}))
```

### 33.1.3 多 handler 优先级

同一 Action 可以在多个元素上注册。事件分发时：

1. GPUI 先通过 Keymap 将按键解析为 Action
2. 然后从焦点元素向根搜索，找到第一个注册了该 Action handler 的元素
3. 调用该 handler

```rust
div()
    .id("workspace")
    .on_action(cx.listener(|_, _: &Save, _, _| {
        // 工作区级 Save handler
        println!("workspace save");
    }))
    .child(
        div()
            .id("editor")
            .key_context("Editor")
            .on_action(cx.listener(|_, _: &Save, _, _| {
                // 编辑器级 Save handler
                // 当焦点在 Editor 内时，这个优先于 workspace handler
                println!("editor save");
            }))
    )
```

焦点在 `editor` 内时，按 `ctrl-s` 触发 "editor save"。焦点不在 `editor` 内时，触发 "workspace save"。

## 33.2 Action 解析流程

### 33.2.1 按键 → Keystroke → Action

完整流程：

```
1. 用户按下键
2. 平台层生成 KeyEvent
3. GPUI 将 KeyEvent 转为 Keystroke
4. 收集当前元素的 KeyContext 栈
5. 在 Keymap 中查找匹配的 KeyBinding
6. KeyBinding 返回 Action
7. 从焦点元素向根查找 Action handler
8. 调用 handler
```

```rust
// 示例：用户在 Editor 内按 ctrl-s

// Step 1-3: 平台 → Keystroke
// keystroke = "ctrl-s"

// Step 4: KeyContext 栈
// ["os=macos Workspace", "Pane", "Editor"]

// Step 5: Keymap 查找
// KeyBinding::new("ctrl-s", Save, Some("Editor")) → 匹配

// Step 6: 得到 Action
// action = Save

// Step 7: 从焦点向根查找 handler
// Editor 上有 on_action(Save) → 找到

// Step 8: 调用
// handler(&Save, window, cx)
```

### 33.2.2 绑定冲突解决

多个绑定匹配时，按以下优先级决定：

1. **上下文深度**：深层上下文优先
2. **注册顺序**：同一深度，后注册优先
3. **精确匹配**：完全匹配的 keystroke 优先于序列前缀

```rust
// 注册顺序影响结果
cx.bind_keys(vec![
    // 先注册：优先级低
    KeyBinding::new("ctrl-s", Save, None),
]);

cx.bind_keys(vec![
    // 后注册：优先级高
    KeyBinding::new("ctrl-s", SaveAs, Some("Editor")),
]);

// 在 Editor 上下文中，ctrl-s 触发 SaveAs
// 在其他上下文中，ctrl-s 触发 Save
```

### 33.2.3 上下文过滤

`KeyBinding` 的上下文谓词决定它是否在当前上下文中生效：

```rust
cx.bind_keys(vec![
    // 只在 Terminal 中生效
    KeyBinding::new("ctrl-shift-c", CopyPath, Some("Terminal")),

    // 只在 Editor 中生效
    KeyBinding::new("ctrl-shift-c", CopyRelativePath, Some("Editor")),

    // 全局生效
    KeyBinding::new("ctrl-q", Quit, None),
]);
```

没有匹配的 Action 时，按键不产生任何操作。

## 33.3 编程式触发

### 33.3.1 `cx.dispatch_action()`

不通过按键，直接触发 Action：

```rust
// 在 View 中通过代码触发
impl EditorView {
    fn save_via_menu(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        window.dispatch_action(Box::new(Save), cx);
    }
}

// 在按钮点击中触发
div()
    .id("save-button")
    .on_click(cx.listener(|this, _, window, cx| {
        // 通过 dispatch_action 触发 Save
        window.dispatch_action(Box::new(Save), cx);
    }))
    .child("保存")
```

### 33.3.2 从非 UI 代码触发 Action

从异步任务中触发 Action：

```rust
impl EditorView {
    fn load_file(&mut self, path: PathBuf, window: &mut Window, cx: &mut Context<Self>) {
        cx.spawn(|view, mut cx| async move {
            let content = tokio::fs::read_to_string(&path).await?;

            view.update(&mut cx, |this, cx| {
                this.content = content;
                cx.notify();

                // 加载完成后触发格式化
                window.dispatch_action(Box::new(Format), cx);
            })
        }).detach();
    }
}
```

从其他 View 触发 Action：

```rust
impl CommandPalette {
    fn execute_selected(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        if let Some(action) = &self.selected_action {
            // 将 Action 发送到当前焦点
            window.dispatch_action(action.boxed_clone(), cx);
            self.dismiss(cx);
        }
    }
}
```

## 33.4 Action 测试

### 33.4.1 模拟按键输入

在测试中模拟按键：

```rust
use gpui::{actions, KeyBinding, TestAppContext, TestDispatcher};

actions!(editor, [Save, Undo]);

#[gpui::test]
fn test_keybindings_dispatch_action(cx: &mut TestAppContext) {
    // 注册快捷键
    cx.update(|cx| {
        cx.bind_keys(vec![
            KeyBinding::new("ctrl-s", Save, Some("Editor")),
            KeyBinding::new("ctrl-z", Undo, Some("Editor")),
        ]);
    });

    // 创建窗口和 View
    let window = cx.update(|cx| {
        cx.open_window(Default::default(), |window, cx| {
            cx.new(|cx| EditorView {
                content: String::new(),
                save_count: 0,
                undo_count: 0,
            })
        })
        .unwrap()
    });

    // 设置焦点
    window.update(cx, |view, window, cx| {
        window.focus(&view.focus_handle, cx);
    }).unwrap();

    // 模拟按键
    cx.dispatch_keystroke(window, gpui::Keystroke::parse("ctrl-s").unwrap());

    // 验证
    window.update(cx, |view, _, _| {
        assert_eq!(view.save_count, 1);
    }).unwrap();
}
```

### 33.4.2 验证 Action 被分发

```rust
#[gpui::test]
fn test_action_dispatch(cx: &mut TestAppContext) {
    let window = cx.update(|cx| {
        cx.open_window(Default::default(), |window, cx| {
            cx.new(|cx| {
                let handle = cx.focus_handle();
                TestView {
                    save_received: false,
                    focus_handle: handle,
                }
            })
        })
        .unwrap()
    });

    // 直接 dispatch_action
    window.update(cx, |_, window, cx| {
        window.dispatch_action(Box::new(Save), cx);
    }).unwrap();

    // 验证 handler 被调用
    window.update(cx, |view, _, _| {
        assert!(view.save_received);
    }).unwrap();
}

struct TestView {
    save_received: bool,
    focus_handle: FocusHandle,
}

impl Render for TestView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("test")
            .key_context("Editor")
            .track_focus(&self.focus_handle)
            .on_action(cx.listener(|this, _: &Save, _, _| {
                this.save_received = true;
            }))
            .child("test")
    }
}
```
