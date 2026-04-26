# 第 59 章：视觉测试

GPUI 提供视觉测试框架，用于验证渲染输出的正确性。

## 59.1 视觉测试框架

视觉测试捕获窗口的渲染结果，与预期截图对比。这确保 UI 变化不会意外改变视觉外观。

### 基本要求

视觉测试目前仅在 macOS 上可用，依赖 CoreGraphics 截图能力。

```rust
#[cfg(target_os = "macos")]
#[gpui::test]
async fn test_visual_counter(cx: &TestAppContext) {
    // 创建窗口
    let window = cx.open_window(WindowOptions::default(), |_, cx| {
        cx.new(|cx| Counter { count: 42 })
    });

    // 等待渲染完成
    cx.run_until_parked();

    // 视觉验证
    window.draw();
    // 与预期截图对比
}
```

## 59.2 VisualTestContext

`VisualTestContext` 提供截图和对比能力：

```rust
#[cfg(target_os = "macos")]
pub struct VisualTestContext {
    // 窗口句柄
    window: AnyWindowHandle,
    // 测试平台
    platform: Rc<TestPlatform>,
}
```

### 截图

```rust
// 捕获当前窗口内容
let screenshot = visual_context.capture();

// 返回包含像素数据的结构
pub struct Screenshot {
    pub width: usize,
    pub height: usize,
    pub pixels: Vec<u8>,  // RGBA
}
```

### 对比

```rust
// 与预期截图对比
let result = screenshot.compare(&expected_screenshot);

// 返回差异信息
pub struct ComparisonResult {
    pub matches: bool,
    pub diff_pixels: usize,
    pub diff_image: Option<Screenshot>,
}
```

## 59.3 截图存储

### 基准截图

视觉测试使用基准截图文件：

```
tests/
  visual/
    screenshots/
      test_counter/
        expected.png    # 基准截图
        actual.png       # 当前运行结果
        diff.png         # 差异图像
```

### 更新基准

```bash
# 更新所有视觉测试的基准截图
UPDATE_VISUAL_TESTS=1 cargo test visual_

# 更新特定测试
UPDATE_VISUAL_TESTS=1 cargo test test_counter_visual
```

## 59.4 像素级对比

对比算法检查每个像素的 RGBA 值：

```rust
fn compare_screenshots(a: &Screenshot, b: &Screenshot) -> ComparisonResult {
    assert_eq!(a.width, b.width);
    assert_eq!(a.height, b.height);

    let mut diff_pixels = 0;
    let mut diff_data = Vec::with_capacity(a.pixels.len());

    for i in 0..a.pixels.len() / 4 {
        let ai = i * 4;
        if a.pixels[ai..ai+4] != b.pixels[ai..ai+4] {
            diff_pixels += 1;
            // 差异标记为红色
            diff_data.extend_from_slice(&[255, 0, 0, 255]);
        } else {
            diff_data.extend_from_slice(&b.pixels[ai..ai+4]);
        }
    }

    ComparisonResult {
        matches: diff_pixels == 0,
        diff_pixels,
        diff_image: if diff_pixels > 0 { Some(...) } else { None },
    }
}
```

### 容差对比

允许微小差异（抗锯齿、子像素渲染）：

```rust
// 带容差的对比
let result = screenshot.compare_with_tolerance(
    &expected,
    tolerance: 1,  // 允许每个通道 1 的差异
    max_diff_percent: 0.01,  // 最多 0.01% 像素不同
);
```

## 59.5 测试不同状态

```rust
#[cfg(target_os = "macos")]
#[gpui::test]
async fn test_button_states(cx: &TestAppContext) {
    let window = cx.open_window(WindowOptions::default(), |_, cx| {
        cx.new(|cx| ButtonDemo::new(cx))
    });

    cx.run_until_pared();

    // 默认状态
    window.draw();
    visual_context.verify_screenshot("button_default");

    // 悬停状态
    window.simulate_mouse_move(Point::new(px(50.), px(25.)));
    cx.run_until_parked();
    window.draw();
    visual_context.verify_screenshot("button_hover");

    // 按下状态
    window.simulate_mouse_down(Point::new(px(50.), px(25.)), MouseButton::Left);
    cx.run_until_parked();
    window.draw();
    visual_context.verify_screenshot("button_active");
}
```

## 59.6 视觉测试的限制

- 仅 macOS 可用
- 不同硬件可能产生微小差异
- 文本渲染可能因系统字体版本不同而有差异
- 运行时间比普通测试长
- 基准截图需要人工审核

### 何时使用视觉测试

- 组件库的回归检测
- 主题/颜色系统验证
- 复杂渲染效果的稳定性
- 动画关键帧验证

### 何时不使用

- 频繁变化的 UI
- 开发中的功能
- 非视觉逻辑的测试
