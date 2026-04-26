# 第 47 章：动画系统

## 47.1 `Animation` struct

### 47.1.1 `duration` 时长

`Animation` 定义一段随时间变化的插值：

```rust
use gpui::Animation;
use std::time::Duration;

let anim = Animation::new(Duration::from_millis(300));
// 300ms 的动画，默认线性缓动
```

### 47.1.2 `oneshot` 单次播放

`oneshot: true` 表示动画只播放一次。设置为 `false` 则循环播放。

```rust
// 单次播放（默认）
let anim = Animation::new(Duration::from_secs(1));
// anim.oneshot == true

// 循环播放
let anim = Animation::new(Duration::from_secs(1)).repeat();
// anim.oneshot == false
```

### 47.1.3 `easing` 缓动函数

缓动函数将线性进度（0.0 → 1.0）映射为非线性进度，控制动画速度变化。

```rust
use gpui::{Animation, ease_out_quint, bounce};

// 自定义缓动
let anim = Animation::new(Duration::from_millis(500))
    .with_easing(ease_out_quint());
// 快速开始，缓慢结束

// 组合缓动
let anim = Animation::new(Duration::from_millis(500))
    .with_easing(bounce(ease_out_quint()));
// 先前进，再后退，形成弹跳效果
```

## 47.2 `AnimationExt` trait

### 47.2.1 `with_animation()` 创建动画元素

`with_animation()` 为任何元素添加动画。接收 `id`、`Animation` 和一个插值回调。

```rust
use gpui::{Animation, AnimationExt, div};
use std::time::Duration;

div()
    .w(px(100.0))
    .with_animation(
        "slide-in",
        Animation::new(Duration::from_millis(300))
            .with_easing(ease_out_quint()),
        |element, progress| {
            // progress: 0.0 → 1.0
            element.left(px(progress * 200.0 - 200.0)) // 从 -200px 滑入到 0
        },
    )
```

插值回调接收 `(element, progress)`，返回修改后的元素。`progress` 经过缓动函数映射后传入，值域 `[0.0, 1.0]`。

### 47.2.2 `with_animations()` 多动画组合

`with_animations()` 支持多段动画按顺序播放：

```rust
use gpui::{Animation, AnimationExt};

div()
    .with_animations(
        "sequence",
        vec![
            Animation::new(Duration::from_millis(200))
                .with_easing(ease_out_quint()),
            Animation::new(Duration::from_millis(100))
                .with_easing(quadratic),
        ],
        |element, animation_ix, progress| {
            match animation_ix {
                0 => {
                    // 第一段动画：淡入
                    element.opacity(progress)
                }
                1 => {
                    // 第二段动画：放大
                    element.scale(progress * 0.1 + 0.9)
                }
                _ => element,
            }
        },
    )
```

## 47.3 内置缓动函数

### 47.3.1 `linear()` 线性

匀速运动：

```rust
pub fn linear(delta: f32) -> f32 {
    delta
}
```

适用于不需要加速度的场景：进度条、骨架屏。

### 47.3.2 `quadratic()` 二次

```rust
pub fn quadratic(delta: f32) -> f32 {
    delta * delta
}
```

慢进快出。适用于简单的启动加速场景。

### 47.3.3 `ease_in_out()` 先加速后减速

```rust
pub fn ease_in_out(delta: f32) -> f32 {
    if delta < 0.5 {
        2.0 * delta * delta
    } else {
        let x = -2.0 * delta + 2.0;
        1.0 - x * x / 2.0
    }
}
```

平滑的开始和结束。最通用的缓动选择。

### 47.3.4 `ease_out_quint()` 快速开始缓慢结束

```rust
pub fn ease_out_quint() -> impl Fn(f32) -> f32 {
    move |delta| 1.0 - (1.0 - delta).powi(5)
}
```

返回闭包。适用于大多数 UI 过渡：面板展开、菜单弹出。

### 47.3.5 `bounce()` 弹跳

```rust
pub fn bounce(easing: impl Fn(f32) -> f32) -> impl Fn(f32) -> f32 {
    move |delta| {
        if delta < 0.5 {
            easing(delta * 2.0)   // 前进
        } else {
            easing((1.0 - delta) * 2.0) // 回退
        }
    }
}
```

组合其他缓动函数。前半段正向播放，后半段反向播放。

```rust
// 先滑入再滑出一点，形成轻微弹跳
Animation::new(Duration::from_millis(400))
    .with_easing(bounce(ease_out_quint()))
```

### 47.3.6 `pulsating_between()` 脉冲

```rust
pub fn pulsating_between(min: f32, max: f32) -> impl Fn(f32) -> f32 {
    let range = max - min;
    move |delta| {
        let t = (delta * 2.0 * PI).sin();
        let breath = (t * t * t + t) / 2.0;
        let normalized_alpha = (breath + 1.0) / 2.0;
        min + (normalized_alpha * range)
    }
}
```

在 min 和 max 之间连续波动。配合 `.repeat()` 实现呼吸灯效果：

```rust
div()
    .opacity(1.0)
    .with_animation(
        "pulse",
        Animation::new(Duration::from_secs(2))
            .repeat()
            .with_easing(pulsating_between(0.3, 1.0)),
        |element, progress| {
            element.opacity(progress)
        },
    )
```

## 47.4 自定义缓动函数

缓动函数签名是 `Fn(f32) -> f32`，输入输出都在 `[0.0, 1.0]` 范围：

```rust
// 弹簧缓动（带过冲）
fn spring(delta: f32) -> f32 {
    let c1 = 1.70158;
    let c3 = c1 + 1.0;
    1.0 + c3 * (delta - 1.0).powi(3) + c1 * (delta - 1.0).powi(2)
}

// 弹性缓动
fn elastic_out(delta: f32) -> f32 {
    if delta == 0.0 || delta == 1.0 {
        return delta;
    }
    let p = 0.4;
    (2.0f32).powf(-10.0 * delta) * ((delta - p / 4.0) * (2.0 * std::f32::consts::PI) / p).sin() + 1.0
}

Animation::new(Duration::from_millis(500))
    .with_easing(spring)
```

## 47.5 动画状态管理

### 47.5.1 触发动画

动画状态由元素 ID 和 `AnimationState` 追踪。当元素被重新渲染时，如果 ID 相同，GPUI 重用之前的 `AnimationState`；如果 ID 不同，重新开始动画。

```rust
struct AnimatedPanel {
    expanded: bool,
}

impl AnimatedPanel {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("panel")
            .on_click(cx.listener(|this, _, _, cx| {
                this.expanded = !this.expanded;
                cx.notify(); // 触发 render 重新执行，动画自动过渡
            }))
            .child(
                div()
                    .with_animation(
                        "expand",
                        Animation::new(Duration::from_millis(200))
                            .with_easing(ease_out_quint()),
                        |element, progress| {
                            let height = if self.expanded {
                                px(progress * 200.0)
                            } else {
                                px((1.0 - progress) * 200.0)
                            };
                            element.h(height)
                        },
                    )
                    .child("Panel content")
            )
    }
}
```

每次 `expanded` 变化时，render 重新执行，动画从当前状态平滑过渡到新状态。

### 47.5.2 动画完成回调

当前 GPUI 的动画系统不直接提供完成回调。需要监听状态变化：

```rust
struct ExpandAnimation {
    progress: f32,
    done: bool,
}

// 在 render 中检测动画完成
if self.progress >= 1.0 && !self.done {
    self.done = true;
    // 执行完成后的逻辑
}
```

### 47.5.3 动画取消与切换

动画在元素被移除或元素 ID 改变时取消。重新渲染同一元素时，如果状态变化导致不同的参数，动画会从头开始。

```rust
// 切换动画时重新触发
.with_animation(
    if self.expanded { "expand-in" } else { "expand-out" },
    Animation::new(Duration::from_millis(200))
        .with_easing(ease_out_quint()),
    |element, progress| {
        // 展开和折叠共享插值逻辑
        let height = px(if self.expanded { progress * 200.0 } else { (1.0 - progress) * 200.0 });
        element.h(height)
    },
)
```

## 47.6 实战：平滑展开/折叠面板

```rust
use gpui::{Animation, AnimationExt, Context, Render, Styled, div, prelude::*};
use std::time::Duration;
use gpui::ease_out_quint;

struct Accordion {
    sections: Vec<Section>,
    open_index: Option<usize>,
}

struct Section {
    title: String,
    content: String,
}

impl Accordion {
    fn new(sections: Vec<Section>) -> Self {
        Self {
            sections,
            open_index: None,
        }
    }

    fn toggle_section(&mut self, index: usize, cx: &mut Context<Self>) {
        if self.open_index == Some(index) {
            self.open_index = None;
        } else {
            self.open_index = Some(index);
        }
        cx.notify();
    }
}

impl Render for Accordion {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let open_index = self.open_index;
        let sections = self.sections.clone();

        div()
            .flex()
            .flex_col()
            .gap_1()
            .children(sections.into_iter().enumerate().map(|(ix, section)| {
                let is_open = open_index == Some(ix);

                div()
                    .border_1()
                    .rounded_md()
                    .overflow_hidden()
                    .child(
                        div()
                            .id(("header", ix))
                            .p_3()
                            .cursor_pointer()
                            .on_click(cx.listener(move |this, _, _, cx| {
                                this.toggle_section(ix, cx);
                            }))
                            .child(section.title)
                    )
                    .child(
                        div()
                            .overflow_hidden()
                            .with_animation(
                                ("accordion", ix),
                                Animation::new(Duration::from_millis(250))
                                    .with_easing(ease_out_quint()),
                                move |element, progress| {
                                    let height = if is_open {
                                        px(progress * 120.0)
                                    } else {
                                        px((1.0 - progress) * 120.0)
                                    };
                                    element.h(height)
                                },
                            )
                            .p_3()
                            .child(section.content)
                    )
            }))
    }
}
```

要点：
- 用动画 ID 区分不同 section（`("accordion", ix)`）
- 展开和折叠共享插值逻辑，`progress` 控制进度
- `overflow_hidden()` 裁剪超出高度的内容
- `cx.notify()` 触发 render 重新执行，动画自动过渡

动画系统的关键规则：
- `Animation::new(duration)` 创建动画
- `.repeat()` 循环播放
- `.with_easing(f)` 自定义缓动
- `with_animation(id, animation, |element, progress| ...)` 应用动画
- `with_animations()` 多段动画顺序播放
- 动画状态由元素 ID 追踪，ID 改变则重新开始
