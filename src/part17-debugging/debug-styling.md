# 第 62 章：调试样式

GPUI 提供 `.debug()` 和 `.debug_below()` 方法，在开发构建中为元素绘制调试边框。

## 62.1 .debug() 边框

`.debug()` 在元素周围绘制彩色边框，帮助可视化布局：

```rust
div()
    .flex()
    .child(div().size_10().bg(blue()).debug())     // 蓝色边框
    .child(div().size_10().bg(red()).debug())      // 红色边框
```

### 实现

```rust
#[cfg(debug_assertions)]
pub fn debug(self) -> Self {
    // 为元素添加调试边框
    self.interactivity().debugging = Some(DebuggingStyle::default());
    self
}
```

仅在 `debug_assertions` 下编译。发布构建中完全消除，零开销。

### 默认调试样式

```rust
pub struct DebuggingStyle {
    pub color: Option<Hsla>,      // 边框颜色
    pub label: Option<String>,    // 可选标签
}

impl Default for DebuggingStyle {
    fn default() -> Self {
        Self {
            color: None,          // 自动分配颜色
            label: None,
        }
    }
}
```

不指定颜色时，GPUI 根据元素在全局树中的位置自动分配颜色，便于区分相邻元素。

## 62.2 .debug_below()

`.debug_below()` 将调试边框绘制在元素内容下方：

```rust
div()
    .size_full()
    .debug_below()  // 边框在子内容下面
    .child(nested_content())
```

### debug vs debug_below

| 方法 | 边框位置 | 使用场景 |
|------|---------|----------|
| `.debug()` | 元素最上层 | 查看元素边界，覆盖内容 |
| `.debug_below()` | 内容下方 | 查看容器布局，不遮挡内容 |

使用 `.debug_below()` 时，边框作为背景层，不会遮挡子元素的视觉内容。

## 62.3 可视化调试技巧

### 布局调试

```rust
// 调试 flex 容器
div()
    .flex()
    .flex_row()
    .gap_2()
    .debug()  // 查看容器实际边界
    .child(left_panel())
    .child(right_panel())
```

### 嵌套调试

```rust
// 为不同层级分配不同颜色
div().debug().child(           // 外层
    div().debug().child(       // 中层
        div().debug()          // 内层
    )
)
```

### 条件调试

```rust
// 只在特定条件下启用调试
div()
    .when(cfg!(debug_assertions), |this| this.debug())
    .child(content())
```

### 标签调试

```rust
// 给元素加标签，方便识别
div()
    .id("main-container")
    .debug()
    .child(content())
```

## 62.4 发布构建中消除

所有调试样式仅在 `debug_assertions` 下编译：

```rust
#[cfg(debug_assertions)]
// 调试代码

#[cfg(not(debug_assertions))]
// 发布构建中完全不存在
```

发布构建中：
- 无额外内存开销
- 无额外绘制调用
- 无性能影响

## 62.5 与 Element Inspector 配合

`.debug()` 和 Element Inspector 互补：

- `.debug()`：快速查看布局和边界
- Inspector：检查元素属性、样式值、源码位置

开发时同时使用两者：
1. 用 `.debug()` 发现布局问题
2. 用 Inspector 定位具体原因
