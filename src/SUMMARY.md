# Summary

- [前言](foreword.md)

# 第一部分：入门

- [第 1 章：什么是 GPUI](part1-intro/what-is-gpui.md)
- [第 2 章：环境搭建](part1-intro/getting-started.md)
- [第 3 章：第一个应用](part1-intro/first-app.md)

# 第二部分：基础

- [第 4 章：Application 与应用生命周期](part2-fundamentals/application.md)
- [第 5 章：Entity 系统](part2-fundamentals/entity.md)
- [第 6 章：Context 类型](part2-fundamentals/context.md)
- [第 7 章：数据流与所有权](part2-fundamentals/data-flow.md)

# 第三部分：Views

- [第 8 章：Render Trait](part3-views/render-trait.md)
- [第 9 章：View 生命周期](part3-views/view-lifecycle.md)
- [第 10 章：View 组合模式](part3-views/view-patterns.md)

# 第四部分：Elements

- [第 11 章：Element Trait](part4-elements/element-trait.md)
- [第 12 章：div 容器](part4-elements/div.md)
- [第 13 章：文本元素](part4-elements/text.md)
- [第 14 章：图片元素](part4-elements/img.md)
- [第 15 章：SVG 元素](part4-elements/svg.md)
- [第 16 章：Canvas 元素](part4-elements/canvas.md)
- [第 17 章：自定义元素](part4-elements/custom-elements.md)

# 第五部分：布局系统

- [第 18 章：Flexbox 弹性布局](part5-layout/flexbox.md)
- [第 19 章：Grid 网格布局](part5-layout/grid.md)
- [第 20 章：尺寸与间距](part5-layout/sizing.md)
- [第 21 章：Taffy 引擎内部](part5-layout/taffy.md)

# 第六部分：样式

- [第 22 章：Styled Trait](part6-styling/styled.md)
- [第 23 章：颜色系统](part6-styling/colors.md)
- [第 24 章：边框、阴影与变换](part6-styling/effects.md)
- [第 25 章：文本样式](part6-styling/text-styling.md)

# 第七部分：事件系统

- [第 26 章：鼠标事件](part7-events/mouse.md)
- [第 27 章：键盘事件](part7-events/keyboard.md)
- [第 28 章：手势事件](part7-events/gesture.md)
- [第 29 章：事件传播机制](part7-events/propagation.md)

# 第八部分：Action 与 Keymap

- [第 30 章：定义 Action](part8-action-keymap/defining-actions.md)
- [第 31 章：键绑定](part8-action-keymap/key-bindings.md)
- [第 32 章：Key Context](part8-action-keymap/key-context.md)
- [第 33 章：Action 分发](part8-action-keymap/action-dispatch.md)

# 第九部分：焦点与文本输入

- [第 34 章：焦点系统](part9-focus-input/focus.md)
- [第 35 章：文本输入与 IME](part9-focus-input/text-input.md)

# 第十部分：异步编程

- [第 36 章：前台执行器](part10-async/foreground.md)
- [第 37 章：后台执行器](part10-async/background.md)
- [第 38 章：Task 管理](part10-async/task.md)

# 第十一部分：状态模式

- [第 39 章：Observe 模式](part11-state/observe.md)
- [第 40 章：Subscribe 模式](part11-state/subscribe.md)
- [第 41 章：Global 状态](part11-state/global.md)
- [第 42 章：Subscription 生命周期](part11-state/subscription.md)

# 第十二部分：高级元素

- [第 43 章：List 虚拟列表](part12-advanced-elements/list.md)
- [第 44 章：UniformList](part12-advanced-elements/uniform-list.md)
- [第 45 章：Anchored 弹出层](part12-advanced-elements/anchored.md)
- [第 46 章：Deferred 延迟渲染](part12-advanced-elements/deferred.md)
- [第 47 章：动画系统](part12-advanced-elements/animation.md)

# 第十三部分：文本系统深入

- [第 48 章：字体管理](part13-text-system/fonts.md)
- [第 49 章：文本塑形](part13-text-system/shaping.md)
- [第 50 章：行布局](part13-text-system/line-layout.md)

# 第十四部分：渲染管线

- [第 51 章：场景图](part14-rendering/scene.md)
- [第 52 章：GPU 渲染管线](part14-rendering/gpu.md)
- [第 53 章：性能优化](part14-rendering/performance.md)

# 第十五部分：平台

- [第 54 章：Platform Trait](part15-platform/platform.md)
- [第 55 章：macOS 后端](part15-platform/macos.md)
- [第 56 章：Linux 后端](part15-platform/linux.md)

# 第十六部分：测试

- [第 57 章：#[gpui::test]](part16-testing/test.md)
- [第 58 章：TestAppContext](part16-testing/test-app.md)
- [第 59 章：视觉测试](part16-testing/visual.md)
- [第 60 章：属性测试](part16-testing/property.md)

# 第十七部分：调试

- [第 61 章：Element Inspector](part17-debugging/inspector.md)
- [第 62 章：调试样式](part17-debugging/debug-styling.md)
- [第 63 章：性能分析](part17-debugging/profiling.md)

# 第十八部分：Cookbook

- [第 64 章：Hello World](part18-cookbook/hello-world.md)
- [第 65 章：计数器应用](part18-cookbook/counter.md)
- [第 66 章：Todo 应用](part18-cookbook/todo.md)
- [第 67 章：简单文本编辑器](part18-cookbook/editor.md)
- [第 68 章：图片画廊](part18-cookbook/gallery.md)
- [第 69 章：数据仪表盘](part18-cookbook/dashboard.md)
- [附录 A：GPUI 宏一览](appendix/macros.md)
- [附录 B：常用 API 速查](appendix/api-cheat.md)
- [附录 C：Zed 中的 GPUI 实战](appendix/zed-case-study.md)
