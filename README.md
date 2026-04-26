# GPUI 从入门到精通

[![Deploy to GitHub Pages](https://github.com/windtail/gpui-book/actions/workflows/deploy.yml/badge.svg)](https://github.com/windtail/gpui-book/actions/workflows/deploy.yml)
[![Book](https://img.shields.io/badge/%E5%9C%A8%E7%BA%BF%E9%98%85%E8%AF%BB-3b82f6)](https://windtail.github.io/gpui-book/)

一本中文的 [GPUI](https://github.com/zed-industries/zed) 教程，面向有 Rust 经验的开发者。

## 在线阅读

<https://windtail.github.io/gpui-book/>

## 关于 GPUI

GPUI 是 [Zed 编辑器](https://zed.dev) 的渲染引擎和 GUI 框架。它采用 GPU 加速渲染，混合的保留/立即模式架构，通过 Entity-View-Element 三层模型组织界面。

| 平台 | 渲染后端 |
|------|----------|
| macOS | Metal |
| Linux | wgpu |
| Windows | DirectX 11 |

## 本书结构

| 部分 | 内容 |
|------|------|
| [入门](src/part1-intro/) | 什么是 GPUI、环境搭建、第一个应用 |
| [基础](src/part2-fundamentals/) | Application、Entity、Context、数据流 |
| [Views](src/part3-views/) | Render trait、View 生命周期、组合模式 |
| [Elements](src/part4-elements/) | Element trait、div、文本、图片、SVG、Canvas |
| [布局系统](src/part5-layout/) | Flexbox、Grid、尺寸与间距、Taffy 引擎 |
| [样式](src/part6-styling/) | Styled trait、颜色、边框/阴影/变换 |
| [事件系统](src/part7-events/) | 鼠标、键盘、手势、事件传播 |
| [Action 与 Keymap](src/part8-action-keymap/) | 定义 Action、键绑定、Key Context |
| [焦点与输入](src/part9-focus-input/) | FocusHandle、文本输入、IME |
| [异步编程](src/part10-async/) | 前台/后台执行器、Task 管理 |
| [状态模式](src/part11-state/) | Observe、Subscribe、Global、Subscription |
| [高级元素](src/part12-advanced-elements/) | 虚拟列表、弹出层、动画 |
| [文本系统深入](src/part13-text-system/) | 字体管理、文本塑形、行布局 |
| [渲染管线](src/part14-rendering/) | 场景图、GPU 渲染、性能优化 |
| [平台](src/part15-platform/) | Platform trait、macOS/Linux 后端 |
| [测试](src/part16-testing/) | `#[gpui::test]`、视觉测试、属性测试 |
| [调试](src/part17-debugging/) | Inspector、调试样式、性能分析 |
| [Cookbook](src/part18-cookbook/) | 计数器、Todo、文本编辑器、图片画廊、仪表盘 |

## 本地构建

```bash
# 安装 mdbook
cargo install mdbook

# 本地预览（http://localhost:3000）
mdbook serve

# 构建静态站点（输出到 book/）
mdbook build
```

## 源码依据

书中所有 API 描述均基于 [Zed 源码中的 GPUI crate](https://github.com/zed-industries/zed/tree/main/crates/gpui)。GPUI 的 API 仍在演进，如发现不一致，建议直接查阅 GPUI 源码——框架本身就是最好的文档。

## License

本书内容采用 [CC BY 4.0](LICENSE) 协议。
