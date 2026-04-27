# GPUI 从入门到精通 - 详细章节规划

> 目标读者：有 Rust 经验的开发者
> 写作风格：代码先行，每章都从具体代码示例入手，再解释原理

---

## 第一部分：入门

### 第 1 章：什么是 GPUI
- 1.1 从 Zed 编辑器说起
- 1.2 GPUI 的核心设计理念
  - 1.2.1 GPU 优先渲染
  - 1.2.2 混合模式：声明式 View + 命令式 Element
  - 1.2.3 单一所有者：Entity 模型
- 1.3 GPUI vs 其他 Rust GUI 框架
  - 1.3.1 与 egui 对比（纯立即模式）
  - 1.3.2 与 Iced 对比（Elm 架构）
  - 1.3.3 与 Slint 对比（DSL + 嵌入式）
- 1.4 GPUI 的成熟度与生态
  - 1.4.1 当前状态与已知限制
  - 1.4.2 平台支持：macOS / Linux / Windows(开发中)
  - 1.4.3 Zed 中的应用实例
- 1.5 阅读本书的预备知识
  - 1.5.1 Rust 基础知识
  - 1.5.2 异步编程概念
  - 1.5.3 GUI 编程基本术语

### 第 2 章：环境搭建
- 2.1 系统要求
  - 2.1.1 Rust 版本与工具链
  - 2.1.2 macOS 依赖：Xcode
  - 2.1.3 Linux 依赖：wgpu 所需图形库、Wayland/X11 开发库
- 2.2 Cargo.toml 配置
  - 2.2.1 添加 `gpui` 依赖
  - 2.2.2 常用 feature flags
  - 2.2.3 依赖的传递性：`util`、`collections` 等
- 2.3 项目结构
  - 2.3.1 推荐的目录布局
  - 2.3.2 `main.rs` 入口约定
  - 2.3.3 模块化建议
- 2.4 gpui::prelude 导览
  - 2.4.1 为什么需要 prelude
  - 2.4.2 prelude 导入了什么
  - 2.4.3 何时使用完整路径

### 第 3 章：第一个应用
- 3.1 最小可运行示例
  - 3.1.1 `Application::new()` + `App::run()`
  - 3.1.2 创建窗口与设置标题
  - 3.1.3 完整的 "Hello, GPUI!"
- 3.2 窗口配置
  - 3.2.1 `WindowBounds` 与尺寸设置
  - 3.2.2 窗口选项：`title_bar`、`window_kind`
  - 3.2.3 全屏与无边框模式
- 3.3 视图与 Render
  - 3.3.1 定义你的第一个 View struct
  - 3.3.2 实现 `Render` trait
  - 3.3.3 用 `div()` 构建 UI
- 3.4 编译、运行、调试
  - 3.4.1 `cargo run` 与常见编译错误
  - 3.4.2 热重载的可能性
  - 3.4.3 本章小结与下一步

---

## 第二部分：基础

### 第 4 章：Application 与应用生命周期
- 4.1 `App` 是唯一的真理
  - 4.1.1 `App` 拥有所有状态
  - 4.1.2 `AppCell` 全局存储
  - 4.1.3 为什么不能直接持有可变引用
- 4.2 应用生命周期
  - 4.2.1 `Application::new()` 初始化
  - 4.2.2 事件循环内部
  - 4.2.3 退出与清理：`quit()`、`on_quit()`
- 4.3 `AppContext` trait
  - 4.3.1 trait 定义与核心方法
  - 4.3.2 `App` 与上下文的关系
  - 4.3.3 在回调中使用上下文
- 4.4 应用级服务
  - 4.4.1 剪贴板操作
  - 4.4.2 文件对话框
  - 4.4.3 打开 URL
  - 4.4.4 应用菜单（macOS）
- 4.5 多窗口管理
  - 4.5.1 `open_window()` 创建新窗口
  - 4.5.2 窗口间的通信
  - 4.5.3 窗口事件监听

### 第 5 章：Entity 系统
- 5.1 `Entity<T>` 是什么
  - 5.1.1 GPUI 的智能指针模式
  - 5.1.2 `Entity<T>` vs `Rc<RefCell<T>>`
  - 5.1.3 实体身份：`EntityId`
- 5.2 创建实体
  - 5.2.1 `cx.new(|cx| MyEntity { ... })`
  - 5.2.2 闭包中的初始化逻辑
  - 5.2.3 `reserve_entity()` 延迟创建
- 5.3 读写实体
  - 5.3.1 `Entity::read()` 获取只读访问
  - 5.3.2 `Entity::update()` 获取可变访问
  - 5.3.3 借用检查与运行时 panic 场景
  - 5.3.4 `WeakEntity<T>` 弱引用与生命周期
- 5.4 实体生命周期
  - 5.4.1 引用计数与自动回收
  - 5.4.2 何时实体会被销毁
  - 5.4.3 悬垂引用的防护
- 5.5 Entity 的最佳实践
  - 5.5.1 什么应该放入 Entity
  - 5.5.2 Entity 与 View 的对应关系
  - 5.5.3 避免 Entity 过多或过少

### 第 6 章：Context 类型
- 6.1 Context 的层次结构
  - 6.1.1 `App` —— 应用级引用
  - 6.1.2 `Context<T>` —— 实体专用上下文
  - 6.1.3 `AppContext` trait —— 统一接口
- 6.2 `Context<T>` 详解
  - 6.2.1 `Deref<Target = App>` 的隐含能力
  - 6.2.2 `VisualContext` trait 与窗口操作
  - 6.2.3 `BorrowAppContext` 的使用场景
- 6.3 上下文方法分类
  - 6.3.1 实体操作：`new()`、`observe()`、`subscribe()`
  - 6.3.2 全局状态：`set_global()`、`global()`
  - 6.3.3 异步操作：`spawn()`、`background_spawn()`
  - 6.3.4 事件操作：`emit()`、`notify()`
- 6.4 ViewContext 与 WindowContext
  - 6.4.1 ViewContext：视图渲染时的上下文
  - 6.4.2 WindowContext：窗口事件处理时的上下文
  - 6.4.3 两者的转换与限制
- 6.5 上下文所有权模式
  - 6.5.1 闭包捕获规则
  - 6.5.2 `cx.with()` 与借用优化
  - 6.5.3 常见的编译错误与解决方案

### 第 7 章：数据流与所有权
- 7.1 GPUI 的数据流模型
  - 7.1.1 数据流向图：Entity → View → Element
  - 7.1.2 单向数据流原则
  - 7.1.3 为什么没有双向绑定
- 7.2 update/read 模式
  - 7.2.1 读多写少的典型场景
  - 7.2.2 嵌套 update 的陷阱
  - 7.2.3 批量更新技巧
- 7.3 数据流模式
  - 7.3.1 状态下沉：将数据传递给子 View
  - 7.3.2 事件上浮：从子 View 向上传递事件
  - 7.3.3 共享状态：Global vs Entity
- 7.4 常见反模式
  - 7.4.1 在 render 中修改状态
  - 7.4.2 Entity 之间的循环引用
  - 7.4.3 忘记调用 `cx.notify()`

---

## 第三部分：Views

### 第 8 章：Render Trait
- 8.1 `Render` trait 定义
  - 8.1.1 `render(&mut self, window, cx) -> impl IntoElement`
  - 8.1.2 为什么需要 `&mut self`
  - 8.1.3 返回值的生命周期
- 8.2 `RenderOnce` trait
  - 8.2.1 无状态组件的优化
  - 8.2.2 `RenderOnce` vs `Render` 的选择
  - 8.2.3 `Component<C>` 包装器
- 8.3 从 render 返回元素树
  - 8.3.1 `div()` 作为根元素
  - 8.3.2 条件渲染
  - 8.3.3 列表渲染与迭代
- 8.4 视图作为 UI 状态容器
  - 8.4.1 View struct 应该包含什么
  - 8.4.2 计算属性与缓存
  - 8.4.3 性能敏感的 render 优化
- 8.5 `#[derive(IntoElement)]`
  - 8.5.1 什么时候需要这个 derive
  - 8.5.2 与 `RenderOnce` 的关系
  - 8.5.3 常见编译错误

### 第 9 章：View 生命周期
- 9.1 视图创建
  - 9.1.1 `cx.new()` 创建流程
  - 9.1.2 `Entity<V>` 与 `AnyView` 的关系
  - 9.1.3 窗口根视图设置
- 9.2 渲染触发
  - 9.2.1 `cx.notify()` 手动通知
  - 9.2.2 `cx.observe()` 自动通知
  - 9.2.3 帧调度与渲染时机
- 9.3 视图缓存
  - 9.3.1 `AnyView::cached()` 的作用
  - 9.3.2 缓存失效的条件
  - 9.3.3 何时使用缓存，何时不用
- 9.4 视图无效化
  - 9.4.1 `refresh()` 强制刷新
  - 9.4.2 `request_animation_frame()` 安排下一帧
  - 9.4.3 局部更新与全量刷新
- 9.5 视图导航
  - 9.5.1 `replace_root_view()` 切换页面
  - 9.5.2 路由模式实现
  - 9.5.3 多页面状态保持

### 第 10 章：View 组合模式
- 10.1 视图组合基础
  - 10.1.1 父 View 引用子 Entity
  - 10.1.2 子 View 通过回调通知父 View
  - 10.1.3 深层嵌套与 prop drilling
- 10.2 组件化模式
  - 10.2.1 可复用 UI 组件的定义
  - 10.2.2 组件参数传递
  - 10.2.3 槽模式 (slot pattern)
- 10.3 何时拆分视图
  - 10.3.1 性能考量：独立更新
  - 10.3.2 关注点分离
  - 10.3.3 过度拆分的代价
- 10.4 视图通信
  - 10.4.1 直接 Entity::update
  - 10.4.2 事件订阅
  - 10.4.3 闭包回调链
- 10.5 实战：构建一个可复用的模态框组件

---

## 第四部分：Elements

### 第 11 章：Element Trait
- 11.1 Element 是 GPUI 的渲染基石
  - 11.1.1 Element vs View 的区别
  - 11.1.2 命令式 vs 声明式
  - 11.1.3 Element 的生命周期
- 11.2 三阶段渲染
  - 11.2.1 `request_layout()` —— 请求布局
  - 11.2.2 `prepaint()` —— 布局确定后的准备
  - 11.2.3 `paint()` —— 实际绘制
  - 11.2.4 `RequestLayoutState` 和 `PrepaintState` 关联类型
- 11.3 `IntoElement` trait
  - 11.3.1 元素转换协议
  - 11.3.2 字符串、数字自动转元素
  - 11.3.3 `AnyElement` 类型擦除
- 11.4 `GlobalElementId`
  - 11.4.1 跨帧元素身份
  - 11.4.2 状态持久化
- 11.5 自定义 Element 的设计决策
  - 11.5.1 什么时候该自定义 Element
  - 11.5.2 与 div 组合的权衡
  - 11.5.3 性能影响

### 第 12 章：div 容器
- 12.1 `div()` 是什么
  - 12.1.1 HTML div 的类比
  - 12.1.2 `Div` struct 的结构
  - 12.1.3 `Interactivity` + `StyleRefinement`
- 12.2 FluentBuilder 模式
  - 12.2.1 方法链的流畅 API
  - 12.2.2 `FluentBuilder` trait
  - 12.2.3 链式调用的限制
- 12.3 子元素管理
  - 12.3.1 `ParentElement` trait
  - 12.3.2 `.child()` 添加单个子元素
  - 12.3.3 `.children()` 批量添加
  - 12.3.4 条件子元素：`.when()`
- 12.4 div 的能力概览
  - 12.4.1 布局：flex、grid、absolute
  - 12.4.2 样式：颜色、边框、阴影
  - 12.4.3 交互：点击、悬停、拖拽
  - 12.4.4 状态类：hover、active、focus
- 12.5 实用技巧
  - 12.5.1 `id()` 的作用与重要性
  - 12.5.2 `.group()` 组合样式
  - 12.5.3 `div` 嵌套深度的性能影响

### 第 13 章：文本元素
- 13.1 静态文本
  - 13.1.1 `"Hello"` 字面量直接作为元素
  - 13.1.2 `SharedString` 的性能优势
- 13.2 `StyledText`
  - 13.2.1 多文本段落
  - 13.2.2 `TextRun` 与样式跨度
  - 13.2.3 文本测量
- 13.3 `InteractiveText`
  - 13.3.1 可点击文本
  - 13.3.2 Tooltip 支持
  - 13.3.3 富文本交互

### 第 14 章：图片元素
- 14.1 `img()` 基础
  - 14.1.1 从资源加载
  - 14.1.2 从路径加载
  - 14.1.3 支持的格式：PNG、JPG、GIF、WebP、SVG、AVIF
- 14.2 `ImageSource` 枚举
  - 14.2.1 `Resource` —— 编译时嵌入
  - 14.2.2 `Render` —— 运行时生成
  - 14.2.3 `Image` —— 预加载的图片
  - 14.2.4 `Custom` —— 自定义加载
- 14.3 `ObjectFit` 模式
  - 14.3.1 Fill / Contain / Cover / ScaleDown / None
  - 14.3.2 选择正确的适配模式
- 14.4 图片加载与缓存
  - 14.4.1 异步加载流程
  - 14.4.2 错误处理与 fallback
  - 14.4.3 GIF 动画支持
- 14.5 图片元素与 `image_cache()`

### 第 15 章：SVG 元素
- 15.1 `svg()` 基础
  - 15.1.1 从内联路径加载
  - 15.1.2 从外部路径加载
- 15.2 SVG 变换
  - 15.2.1 `Transformation` struct
  - 15.2.2 缩放：`scale()`
  - 15.2.3 平移：`translate()`
  - 15.2.4 旋转：`rotate()`
  - 15.2.5 链式变换
- 15.3 SVG 与 img 的对比
  - 15.3.1 何时用 svg，何时用 img
  - 15.3.2 性能考量
  - 15.3.3 交互能力差异

### 第 16 章：Canvas 元素
- 16.1 `canvas()` 函数
  - 16.1.1 prepaint 闭包：布局准备
  - 16.1.2 paint 闭包：自定义绘制
- 16.2 绘制操作
  - 16.2.1 绘制 Quad（矩形）
  - 16.2.2 绘制 Path（路径）
  - 16.2.3 绘制 Sprite（贴图）
- 16.3 使用场景
  - 16.3.1 图表绘制
  - 16.3.2 自定义进度条
  - 16.3.3 粒子效果
- 16.4 Canvas vs 自定义 Element

### 第 17 章：自定义元素
- 17.1 什么时候需要自定义 Element
  - 17.1.1 div 无法满足的场景
  - 17.1.2 性能关键路径
  - 17.1.3 特殊布局需求
- 17.2 实现 Element trait
  - 17.2.1 定义 struct 与关联类型
  - 17.2.2 实现 `request_layout()`
  - 17.2.3 实现 `prepaint()`
  - 17.2.4 实现 `paint()`
  - 17.2.5 完整示例：自定义滚动条
- 17.3 自定义布局
  - 17.3.1 `AvailableSpace` 约束
  - 17.3.2 子元素布局委托
  - 17.3.3 内容裁剪与 `with_content_mask()`
- 17.4 交互支持
  - 17.4.1 注册事件处理器
  - 17.4.2 Hitbox 与命中检测
- 17.5 性能优化
  - 17.5.1 缓存布局计算结果
  - 17.5.2 减少状态传递
  - 17.5.3 惰性绘制

---

## 第五部分：布局系统

### 第 18 章：Flexbox 弹性布局
- 18.1 主轴与交叉轴
  - 18.1.1 `flex_row()` vs `flex_col()`
  - 18.1.2 反向布局：`flex_row_rev()`、`flex_col_rev()`
- 18.2 弹性项目伸缩
  - 18.2.1 `flex_grow()` —— 填充剩余空间
  - 18.2.2 `flex_shrink()` —— 空间不足时收缩
  - 18.2.3 `flex_basis()` —— 初始尺寸
  - 18.2.4 `flex_1()` 快捷方法
- 18.3 对齐与分布
  - 18.3.1 主轴：`justify_start/center/end/between/around/evenly`
  - 18.3.2 交叉轴：`items_start/center/end/stretch/baseline`
  - 18.3.3 单项目对齐：`self_start/center/end/stretch`
- 18.4 换行
  - 18.4.1 `flex_wrap()` 与 `flex_nowrap()`
  - 18.4.2 `flex_wrap_reverse`
- 18.5 间距
  - 18.5.1 `gap()` 统一间距
  - 18.5.2 `col_gap()` 与 `row_gap()`
- 18.6 实战：常见布局模式
  - 18.6.1 经典三栏布局
  - 18.6.2 居中卡片
  - 18.6.3 自适应表单布局

### 第 19 章：Grid 网格布局
- 19.1 网格基础
  - 19.1.1 `grid()` 激活网格
  - 19.1.2 `grid_cols()` 与 `grid_rows()` 定义轨道
  - 19.1.3 隐式网格与显式网格
- 19.2 网格放置
  - 19.2.1 `col_start()`、`col_end()`、`col_span()`
  - 19.2.2 `row_start()`、`row_end()`、`row_span()`
- 19.3 内容尺寸
  - 19.3.1 `min-content` 与 `max-content`
- 19.4 Grid vs Flex
  - 19.4.1 何时选 Grid
  - 19.4.2 何时选 Flex
  - 19.4.3 嵌套使用
- 19.5 实战：响应式数据表格

### 第 20 章：尺寸与间距
- 20.1 像素单位系统
  - 20.1.1 `Pixels` —— 逻辑像素
  - 20.1.2 `ScaledPixels` —— 物理像素（考虑 DPR）
  - 20.1.3 `DevicePixels` —— 设备像素（整数）
  - 20.1.4 `px()`、`sp()` 转换
- 20.2 相对单位
  - 20.2.1 `Rems` —— 根 em 单位
  - 20.2.2 `rems()` 构造函数
  - 20.2.3 与系统字体大小的关系
- 20.3 Length 类型
  - 20.3.1 `Absolute` vs `Relative` vs `Auto`
  - 20.3.2 `relative()` 分数长度
  - 20.3.3 `auto()` 自动尺寸
- 20.4 Margin 与 Padding
  - 20.4.1 `m()`、`mt()`、`mr()`、`mb()`、`ml()`
  - 20.4.2 `p()`、`pt()`、`pr()`、`pb()`、`pl()`
  - 20.4.3 `mx/my` 与 `px/py` 快捷方法
  - 20.4.4 负边距的可能性
- 20.5 溢出处理
  - 20.5.1 `overflow_hidden()` 裁剪
  - 20.5.2 `overflow_scroll()` 滚动
  - 20.5.3 `content_mask` 内部机制

### 第 21 章：Taffy 引擎内部
- 21.1 Taffy 简介
  - 21.1.1 什么是 Taffy
  - 21.1.2 GPUI 如何集成 Taffy
- 21.2 `LayoutId`
  - 21.2.1 不透明布局句柄
  - 21.2.2 布局请求传播
- 21.3 `AvailableSpace`
  - 21.3.1 `Definite` vs `MinContent` vs `MaxContent`
  - 21.3.2 约束传播机制
- 21.4 自定义布局集成
  - 21.4.1 `request_measured_layout()` 接入外部测量
  - 21.4.2 自定义布局算法
- 21.5 布局调试
  - 21.5.1 打印布局树
  - 21.5.2 性能分析

---

## 第六部分：样式

### 第 22 章：Styled Trait
- 22.1 `Styled` trait 概览
  - 22.1.1 Tailwind-like 的 Rust 实现
  - 22.1.2 `style() -> &mut StyleRefinement`
  - 22.1.3 方法链与 fluent API
- 22.2 状态驱动的样式
  - 22.2.1 hover 状态
  - 22.2.2 active 状态
  - 22.2.3 focus 状态
  - 22.2.4 selected 状态
  - 22.2.5 disabled 状态
- 22.3 分组样式
  - 22.3.1 `id()` 与元素标识
  - 22.3.2 `group()` 与组选择器
  - 22.3.3 跨组联动
- 22.4 条件样式
  - 22.4.1 `.when()` 方法
  - 22.4.2 `.when_some()` 方法
- 22.5 自定义样式扩展
  - 22.5.1 扩展 `StyleRefinement`
  - 22.5.2 自定义状态

### 第 23 章：颜色系统
- 23.1 `Hsla` 颜色模型
  - 23.1.1 色相、饱和度、亮度、透明度
  - 23.1.2 `hsla()` 构造函数
- 23.2 RGB 颜色
  - 23.2.1 `rgb()` 构造函数
  - 23.2.2 `rgba()` 带透明度
- 23.3 命名颜色
  - 23.3.1 `gpui::colors` 模块
  - 23.3.2 内置颜色常量
- 23.4 背景与填充
  - 23.4.1 `bg()` 方法
  - 23.4.2 渐变背景
- 23.5 外观 (Appearance) 系统
  - 23.5.1 `DefaultAppearance`
  - 23.5.2 `Colors` 主题色
  - 23.5.3 暗色/亮色切换

### 第 24 章：边框、阴影与变换
- 24.1 边框
  - 24.1.1 `border_1()` 到 `border_4()` 快捷方法
  - 24.1.2 边框颜色
  - 24.1.3 边框样式：`border_solid()`、`border_dashed()`、`border_dotted()`
  - 24.1.4 单边边框
- 24.2 圆角
  - 24.2.1 `rounded_none/sm/md/lg/xl/2xl/3xl/full`
  - 24.2.2 `Corners<T>` 自定义圆角
- 24.3 阴影
  - 24.3.1 `shadow_sm/md/lg/xl` 预设
  - 24.3.2 自定义阴影
- 24.4 透明度
  - 24.4.1 `opacity()` 方法
  - 24.4.2 性能影响
- 24.5 变换
  - 24.5.1 旋转
  - 24.5.2 缩放
  - 24.5.3 平移

### 第 25 章：文本样式
- 25.1 字体设置
  - 25.1.1 `font_family()` 字体系列
  - 25.1.2 `text_size()` 字号
  - 25.1.3 `text_xs/sm/base/lg/xl/2xl` 快捷方法
- 25.2 字重与字形
  - 25.2.1 `font_weight()` 字重
  - 25.2.2 `font_style()` 斜体
- 25.3 文本颜色与装饰
  - 25.3.1 `text_color()`
  - 25.3.2 `underline()` 下划线
  - 25.3.3 `strikethrough()` 删除线
- 25.4 文本排版
  - 25.4.1 `text_left/center/right/justify`
  - 25.4.2 `line_height()` 行高
  - 25.4.3 `tracking()` 字间距
  - 25.4.4 `leading()` 行间距
- 25.5 文本截断
  - 25.5.1 `truncate()` 省略号
  - 25.5.2 `overflow` 处理

---

## 第七部分：事件系统

### 第 26 章：鼠标事件
- 26.1 鼠标点击
  - 26.1.1 `MouseDownEvent` 与 `MouseUpEvent`
  - 26.1.2 `MouseClickEvent` 双击检测
  - 26.1.3 `MouseButton` 枚举
- 26.2 鼠标移动
  - 26.2.1 `MouseMoveEvent`
  - 26.2.2 拖拽检测
  - 26.2.3 `MouseExitEvent`
- 26.3 滚动
  - 26.3.1 `ScrollWheelEvent`
  - 26.3.2 `ScrollDelta`：Pixels vs Lines
- 26.4 压力感应
  - 26.4.1 `MousePressureEvent`
  - 26.4.2 `PressureStage`
- 26.5 文件拖放
  - 26.5.1 `FileDropEvent` 状态机
  - 26.5.2 Entered → Pending → Submit/Exited
- 26.6 事件注册
  - 26.6.1 `.on_mouse_down()` / `.on_mouse_up()`
  - 26.6.2 `.on_click()`
  - 26.6.3 `.on_hover()`
  - 26.6.4 `.on_scroll_wheel()`
  - 26.6.5 拖拽事件完整示例
- 26.7 Hitbox 与命中检测
  - 26.7.1 Hitbox 是什么
  - 26.7.2 自定义 Hitbox

### 第 27 章：键盘事件
- 27.1 `KeyDownEvent` 与 `KeyUpEvent`
- 27.2 `Keystroke` 解析
  - 27.2.1 键码表示
  - 27.2.2 修饰符组合
- 27.3 `ModifiersChangedEvent`
  - 27.3.1 `Modifiers` 状态
  - 27.3.2 修饰符追踪
- 27.4 键盘事件注册
  - 27.4.1 `.on_key_down()` / `.on_key_up()`
  - 27.4.2 `.on_modifiers_changed()`
- 27.5 键盘事件与 Action 的关系

### 第 28 章：手势事件
- 28.1 `PinchEvent` 捏合缩放
- 28.2 Touch 阶段追踪
- 28.3 平台特定手势
- 28.4 手势与缩放的组合

### 第 29 章：事件传播机制
- 29.1 事件传播阶段
  - 29.1.1 Capture 阶段（捕获）
  - 29.1.2 Bubble 阶段（冒泡）
  - 29.1.3 `DispatchPhase` 枚举
- 29.2 控制传播
  - 29.2.1 `cx.stop_propagation()`
  - 29.2.2 `cx.prevent_default()`
- 29.3 事件分发树
  - 29.3.1 事件路由算法
  - 29.3.2 Hitbox 在事件分发中的作用
- 29.4 事件传播实战
  - 29.4.1 模态框阻止底层点击
  - 29.4.2 嵌套列表的点击冲突
  - 29.4.3 拖拽时的鼠标捕获

---

## 第八部分：Action 与 Keymap

### 第 30 章：定义 Action
- 30.1 `Action` trait
  - 30.1.1 trait 定义与核心方法
  - 30.1.2 JSON 序列化与反序列化
- 30.2 `actions!()` 宏
  - 30.2.1 快速定义无参数 Action
  - 30.2.2 命名空间约定
- 30.3 `#[derive(Action)]`
  - 30.3.1 带参数的 Action
  - 30.3.2 JSON Schema 生成
- 30.4 `register_action!` 宏
  - 30.4.1 手动注册
  - 30.4.2 `#[register_action]` 属性宏
- 30.5 特殊 Action
  - 30.5.1 `NoAction`
  - 30.5.2 `Unbind`
- 30.6 Action 设计原则
  - 30.6.1 Action 应该是什么粒度
  - 30.6.2 Action 与业务逻辑的边界
  - 30.6.3 Action 命名约定

### 第 31 章：键绑定
- 31.1 `KeyBinding` 结构
  - 31.1.1 keystroke 字符串格式
  - 31.1.2 绑定到 Action
  - 31.1.3 上下文谓词
- 31.2 注册键绑定
  - 31.2.1 `cx.bind_keys()` 添加绑定
  - 31.2.2 批量注册
  - 31.2.3 覆盖与优先级
- 31.3 键绑定语法
  - 31.3.1 修饰符：cmd、ctrl、alt、shift
  - 31.3.2 键名：字母、数字、功能键
  - 31.3.3 组合键与序列
- 31.4 动态键绑定
  - 31.4.1 运行时修改绑定
  - 31.4.2 用户自定义快捷键
- 31.5 实战：实现完整的快捷键系统

### 第 32 章：Key Context
- 32.1 `KeyContext` 是什么
  - 32.1.1 key-value 对
  - 32.1.2 上下文栈
- 32.2 设置上下文
  - 32.2.1 `.key_context()` 在元素上
  - 32.2.2 `key_context()` 方法链
  - 32.2.3 动态更新
- 32.3 上下文谓词
  - 32.3.1 `KeyBindingContextPredicate` 枚举
  - 32.3.2 `Identifier`、`Equal`、`NotEqual`
  - 32.3.3 `Descendant`、`And`、`Or`、`Not`
  - 32.3.4 谓词匹配规则
- 32.4 实战：编辑器模式切换
  - 32.4.1 normal 模式 vs insert 模式
  - 32.4.2 不同模式的快捷键覆盖
  - 32.4.3 上下文动态切换

### 第 33 章：Action 分发
- 33.1 `on_action()` 注册
  - 33.1.1 在元素上注册
  - 33.1.2 Handler 签名
  - 33.1.3 多 handler 优先级
- 33.2 Action 解析流程
  - 33.2.1 按键 → Keystroke → Action
  - 33.2.2 绑定冲突解决
  - 33.2.3 上下文过滤
- 33.3 编程式触发
  - 33.3.1 `cx.dispatch_action()`
  - 33.3.2 从非 UI 代码触发 Action
- 33.4 Action 测试
  - 33.4.1 模拟按键输入
  - 33.4.2 验证 Action 被分发

---

## 第九部分：焦点与文本输入

### 第 34 章：焦点系统
- 34.1 `FocusHandle`
  - 34.1.1 创建：`cx.focus_handle()`
  - 34.1.2 `FocusId` 唯一标识
  - 34.1.3 弱引用与生命周期
- 34.2 焦点追踪
  - 34.2.1 `track_focus()` 在元素上
  - 34.2.2 焦点变化回调
  - 34.2.3 `FocusOutEvent`
- 34.3 焦点管理
  - 34.3.1 `cx.focus()` 设置焦点
  - 34.3.2 `cx.blur()` 移除焦点
  - 34.3.3 `is_focused()` 检查焦点
- 34.4 Tab 导航
  - 34.4.1 `TabStopMap` 焦点顺序
  - 34.4.2 焦点分组
  - 34.4.3 自定义 Tab 顺序
- 34.5 实战：表单焦点管理

### 第 35 章：文本输入与 IME
- 35.1 `EntityInputHandler` trait
  - 35.1.1 trait 方法概览
  - 35.1.2 `text_for_range()` 获取文本
  - 35.1.3 `replace_text_in_range()` 替换文本
  - 35.1.4 `selected_text_range()` 选区
- 35.2 `ElementInputHandler`
  - 35.2.1 关联 View 与输入处理
  - 35.2.2 注册与注销
- 35.3 IME 支持
  - 35.3.1 标记文本 (marked text)
  - 35.3.2 `marked_text_range()`
  - 35.3.3 `replace_and_mark_text_in_range()`
- 35.4 文本范围操作
  - 35.4.1 UTF-8 vs UTF-16 范围
  - 35.4.2 `bounds_for_range()` 光标定位
  - 35.4.3 `character_index_for_point()` 点击定位
- 35.5 实战：实现一个简单的文本输入框

---

## 第十部分：异步编程

### 第 36 章：前台执行器
- 36.1 `ForegroundExecutor`
  - 36.1.1 主线程异步
  - 36.1.2 为什么需要前台执行器
- 36.2 `cx.spawn()`
  - 36.2.1 从 Entity 启动异步任务
  - 36.2.2 `WeakEntity` 检查防止悬垂
  - 36.2.3 任务返回值
- 36.3 `Task<T>`
  - 36.3.1 GPUI 的任务类型
  - 36.3.2 `Task::detach()` 分离任务
  - 36.3.3 `Task::shared()` 共享任务
- 36.4 View 生命周期与任务
  - 36.4.1 View 销毁时任务自动取消
  - 36.4.2 任务中的 `cx.notify()`
  - 36.4.3 任务冲突与取消策略
- 36.5 实战：异步加载数据并更新 UI

### 第 37 章：后台执行器
- 37.1 `BackgroundExecutor`
  - 37.1.1 后台线程池
  - 37.1.2 `cx.background_spawn()`
- 37.2 线程安全
  - 37.2.1 `Send + Sync` 要求
  - 37.2.2 不能直接操作 UI
  - 37.2.3 数据传递模式
- 37.3 工具方法
  - 37.3.1 `timer()` 定时
  - 37.3.2 `block_on()` 阻塞等待
  - 37.3.3 `scoped()` 作用域任务
- 37.4 `FallibleTask`
  - 37.4.1 优雅取消
  - 37.4.2 错误传播
- 37.5 前台与后台的协作模式

### 第 38 章：Task 管理
- 38.1 Task 生命周期
  - 38.1.1 创建 → 运行 → 完成/取消
  - 38.1.2 Task drop 时的行为
- 38.2 多任务协调
  - 38.2.1 任务串行化
  - 38.2.2 任务并行化
  - 38.2.3 任务队列
- 38.3 `Priority` 枚举
  - 38.3.1 任务优先级
  - 38.3.2 调度策略
- 38.4 错误处理
  - 38.4.1 `Result` 在 Task 中的传播
  - 38.4.2 UI 中的错误展示
- 38.5 实战：可取消的搜索请求

---

## 第十一部分：状态模式

### 第 39 章：Observe 模式
- 39.1 `cx.observe()`
  - 39.1.1 观察外部 Entity
  - 39.1.2 回调签名与触发条件
  - 39.1.3 回调中的 `cx.notify()`
- 39.2 `cx.observe_self()`
  - 39.2.1 自观察模式
  - 39.2.2 使用场景
- 39.3 `cx.observe_global()`
  - 39.3.1 观察全局状态变化
  - 39.3.2 自动响应主题切换
- 39.4 Observer 回调生命周期
  - 39.4.1 回调何时被调用
  - 39.4.2 回调的清理
- 39.5 性能考量
  - 39.5.1 过多 observe 的影响
  - 39.5.2 批量更新的优化

### 第 40 章：Subscribe 模式
- 40.1 `EventEmitter<E>` trait
  - 40.1.1 定义事件类型
  - 40.1.2 实现 trait
- 40.2 `cx.emit()`
  - 40.2.1 发射事件
  - 40.2.2 事件类型的设计
- 40.3 `cx.subscribe()`
  - 40.3.1 订阅其他 Entity 的事件
  - 40.3.2 Handler 签名
  - 40.3.3 取消订阅
- 40.4 `cx.subscribe_self()`
- 40.5 Event vs Observe
  - 40.5.1 什么时候用 emit/subscribe
  - 40.5.2 什么时候用 observe
  - 40.5.3 混合使用
- 40.6 实战：事件驱动的通知系统

### 第 41 章：Global 状态
- 41.1 `Global` marker trait
  - 41.1.1 标记全局单例
  - 41.1.2 类型安全的保证
- 41.2 `cx.set_global()` 与 `cx.global()`
  - 41.2.1 设置全局状态
  - 41.2.2 读取全局状态
  - 41.2.3 `cx.try_global()` 安全访问
- 41.3 `cx.update_global()`
  - 41.3.1 更新全局状态
  - 41.3.2 闭包签名
- 41.4 `ReadGlobal` 与 `UpdateGlobal` trait
  - 41.4.1 便捷访问方法
  - 41.4.2 实现 trait
- 41.5 全局状态模式
  - 41.5.1 主题配置
  - 41.5.2 应用设置
  - 41.5.3 全局服务（日志、遥测）
- 41.6 Global 的限制与替代方案

### 第 42 章：Subscription 生命周期
- 42.1 `Subscription` struct
  - 42.1.1 RAII 模式
  - 42.1.2 Drop 时自动取消
- 42.2 `Subscription::detach()`
  - 42.2.1 手动解除关联
  - 42.2.2 延长生命周期
- 42.3 `Subscription::join()`
  - 42.3.1 组合多个订阅
  - 42.3.2 统一生命周期管理
- 42.4 内存考量
  - 42.4.1 `SubscriberSet` 内部结构
  - 42.4.2 泄漏检测
  - 42.4.3 大量订阅的性能

---

## 第十二部分：高级元素

### 第 43 章：List 虚拟列表
- 43.1 为什么需要虚拟列表
  - 43.1.1 大量数据的性能问题
  - 43.1.2 视口渲染原理
- 43.2 `ListState`
  - 43.2.1 创建与配置
  - 43.2.2 渲染项回调
  - 43.2.3 状态管理
- 43.3 `list()` 元素
  - 43.3.1 基本用法
  - 43.3.2 滚动集成
- 43.4 可变高度项
  - 43.4.1 高度估算
  - 43.4.2 测量与修正
- 43.5 数据更新
  - 43.5.1 `splice()` 局部更新
  - 43.5.2 `reset()` 全量重置
- 43.6 实战：大型数据列表

### 第 44 章：UniformList
- 44.1 何时使用 UniformList
  - 44.1.1 所有项高度相同
  - 44.1.2 比 List 更简单高效
- 44.2 `uniform_list()` 函数
  - 44.2.1 基本用法
  - 44.2.2 项渲染回调
- 44.3 滚动控制
  - 44.3.1 `UniformListScrollHandle`
  - 44.3.2 `scroll_to_item()` 滚动到指定项
  - 44.3.3 `ScrollStrategy`：Top/Center/Bottom/Nearest
- 44.4 装饰器
  - 44.4.1 `UniformListDecoration` trait
  - 44.4.2 自定义行样式

### 第 45 章：Anchored 弹出层
- 45.1 `anchored()` 函数
  - 45.1.1 锚点定位原理
  - 45.1.2 基本用法
- 45.2 `Anchored` 元素
  - 45.2.1 锚点方向
  - 45.2.2 偏移量
- 45.3 适配模式
  - 45.3.1 `AnchoredFitMode`
  - 45.3.2 `SnapToWindow`
  - 45.3.3 `SnapToWindowWithMargin`
  - 45.3.4 `SwitchAnchor` 翻转锚点
- 45.4 定位模式
  - 45.4.1 `Window` 模式
  - 45.4.2 `Local` 模式
- 45.5 实战：下拉菜单、Tooltip、Popover

### 第 46 章：Deferred 延迟渲染
- 46.1 `deferred()` 函数
  - 46.1.1 延迟布局与绘制
  - 46.1.2 渲染在正常内容之上
- 46.2 `Deferred` 元素
  - 46.2.1 `with_priority()` 设置优先级
  - 46.2.2 优先级排序
- 46.3 使用场景
  - 46.3.1 Tooltip
  - 46.3.2 Context Menu
  - 46.3.3 Modal
  - 46.3.4 拖拽预览
- 46.4 多层 Deferred 的排序

### 第 47 章：动画系统
- 47.1 `Animation` struct
  - 47.1.1 `duration` 时长
  - 47.1.2 `oneshot` 单次播放
  - 47.1.3 `easing` 缓动函数
- 47.2 `AnimationExt` trait
  - 47.2.1 `with_animation()` 创建动画元素
  - 47.2.2 `with_animations()` 多动画组合
- 47.3 内置缓动函数
  - 47.3.1 `linear()` 线性
  - 47.3.2 `quadratic()` 二次
  - 47.3.3 `ease_in_out()` 先加速后减速
  - 47.3.4 `ease_out_quint()` 快速开始缓慢结束
  - 47.3.5 `bounce()` 弹跳
  - 47.3.6 `pulsating_between()` 脉冲
- 47.4 自定义缓动函数
- 47.5 动画状态管理
  - 47.5.1 触发动画
  - 47.5.2 动画完成回调
  - 47.5.3 动画取消与切换
- 47.6 实战：平滑展开/折叠面板

---

## 第十三部分：文本系统深入

### 第 48 章：字体管理
- 48.1 `Font` struct
  - 48.1.1 字体描述符
  - 48.1.2 `FontFamily`、`FontWeight`、`FontStyle`
- 48.2 字体加载
  - 48.2.1 系统字体
  - 48.2.2 自定义字体
  - 48.2.3 `FontId` 标识符
- 48.3 `FontMetrics`
  - 48.3.1 ascent、descent
  - 48.3.2 cap_height、x_height
- 48.4 字体特性
  - 48.4.1 OpenType 特性
  - 48.4.2 `FontFeatures`
- 48.5 字体回退
  - 48.5.1 `FontFallbacks` 链
  - 48.5.2 多语言支持

### 第 49 章：文本塑形
- 49.1 `TextSystem` 架构
  - 49.1.1 全局文本系统
  - 49.1.2 `WindowTextSystem` 每窗口实例
- 49.2 `ShapedGlyph` 与 `ShapedRun`
  - 49.2.1 Glyph 度量
  - 49.2.2 子像素渲染变体
- 49.3 Glyph 栅格化
  - 49.3.1 缓存机制
  - 49.3.2 Atlas 纹理
- 49.4 文本测量
  - 49.4.1 性能优化
  - 49.4.2 缓存策略

### 第 50 章：行布局
- 50.1 `LineWrapper`
  - 50.1.1 换行算法
  - 50.1.2 可用空间约束
- 50.2 `WrappedLine` 与 `WrappedLineLayout`
  - 50.2.1 布局结果
  - 50.2.2 光标位置计算
- 50.3 UTF-8 vs UTF-16
  - 50.3.1 范围转换
  - 50.3.2 与 IME 的关系
- 50.4 双向文本支持

---

## 第十四部分：渲染管线

### 第 51 章：场景图
- 51.1 `Scene` struct
  - 51.1.1 PaintOperation 集合
  - 51.1.2 层级结构
- 51.2 `Primitive` 枚举
  - 51.2.1 `Quad` —— 矩形
  - 51.2.2 `Shadow` —— 阴影
  - 51.2.3 `Path` —— 路径
  - 51.2.4 `Underline` —— 下划线
  - 51.2.5 `MonochromeSprite` —— 单色贴图
  - 51.2.6 `SubpixelSprite` —— 子像素贴图
  - 51.2.7 `PolychromeSprite` —— 彩色贴图
  - 51.2.8 `Surface` —— 平台表面
- 51.3 `PrimitiveBatch` GPU 批处理
- 51.4 `TransformationMatrix` 2D 变换
  - 51.4.1 unit、translate、rotate、scale
- 51.5 图层管理
  - 51.5.1 内容裁剪
  - 51.5.2 Bounds tree 空间索引

### 第 52 章：GPU 渲染管线
- 52.1 从 Scene 到屏幕
  - 52.1.1 渲染管线概述
  - 52.1.2 着色器
- 52.2 Atlas 纹理
  - 52.2.1 Glyph 缓存
  - 52.2.2 纹理分配
- 52.3 Sprite 渲染
  - 52.3.1 贴图采样
  - 52.3.2 亚像素渲染
- 52.4 Surface 合成
  - 52.4.1 macOS CoreVideo
  - 52.4.2 Vulkan/Metal 合成

### 第 53 章：性能优化
- 53.1 视图缓存策略
  - 53.1.1 何时缓存、何时不缓存
  - 53.1.2 缓存失效控制
- 53.2 最小化重渲染
  - 53.2.1 精确的 `cx.notify()`
  - 53.2.2 observe 的粒度控制
- 53.3 `Window::refresh()` 控制
- 53.4 `request_animation_frame()`
- 53.5 profiler 模块
  - 53.5.1 线程性能追踪
  - 53.5.2 Task 计时
- 53.6 性能 checklist

---

## 第十五部分：平台

### 第 54 章：Platform Trait
- 54.1 `Platform` trait 概览
  - 54.1.1 平台抽象设计
  - 54.1.2 平台特定实现
- 54.2 Feature Flags
  - 54.2.1 `wayland`、`x11`
  - 54.2.2 `screen-capture`
- 54.3 平台能力检测

### 第 55 章：macOS 后端
- 55.1 Metal 渲染
- 55.2 原生装饰与菜单栏
- 55.3 触控板手势

### 第 56 章：Linux 后端
- 56.1 Vulkan 渲染
- 56.2 Wayland vs X11
- 56.3 Compositor 检测
- 56.4 Layer Shell 支持

---

## 第十六部分：测试

### 第 57 章：#[gpui::test]
- 57.1 宏用法
  - 57.1.1 基本用法
  - 57.1.2 测试函数签名
- 57.2 测试配置
  - 57.2.1 SEED 环境变量
  - 57.2.2 测试超时

### 第 58 章：TestAppContext
- 58.1 `TestAppContext` API
- 58.2 模拟输入事件
- 58.3 `TestDispatcher` 确定性调度
- 58.4 多上下文测试（协作 UI）

### 第 59 章：视觉测试
- 59.1 视觉测试框架 (macOS)
- 59.2 截图比对
- 59.3 `VisualTestContext`

### 第 60 章：属性测试
- 60.1 `#[gpui::property_test]` 宏
- 60.2 Proptest 集成
- 60.3 种子控制与重现

---

## 第十七部分：调试

### 第 61 章：Element Inspector
- 61.1 Inspector 设置
- 61.2 元素拾取模式
- 61.3 `InspectorElementId` 与源码位置
- 61.4 自定义 Inspector 渲染器

### 第 62 章：调试样式
- 62.1 `.debug()` 边框
- 62.2 `.debug_below()` 子树调试
- 62.3 可视化调试技巧

### 第 63 章：性能分析
- 63.1 profiler 模块使用
- 63.2 帧时间分析
- 63.3 `GPUI_MANIFEST_DIR` 与构建信息

---

## 第十八部分：Cookbook

### 第 64 章：Hello World
- 最简 GPUI 应用
- 窗口居中显示文本

### 第 65 章：计数器应用
- Entity 状态管理
- 按钮点击处理
- `cx.notify()` 模式

### 第 66 章：Todo 应用
- List 虚拟列表渲染
- 文本输入创建新项
- 删除与完成操作
- 键盘快捷键

### 第 67 章：简单文本编辑器
- 文本输入处理
- 焦点管理
- 快捷键系统
- 基本编辑操作

### 第 68 章：图片画廊
- Grid 布局
- 图片懒加载
- 缩放与平移
- 动画过渡

### 第 69 章：数据仪表盘
- 多窗口管理
- Tab 导航
- 分割面板
- 异步实时数据
- 完整的 Action/Keymap 系统

---

## 附录

### 附录 A：GPUI 宏一览
- `actions!()` 宏
- `register_action!` 宏
- `#[derive(Action)]`
- `#[gpui::test]`
- `#[gpui::property_test]`
- `#[register_action]`
- `derive(IntoElement)`

### 附录 B：常用 API 速查
- Entity 操作速查
- Context 方法速查
- Styled 方法速查
- div 事件注册速查
- 动画与缓动函数速查

### 附录 C：Zed 中的 GPUI 实战
- Zed 的 View 架构分析
- 编辑器核心组件拆解
- 从 Zed 源码学习的最佳实践
