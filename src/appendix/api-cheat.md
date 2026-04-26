# 附录 B：常用 API 速查

本附录以表格形式汇总 GPUI 最常用的 API，便于开发时快速查阅。

## Entity 操作速查

| 方法 | 签名 | 说明 |
|------|------|------|
| `cx.new()` | `\|cx\| T -> Entity<T>` | 创建新的 Entity |
| `entity.read(cx)` | `&mut App -> &T` | 只读访问 Entity 数据 |
| `entity.update(cx, \|this, cx\| ...)` | `&mut App -> R` | 可变访问 Entity 数据 |
| `entity.downgrade()` | `() -> WeakEntity<T>` | 创建弱引用 |
| `weak_entity.upgrade()` | `&App -> Option<Entity<T>>` | 从弱引用提升 |
| `entity.entity_id()` | `() -> EntityId` | 获取实体 ID |
| `entity.into_any()` | `() -> AnyEntity` | 类型擦除 |
| `cx.reserve_entity()` | `() -> Reservation<T>` | 预留 Entity slot |

```rust
// 创建
let entity: Entity<Counter> = cx.new(|cx| Counter { count: 0 });

// 只读读取
let count = entity.read(cx).count;

// 可变更新
entity.update(cx, |this, cx| {
    this.count += 1;
    cx.notify();  // View 的 Entity 需要手动 notify
});

// 弱引用（用于闭包捕获，避免循环引用）
let weak = entity.downgrade();
cx.spawn(move |this, mut cx| async move {
    if let Some(entity) = weak.upgrade(&cx) {
        // Entity 仍然存活
    }
}).detach();
```

---

## Context 方法速查（按类型）

### App 级别方法

| 方法 | 说明 |
|------|------|
| `cx.open_window(opts, build)` | 打开新窗口 |
| `cx.update_window(handle, \|w, cx\| ...)` | 更新指定窗口 |
| `cx.close_window(handle)` | 关闭窗口 |
| `cx.bind_keys(vec![...])` | 注册全局键绑定 |
| `cx.set_global(T)` | 设置全局状态 |
| `cx.global::<T>()` | 读取全局状态 |
| `cx.try_global::<T>()` | 安全读取全局状态 |
| `cx.update_global::<T, R>(\|this, cx\| ...)` | 更新全局状态 |
| `cx.observe_global::<T>(\|cx\| ...)` | 监听全局状态变化 |
| `cx.on_quit(\|cx\| ...)` | 注册退出回调 |
| `cx.quit()` | 退出应用 |
| `cx.spawn(\|cx\| async move { ... })` | 启动前台异步任务 |
| `cx.background_spawn(async move { ... })` | 启动后台异步任务 |
| `cx.foreground_executor()` | 获取前台执行器 |
| `cx.background_executor()` | 获取后台执行器 |
| `cx.push_notification(msg)` | 推送系统通知 |
| `cx.open(path)` | 打开文件选择器 |
| `cx.save_file_dialog(opts)` | 打开保存对话框 |
| `cx.open_url(url)` | 打开 URL |
| `cx.write_to_clipboard(item)` | 写入剪贴板 |
| `cx.read_from_clipboard()` | 读取剪贴板 |

### Context\<T\>（Entity 专用上下文）

| 方法 | 说明 |
|------|------|
| `cx.notify()` | 标记当前 Entity 需要重新渲染 |
| `cx.observe(entity, \|this, entity, cx\| ...)` | 观察其他 Entity |
| `cx.observe_self(\|this, cx\| ...)` | 自观察 |
| `cx.subscribe(entity, \|this, entity, event, cx\| ...)` | 订阅其他 Entity 的事件 |
| `cx.emit(Event::Something)` | 发射事件（需实现 `EventEmitter`） |
| `cx.entity()` | 获取自身的 Entity 句柄 |
| `cx.focus_handle()` | 创建焦点句柄 |
| `cx.focused()` | 获取当前焦点 |
| `cx.focus(&handle)` | 设置焦点 |
| `cx.blur()` | 移除焦点 |
| `cx.spawn(\|entity, cx\| async move { ... })` | 从 Entity 启动异步任务 |
| `cx.refresh()` | 强制刷新窗口 |
| `cx.request_animation_frame()` | 安排下一帧渲染 |

---

## Styled 方法速查

### 布局

| 方法 | 说明 |
|------|------|
| `.flex()` | 激活 flexbox |
| `.flex_row()` / `.flex_col()` | 主轴方向 |
| `.flex_wrap()` / `.flex_nowrap()` | 换行 |
| `.items_center/start/end/stretch` | 交叉轴对齐 |
| `.justify_center/start/end/between/around/evenly` | 主轴分布 |
| `.self_center/start/end/stretch` | 单元素对齐 |
| `.gap(n)` / `.gap_x(n)` / `.gap_y(n)` | 间距 |
| `.grid()` | 激活网格布局 |
| `.grid_cols([...])` / `.grid_rows([...])` | 定义轨道 |
| `.col_span(n)` / `.row_span(n)` | 跨列/跨行 |

### 尺寸

| 方法 | 说明 |
|------|------|
| `.size_full()` | 宽高 100% |
| `.w(n)` / `.h(n)` | 固定宽高 |
| `.min_w(n)` / `.min_h(n)` | 最小宽高 |
| `.max_w(n)` / `.max_h(n)` | 最大宽高 |
| `.flex_1()` | flex-grow: 1 |
| `.flex_grow()` | 允许增长 |
| `.flex_shrink()` | 允许收缩 |
| `.flex_basis(n)` | 初始尺寸 |

### 间距

| 方法 | 说明 |
|------|------|
| `.p(n)` | 四面 padding |
| `.px(n)` / `.py(n)` | 水平/垂直 padding |
| `.pt/pr/pb/pl(n)` | 单面 padding |
| `.m(n)` | 四面 margin |
| `.mx(n)` / `.my(n)` | 水平/垂直 margin |
| `.mt/mr/mb/ml(n)` | 单面 margin |

### 颜色

| 方法 | 说明 |
|------|------|
| `.bg(color)` | 背景色 |
| `.text_color(color)` | 文字颜色 |
| `.border_color(color)` | 边框颜色 |
| `rgb(0xRRGGBB)` | RGB 颜色构造 |
| `rgba(0xRRGGBBAA)` | RGBA 颜色构造 |
| `hsla(h, s, l, a)` | HSLA 颜色构造 |

### 边框与圆角

| 方法 | 说明 |
|------|------|
| `.border_1()` ~ `.border_4()` | 边框宽度快捷方法 |
| `.border_t_1()` / `.border_b_1()` | 上/下边框 |
| `.rounded_sm/md/lg/xl/full` | 圆角预设 |
| `.border_solid/dashed/dotted` | 边框样式 |

### 文字

| 方法 | 说明 |
|------|------|
| `.text_size(px(n))` | 字号 |
| `.text_xs/sm/base/lg/xl/2xl` | 预设字号 |
| `.font_weight(FontWeight::BOLD)` | 字重 |
| `.font_family("monospace")` | 字体 |
| `.line_height(px(n))` | 行高 |
| `.text_left/center/right` | 对齐 |
| `.underline()` / `.strikethrough()` | 文字装饰 |
| `.truncate()` | 截断显示省略号 |

### 溢出与可见性

| 方法 | 说明 |
|------|------|
| `.overflow_hidden()` | 裁剪超出内容 |
| `.overflow_scroll()` | 允许滚动 |
| `.overflow_x_scroll()` / `.overflow_y_scroll()` | 单向滚动 |
| `.visible()` / `.invisible()` / `.hidden()` | 可见性 |

---

## Div 事件注册速查

### 鼠标事件

| 方法 | 事件类型 | 说明 |
|------|---------|------|
| `.on_click(\|_, _, cx\| ...)` | `ClickEvent` | 点击 |
| `.on_mouse_down(button, \|_, _, cx\| ...)` | `MouseDownEvent` | 鼠标按下 |
| `.on_mouse_up(button, \|_, _, cx\| ...)` | `MouseUpEvent` | 鼠标释放 |
| `.on_hover(\|_, _, cx\| ...)` | `HoverEvent` | 悬停 |
| `.on_mouse_move(\|_, _, cx\| ...)` | `MouseMoveEvent` | 鼠标移动 |
| `.on_scroll_wheel(\|_, _, cx\| ...)` | `ScrollWheelEvent` | 滚轮滚动 |

### 键盘事件

| 方法 | 事件类型 | 说明 |
|------|---------|------|
| `.on_key_down(\|_, _, cx\| ...)` | `KeyDownEvent` | 按键按下 |
| `.on_key_up(\|_, _, cx\| ...)` | `KeyUpEvent` | 按键释放 |
| `.on_modifiers_changed(...)` | `ModifiersChangedEvent` | 修饰符变化 |

### Action 事件

| 方法 | 说明 |
|------|------|
| `.on_action(\|action: &A, _, cx\| ...)` | 注册指定 Action 的处理器 |

### 拖拽事件

| 方法 | 说明 |
|------|------|
| `.on_drag(value, \|_, _, cx\| tooltip)` | 开始拖拽 |
| `.on_drop(\|_, _, cx\| ...)` | 放置目标 |
| `.on_drag_move(\|event, _, cx\| ...)` | 拖拽移动中 |

### 手势事件

| 方法 | 事件类型 | 说明 |
|------|---------|------|
| `.on_pinch(\|_, _, cx\| ...)` | `PinchEvent` | 捏合缩放 |

### 焦点

| 方法 | 说明 |
|------|------|
| `.track_focus(&handle)` | 跟踪焦点状态 |
| `.on_focus(\|_, _, cx\| ...)` | 获得焦点 |
| `.on_blur(\|_, _, cx\| ...)` | 失去焦点 |

### Tooltip

| 方法 | 说明 |
|------|------|
| `.tooltip(\|_, cx\| AnyView)` | 设置悬停提示 |

### 条件与变换

| 方法 | 说明 |
|------|------|
| `.when(cond, \|div\| div.child(...))` | 条件应用 |
| `.when_some(opt, \|div, val\| ...)` | Option 条件应用 |
| `.id(element_id)` | 设置元素 ID（用于事件和缓存） |
| `.group(name)` | 设置组名（用于子元素样式联动） |
| `.key_context(ctx)` | 设置键盘上下文 |

---

## 动画与缓动函数速查

### 创建动画

```rust
use std::time::Duration;
use gpui::*;

// 基本动画
let anim = Animation::new(Duration::from_millis(300));

// 带缓动
let anim = Animation::new(Duration::from_millis(500))
    .with_easing(ease_in_out);

// 循环动画
let anim = Animation::new(Duration::from_secs(2))
    .repeat()
    .with_easing(|t| t);
```

### 应用动画

```rust
// 单元素动画
div()
    .with_animation(
        "fade-in",
        Animation::new(Duration::from_millis(300))
            .with_easing(ease_out_quint),
        |element, progress| {
            element.opacity(progress)
        },
    )

// 多动画链
div()
    .with_animations(
        "slide-fade",
        vec![
            Animation::new(Duration::from_millis(200)),
            Animation::new(Duration::from_millis(400)),
        ],
        |element, animation_index, progress| {
            match animation_index {
                0 => element.opacity(progress),
                1 => element.translate_x(px(100. * progress)),
                _ => element,
            }
        },
    )
```

### 内置缓动函数

| 函数 | 公式 | 效果 |
|------|------|------|
| `linear` | `t` | 匀速 |
| `quadratic` | `t * t` | 二次加速（越来越快） |
| `ease_in_out` | 分段函数 | 先加速后减速 |
| `bounce(easing)` | 反弹效果 | 弹性运动 |
| `pulsating_between(min, max)` | 正弦函数 | 脉冲/呼吸效果 |

### 自定义缓动函数

```rust
// 三次缓入缓出
fn cubic_bezier(t: f32) -> f32 {
    if t < 0.5 {
        4.0 * t * t * t
    } else {
        1.0 - (-2.0 * t + 2.0).powi(3) / 2.0
    }
}

let anim = Animation::new(Duration::from_millis(500))
    .with_easing(cubic_bezier);
```

---

## 列表速查

### ListState

| 方法 | 说明 |
|------|------|
| `ListState::new(count, alignment, overdraw)` | 创建列表状态 |
| `state.splice(range, new_count)` | 数据变更通知 |
| `state.reset(count)` | 重置全部数据 |
| `state.scroll_to_item(index, strategy)` | 滚动到指定项 |
| `state.on_scroll(\|event, window, cx\| ...)` | 设置滚动回调 |

### UniformList

| 方法 | 说明 |
|------|------|
| `uniform_list(id, count, \|range, window, cx\| vec)` | 创建统一高度列表 |
| `UniformListScrollHandle::new()` | 创建滚动句柄 |
| `scroll_handle.scroll_to_item(index, strategy)` | 滚动到项 |

### ScrollStrategy

| 值 | 说明 |
|----|------|
| `Top` | 滚动到项的顶部 |
| `Center` | 滚动到项的中间 |
| `Bottom` | 滚动到项的底部 |
| `Nearest` | 滚动到最近的可见位置 |

---

## 图片速查

### ImageSource (`From` trait 实现)

`ImageSource` 不通过构造方法创建，而是通过 `From` trait 自动转换：

| 转换来源 | 示例 | 说明 |
|---------|------|------|
| `From<&str>` | `img("https://example.com/img.png")` | URI 字符串或编译时资源路径 |
| `From<&Path>` / `From<PathBuf>` | `img(&PathBuf::from("/tmp/img.png"))` | 本地文件系统路径 |
| `From<Arc<RenderImage>>` | `img(Arc::new(render_image))` | 直接传入渲染后的图像数据 |
| `From<F>` (自定义加载函数) | `img(|_cx| async { Ok(image) })` | 自定义异步加载逻辑 |

```rust
// 最常见的用法：直接传字符串
img("path/to/image.png")                    // 相对资源路径
img("https://example.com/image.png")        // 网络 URL

// 从文件系统路径
img(&PathBuf::from("/absolute/path/img.png"))

// 从 RenderImage 数据
let image: Arc<RenderImage> = ...;
img(image)

// 自定义加载
img(|cx| async move {
    // 自定义加载逻辑，返回 Arc<RenderImage>
    Ok(loaded_image)
})
```

### ObjectFit 模式

| 值 | 说明 |
|----|------|
| `ObjectFit::Fill` | 拉伸填满容器 |
| `ObjectFit::Contain` | 保持比例完整显示 |
| `ObjectFit::Cover` | 保持比例填满容器（裁剪） |
| `ObjectFit::ScaleDown` | 不放大，小图不缩放 |
| `ObjectFit::None` | 原始尺寸 |

### 图片样式

| 方法 | 说明 |
|------|------|
| `.grayscale()` | 灰度滤镜 |
| `.saturate(factor)` | 饱和度调整 |
| `.hue_rotate(degrees)` | 色相旋转 |
