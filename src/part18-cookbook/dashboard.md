# 第 69 章：数据仪表盘

本章构建一个复杂的数据仪表盘应用，包含多窗口、Tab 导航、分割面板、异步数据和完整的 Action/Keymap 系统。

## Cargo.toml

```toml
[package]
name = "dashboard"
version = "0.1.0"
edition = "2024"

[dependencies]
gpui = "0.2.2"
```

## 定义 Action

```rust
use gpui::*;
use std::time::Duration;
use std::collections::HashMap;

actions!(dashboard, [
    NewWindow,
    RefreshData,
    TabOverview,
    TabMetrics,
    TabLogs,
    SplitPane,
]);

#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema)]
struct SwitchTab {
    index: usize,
}
```

## 数据模型

```rust
#[derive(Clone)]
struct MetricData {
    label: SharedString,
    value: f64,
    unit: SharedString,
    trend: f64, // 正数为上升，负数为下降
}

#[derive(Clone)]
struct LogEntry {
    timestamp: SharedString,
    level: SharedString,
    message: SharedString,
}
```

## Dashboard View

```rust
struct Dashboard {
    active_tab: usize,
    tab_names: Vec<SharedString>,
    metrics: Vec<MetricData>,
    logs: Vec<LogEntry>,
    refreshing: bool,
    focus_handles: Vec<FocusHandle>,
}

impl Dashboard {
    fn new(cx: &mut Context<Self>) -> Self {
        let mut focus_handles = Vec::new();
        for _ in 0..3 {
            focus_handles.push(cx.focus_handle());
        }

        Self {
            active_tab: 0,
            tab_names: vec![
                SharedString::from("Overview"),
                SharedString::from("Metrics"),
                SharedString::from("Logs"),
            ],
            metrics: Vec::new(),
            logs: Vec::new(),
            refreshing: false,
            focus_handles,
        }
    }

    fn switch_tab(&mut self, index: usize, cx: &mut Context<Self>) {
        if index < self.tab_names.len() {
            self.active_tab = index;
            cx.notify();
        }
    }

    fn refresh_data(&mut self, cx: &mut Context<Self>) {
        if self.refreshing {
            return;
        }
        self.refreshing = true;
        cx.notify();

        // 异步加载模拟数据
        cx.spawn(|view, mut cx| async move {
            // 模拟网络请求延迟
            cx.background_executor().timer(Duration::from_secs(1)).await;

            let metrics = vec![
                MetricData {
                    label: SharedString::from("CPU Usage"),
                    value: 67.3,
                    unit: SharedString::from("%"),
                    trend: 2.1,
                },
                MetricData {
                    label: SharedString::from("Memory"),
                    value: 4.2,
                    unit: SharedString::from("GB"),
                    trend: -0.5,
                },
                MetricData {
                    label: SharedString::from("Requests/s"),
                    value: 1240.0,
                    unit: SharedString::from("req/s"),
                    trend: 15.3,
                },
                MetricData {
                    label: SharedString::from("Latency p99"),
                    value: 45.0,
                    unit: SharedString::from("ms"),
                    trend: -8.2,
                },
            ];

            let logs = vec![
                LogEntry {
                    timestamp: SharedString::from("10:42:01"),
                    level: SharedString::from("INFO"),
                    message: SharedString::from("Server started on port 8080"),
                },
                LogEntry {
                    timestamp: SharedString::from("10:42:05"),
                    level: SharedString::from("WARN"),
                    message: SharedString::from("High memory usage detected"),
                },
                LogEntry {
                    timestamp: SharedString::from("10:42:12"),
                    level: SharedString::from("ERROR"),
                    message: SharedString::from("Connection pool exhausted"),
                },
                LogEntry {
                    timestamp: SharedString::from("10:42:15"),
                    level: SharedString::from("INFO"),
                    message: SharedString::from("Connection pool recovered"),
                },
            ];

            // 更新视图状态
            view.update(&mut cx, |this, cx| {
                this.metrics = metrics;
                this.logs = logs;
                this.refreshing = false;
                cx.notify();
            }).ok();
        }).detach();
    }

    fn open_new_window(&self, cx: &mut Context<Self>) {
        // 创建新窗口，共享相同的数据源
        cx.open_window(WindowOptions::default(), |_window, cx| {
            cx.new(|cx| Self::new(cx))
        }).ok();
    }
}
```

## Tab 栏渲染

```rust
impl Dashboard {
    fn render_tab_bar(&self, cx: &mut Context<Self>) -> impl IntoElement + '_ {
        let active = self.active_tab;

        div()
            .flex()
            .bg(rgb(0x1a1a1a))
            .border_b_1()
            .border_color(rgb(0x333333))
            .children(
                self.tab_names.iter().enumerate().map(|(i, name)| {
                    let is_active = i == active;
                    div()
                        .id(SharedString::from(format!("tab-{}", i)))
                        .px_4()
                        .py_2()
                        .text_size(px(14.))
                        .text_color(if is_active { rgb(0xffffff) } else { rgb(0x888888) })
                        .border_b_2()
                        .border_color(if is_active { rgb(0x3b82f6) } else { rgb(0x333333) })
                        .cursor_pointer()
                        .on_click(cx.listener(move |this, _, _, cx| {
                            this.switch_tab(i, cx);
                        }))
                        .child(name.clone())
                }),
            )
    }
}
```

## 内容面板渲染

```rust
impl Dashboard {
    fn render_content(&self, cx: &mut Context<Self>) -> impl IntoElement + '_ {
        match self.active_tab {
            0 => self.render_overview(cx),
            1 => self.render_metrics(cx),
            2 => self.render_logs(cx),
            _ => div().child("Unknown tab"),
        }
    }

    fn render_overview(&self, cx: &mut Context<Self>) -> impl IntoElement + '_ {
        div()
            .flex_1()
            .p_4()
            .overflow_scroll()
            .child(
                div()
                    .grid()
                    .grid_cols(2)
                    .gap_4()
                    .children(
                        self.metrics.iter().enumerate().map(|(i, m)| {
                            self.render_metric_card(i, m)
                        }).collect::<Vec<_>>(),
                    ),
            )
    }

    fn render_metric_card(&self, _index: usize, metric: &MetricData) -> impl IntoElement + '_ {
        let trend_color = if metric.trend > 0.0 {
            rgb(0x51cf66)
        } else {
            rgb(0xff6b6b)
        };
        let trend_sign = if metric.trend > 0.0 { "+" } else { "" };

        div()
            .p_4()
            .rounded_md()
            .bg(rgb(0x2a2a2a))
            .border_1()
            .border_color(rgb(0x3a3a3a))
            .child(
                div()
                    .text_size(px(14.))
                    .text_color(rgb(0xaaaaaa))
                    .child(metric.label.clone()),
            )
            .child(
                div()
                    .flex()
                    .items_end()
                    .gap_2()
                    .mt_2()
                    .child(
                        div()
                            .text_size(px(32.))
                            .text_color(rgb(0xffffff))
                            .font_weight(FontWeight::BOLD)
                            .child(format!("{:.1}", metric.value)),
                    )
                    .child(
                        div()
                            .text_size(px(14.))
                            .text_color(rgb(0x888888))
                            .child(metric.unit.clone()),
                    ),
            )
            .child(
                div()
                    .text_size(px(12.))
                    .text_color(trend_color)
                    .mt_1()
                    .child(format!("{}{:.1}%", trend_sign, metric.trend)),
            )
    }

    fn render_metrics(&self, cx: &mut Context<Self>) -> impl IntoElement + '_ {
        div()
            .flex_1()
            .p_4()
            .overflow_scroll()
            .child(
                div()
                    .text_size(px(18.))
                    .text_color(rgb(0xffffff))
                    .mb_4()
                    .child("All Metrics"),
            )
            .child(
                div()
                    .flex()
                    .flex_col()
                    .gap_2()
                    .children(
                        self.metrics.iter().map(|m| {
                            div()
                                .flex()
                                .justify_between()
                                .items_center()
                                .p_3()
                                .rounded_sm()
                                .bg(rgb(0x2a2a2a))
                                .child(div().text_color(rgb(0xffffff)).child(m.label.clone()))
                                .child(
                                    div().flex().items_center().gap_2()
                                        .child(div().text_size(px(20.)).text_color(rgb(0xffffff))
                                            .child(format!("{:.1}", m.value)))
                                        .child(div().text_color(rgb(0x888888)).child(m.unit.clone()))
                                )
                        }).collect::<Vec<_>>(),
                    ),
            )
    }

    fn render_logs(&self, cx: &mut Context<Self>) -> impl IntoElement + '_ {
        let log_color = |level: &str| -> Hsla {
            match level {
                "ERROR" => rgb(0xff6b6b),
                "WARN" => rgb(0xffd93d),
                "INFO" => rgb(0x51cf66),
                _ => rgb(0xaaaaaa),
            }
        };

        div()
            .flex_1()
            .p_4()
            .overflow_scroll()
            .child(
                div()
                    .flex()
                    .flex_col()
                    .gap_1()
                    .children(
                        self.logs.iter().map(|log| {
                            div()
                                .flex()
                                .gap_3()
                                .p_2()
                                .rounded_sm()
                                .font_family("monospace")
                                .text_size(px(14.))
                                .child(
                                    div()
                                        .text_color(rgb(0x666666))
                                        .w(px(70.))
                                        .child(log.timestamp.clone()),
                                )
                                .child(
                                    div()
                                        .text_color(log_color(&log.level))
                                        .w(px(50.))
                                        .font_weight(FontWeight::BOLD)
                                        .child(log.level.clone()),
                                )
                                .child(
                                    div()
                                        .flex_1()
                                        .text_color(rgb(0xd4d4d4))
                                        .child(log.message.clone()),
                                )
                        }).collect::<Vec<_>>(),
                    ),
            )
    }
}
```

## Render 与工具栏

```rust
impl Render for Dashboard {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full()
            .flex()
            .flex_col()
            .bg(rgb(0x1e1e1e))
            // 工具栏
            .child(
                div()
                    .flex()
                    .items_center()
                    .justify_between()
                    .px_4()
                    .py_2()
                    .bg(rgb(0x252525))
                    .border_b_1()
                    .border_color(rgb(0x333333))
                    .child(
                        div()
                            .text_size(px(16.))
                            .text_color(rgb(0xffffff))
                            .child("Dashboard"),
                    )
                    .child(
                        div()
                            .flex()
                            .gap_2()
                            .child(
                                button("new-window", "New Window")
                                    .on_click(cx.listener(|this, _, _, cx| {
                                        this.open_new_window(cx);
                                    })),
                            )
                            .child(
                                button("refresh", if self.refreshing { "Loading..." } else { "Refresh" })
                                    .when(self.refreshing, |btn| btn.enabled(false))
                                    .on_click(cx.listener(|this, _, _, cx| {
                                        this.refresh_data(cx);
                                    })),
                            ),
                    ),
            )
            // Tab 栏
            .child(self.render_tab_bar(cx))
            // 内容区
            .child(self.render_content(cx))
    }
}
```

## 注册 Action 与 Keymap

```rust
fn register_actions(cx: &mut App) {
    cx.bind_keys(vec![
        KeyBinding::new("ctrl-n", NewWindow, None),
        KeyBinding::new("cmd-n", NewWindow, None),
        KeyBinding::new("ctrl-r", RefreshData, None),
        KeyBinding::new("cmd-r", RefreshData, None),
        KeyBinding::new("ctrl-1", SwitchTab { index: 0 }, None),
        KeyBinding::new("ctrl-2", SwitchTab { index: 1 }, None),
        KeyBinding::new("ctrl-3", SwitchTab { index: 2 }, None),
    ]);
}
```

## 主函数

```rust
fn main() {
    let app = Application::new();
    app.run(|cx| {
        // 注册全局键绑定
        cx.bind_keys(vec![
            KeyBinding::new("ctrl-n", NewWindow, None),
            KeyBinding::new("cmd-n", NewWindow, None),
            KeyBinding::new("ctrl-r", RefreshData, None),
            KeyBinding::new("cmd-r", RefreshData, None),
            KeyBinding::new("ctrl-1", SwitchTab { index: 0 }, None),
            KeyBinding::new("ctrl-2", SwitchTab { index: 1 }, None),
            KeyBinding::new("ctrl-3", SwitchTab { index: 2 }, None),
        ]);

        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(800.), px(600.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Dashboard")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| {
                let dashboard = cx.new(|cx| {
                    let mut d = Dashboard::new(cx);
                    d.refresh_data(cx); // 初始加载
                    d
                });

                // 在根元素上注册 Action 处理
                dashboard
            },
        );
    });
}
```

## 完整代码

```rust
use gpui::*;
use std::time::Duration;

fn button(id: impl Into<ElementId>, text: impl Into<SharedString>) -> impl IntoElement {
    div()
        .id(id)
        .px_3()
        .py_1()
        .rounded_md()
        .bg(rgb(0x3b82f6))
        .text_white()
        .cursor_pointer()
        .child(text.into())
}

actions!(dashboard, [
    NewWindow,
    RefreshData,
]);

#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema)]
struct SwitchTab {
    index: usize,
}

#[derive(Clone)]
struct MetricData {
    label: SharedString,
    value: f64,
    unit: SharedString,
    trend: f64,
}

#[derive(Clone)]
struct LogEntry {
    timestamp: SharedString,
    level: SharedString,
    message: SharedString,
}

struct Dashboard {
    active_tab: usize,
    tab_names: Vec<SharedString>,
    metrics: Vec<MetricData>,
    logs: Vec<LogEntry>,
    refreshing: bool,
}

impl Dashboard {
    fn new(_cx: &mut Context<Self>) -> Self {
        Self {
            active_tab: 0,
            tab_names: vec![
                SharedString::from("Overview"),
                SharedString::from("Metrics"),
                SharedString::from("Logs"),
            ],
            metrics: Vec::new(),
            logs: Vec::new(),
            refreshing: false,
        }
    }

    fn switch_tab(&mut self, index: usize, cx: &mut Context<Self>) {
        if index < self.tab_names.len() {
            self.active_tab = index;
            cx.notify();
        }
    }

    fn refresh_data(&mut self, cx: &mut Context<Self>) {
        if self.refreshing { return; }
        self.refreshing = true;
        cx.notify();

        cx.spawn(|view, mut cx| async move {
            cx.background_executor().timer(Duration::from_secs(1)).await;

            let metrics = vec![
                MetricData { label: SharedString::from("CPU Usage"), value: 67.3, unit: SharedString::from("%"), trend: 2.1 },
                MetricData { label: SharedString::from("Memory"), value: 4.2, unit: SharedString::from("GB"), trend: -0.5 },
                MetricData { label: SharedString::from("Requests/s"), value: 1240.0, unit: SharedString::from("req/s"), trend: 15.3 },
                MetricData { label: SharedString::from("Latency p99"), value: 45.0, unit: SharedString::from("ms"), trend: -8.2 },
            ];
            let logs = vec![
                LogEntry { timestamp: SharedString::from("10:42:01"), level: SharedString::from("INFO"), message: SharedString::from("Server started on port 8080") },
                LogEntry { timestamp: SharedString::from("10:42:05"), level: SharedString::from("WARN"), message: SharedString::from("High memory usage detected") },
                LogEntry { timestamp: SharedString::from("10:42:12"), level: SharedString::from("ERROR"), message: SharedString::from("Connection pool exhausted") },
                LogEntry { timestamp: SharedString::from("10:42:15"), level: SharedString::from("INFO"), message: SharedString::from("Connection pool recovered") },
            ];

            view.update(&mut cx, |this, cx| {
                this.metrics = metrics;
                this.logs = logs;
                this.refreshing = false;
                cx.notify();
            }).ok();
        }).detach();
    }

    fn open_new_window(&self, cx: &mut Context<Self>) {
        cx.open_window(WindowOptions::default(), |_window, cx| {
            cx.new(|cx| {
                let mut d = Self::new(cx);
                d.refresh_data(cx);
                d
            })
        }).ok();
    }
}

impl Dashboard {
    fn render_tab_bar(&self, cx: &mut Context<Self>) -> impl IntoElement + '_ {
        let active = self.active_tab;
        div()
            .flex()
            .bg(rgb(0x1a1a1a))
            .border_b_1()
            .border_color(rgb(0x333333))
            .children(
                self.tab_names.iter().enumerate().map(|(i, name)| {
                    let is_active = i == active;
                    div()
                        .id(SharedString::from(format!("tab-{}", i)))
                        .px_4().py_2()
                        .text_size(px(14.))
                        .text_color(if is_active { rgb(0xffffff) } else { rgb(0x888888) })
                        .border_b_2()
                        .border_color(if is_active { rgb(0x3b82f6) } else { rgb(0x333333) })
                        .cursor_pointer()
                        .on_click(cx.listener(move |this, _, _, cx| {
                            this.switch_tab(i, cx);
                        }))
                        .child(name.clone())
                }),
            )
    }

    fn render_content(&self, cx: &mut Context<Self>) -> impl IntoElement + '_ {
        match self.active_tab {
            0 => self.render_overview(cx),
            1 => self.render_metrics(cx),
            2 => self.render_logs(cx),
            _ => div().child("Unknown tab"),
        }
    }

    fn render_overview(&self, _cx: &mut Context<Self>) -> impl IntoElement + '_ {
        div()
            .flex_1().p_4().overflow_scroll()
            .child(
                div().grid().grid_cols(2).gap_4()
                    .children(self.metrics.iter().enumerate().map(|(i, m)| self.render_metric_card(i, m)).collect::<Vec<_>>()),
            )
    }

    fn render_metric_card(&self, _index: usize, metric: &MetricData) -> impl IntoElement + '_ {
        let trend_color = if metric.trend > 0.0 { rgb(0x51cf66) } else { rgb(0xff6b6b) };
        let trend_sign = if metric.trend > 0.0 { "+" } else { "" };
        div()
            .p_4().rounded_md().bg(rgb(0x2a2a2a)).border_1().border_color(rgb(0x3a3a3a))
            .child(div().text_size(px(14.)).text_color(rgb(0xaaaaaa)).child(metric.label.clone()))
            .child(
                div().flex().items_end().gap_2().mt_2()
                    .child(div().text_size(px(32.)).text_color(rgb(0xffffff)).font_weight(FontWeight::BOLD).child(format!("{:.1}", metric.value)))
                    .child(div().text_size(px(14.)).text_color(rgb(0x888888)).child(metric.unit.clone())),
            )
            .child(div().text_size(px(12.)).text_color(trend_color).mt_1().child(format!("{}{:.1}%", trend_sign, metric.trend)))
    }

    fn render_metrics(&self, _cx: &mut Context<Self>) -> impl IntoElement + '_ {
        div()
            .flex_1().p_4().overflow_scroll()
            .child(div().text_size(px(18.)).text_color(rgb(0xffffff)).mb_4().child("All Metrics"))
            .child(
                div().flex().flex_col().gap_2()
                    .children(self.metrics.iter().map(|m| {
                        div().flex().justify_between().items_center().p_3().rounded_sm().bg(rgb(0x2a2a2a))
                            .child(div().text_color(rgb(0xffffff)).child(m.label.clone()))
                            .child(div().flex().items_center().gap_2()
                                .child(div().text_size(px(20.)).text_color(rgb(0xffffff)).child(format!("{:.1}", m.value)))
                                .child(div().text_color(rgb(0x888888)).child(m.unit.clone())))
                    }).collect::<Vec<_>>()),
            )
    }

    fn render_logs(&self, _cx: &mut Context<Self>) -> impl IntoElement + '_ {
        let log_color = |level: &str| -> Hsla {
            match level {
                "ERROR" => rgb(0xff6b6b),
                "WARN" => rgb(0xffd93d),
                "INFO" => rgb(0x51cf66),
                _ => rgb(0xaaaaaa),
            }
        };
        div()
            .flex_1().p_4().overflow_scroll()
            .child(
                div().flex().flex_col().gap_1()
                    .children(self.logs.iter().map(|log| {
                        div().flex().gap_3().p_2().rounded_sm().font_family("monospace").text_size(px(14.))
                            .child(div().text_color(rgb(0x666666)).w(px(70.)).child(log.timestamp.clone()))
                            .child(div().text_color(log_color(&log.level)).w(px(50.)).font_weight(FontWeight::BOLD).child(log.level.clone()))
                            .child(div().flex_1().text_color(rgb(0xd4d4d4)).child(log.message.clone()))
                    }).collect::<Vec<_>>()),
            )
    }
}

impl Render for Dashboard {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .size_full().flex().flex_col().bg(rgb(0x1e1e1e))
            .child(
                div().flex().items_center().justify_between().px_4().py_2().bg(rgb(0x252525))
                    .border_b_1().border_color(rgb(0x333333))
                    .child(div().text_size(px(16.)).text_color(rgb(0xffffff)).child("Dashboard"))
                    .child(
                        div().flex().gap_2()
                            .child(button("new-window", "New Window")
                                .on_click(cx.listener(|this, _, _, cx| this.open_new_window(cx))))
                            .child(button("refresh", if self.refreshing { "Loading..." } else { "Refresh" })
                                .on_click(cx.listener(|this, _, _, cx| this.refresh_data(cx)))),
                    ),
            )
            .child(self.render_tab_bar(cx))
            .child(self.render_content(cx))
    }
}

fn main() {
    let app = Application::new();
    app.run(|cx| {
        cx.bind_keys(vec![
            KeyBinding::new("ctrl-n", NewWindow, None),
            KeyBinding::new("cmd-n", NewWindow, None),
            KeyBinding::new("ctrl-r", RefreshData, None),
            KeyBinding::new("cmd-r", RefreshData, None),
            KeyBinding::new("ctrl-1", SwitchTab { index: 0 }, None),
            KeyBinding::new("ctrl-2", SwitchTab { index: 1 }, None),
            KeyBinding::new("ctrl-3", SwitchTab { index: 2 }, None),
        ]);

        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(
                    Bounds::new(Point::new(px(100.), px(100.)), size(px(800.), px(600.))),
                )),
                titlebar: Some(TitlebarOptions {
                    title: Some(SharedString::from("Dashboard")),
                    ..Default::default()
                }),
                ..Default::default()
            },
            |window, cx| {
                let dashboard = cx.new(|cx| {
                    let mut d = Dashboard::new(cx);
                    d.refresh_data(cx);
                    d
                });
                dashboard
            },
        );
    });
}
```

## 关键要点

- `cx.open_window()` 可以在回调中调用以创建新窗口，每个窗口有独立的 View 实例
- Tab 导航通过 View 状态（`active_tab: usize`）控制，`render()` 中 `match` 分支选择面板
- `cx.spawn(|view, mut cx| async move { ... })` 启动异步任务，`view.update()` 安全更新状态
- `cx.background_executor().timer()` 模拟异步延迟
- `cx.bind_keys()` 注册全局快捷键，`KeyBinding::new(keystroke, action, context)`
- 带参数的 Action（如 `SwitchTab { index: 0 }`）需要在 struct 上实现 `Action` trait 或使用 `#[derive(Action)]`
- 多窗口间状态默认独立——如果需要共享数据，使用 `Global` 或 `Entity` 传递
- `.when(condition, |btn| btn.enabled(false))` 条件禁用按钮
- 刷新状态标记防止重复请求
