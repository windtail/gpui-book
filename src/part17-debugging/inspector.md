# 第 61 章：Element Inspector

GPUI 内置 Element Inspector，允许运行时检查和调试 UI 元素树。

## 61.1 Inspector 设置

Inspector 在 `debug_assertions` 或启用 `inspector` feature 时可用：

```rust
#[cfg(any(feature = "inspector", debug_assertions))]
pub mod conditional {
    // Inspector 实现
}
```

### 启用 Inspector

```rust
// 应用中注册 inspector 渲染器
cx.set_inspector_renderer(|inspector, window, cx| {
    // 返回自定义的 inspector UI
    inspector_panel(inspector, window, cx)
});
```

## 61.2 InspectorElementId

每个可检查的元素有唯一标识：

```rust
#[derive(Debug, Eq, PartialEq, Hash, Clone)]
pub struct InspectorElementId {
    /// 稳定部分（路径）
    pub path: Rc<InspectorElementPath>,
    /// 区分同路径的多个实例
    pub instance_id: usize,
}
```

### InspectorElementPath

路径包含全局元素 ID 和源码位置：

```rust
pub struct InspectorElementPath {
    /// 祖先元素的 GlobalElementId
    pub global_id: GlobalElementId,
    /// 元素构建时的源码位置
    pub source_location: &'static panic::Location<'static>,
}
```

源码位置让开发者可以直接定位到产生该元素的代码行。

## 61.3 元素拾取模式

Inspector 有两种模式：悬停和选择。

### Picking 模式

```rust
impl Inspector {
    /// 开始元素拾取模式
    pub fn start_picking(&mut self) {
        self.pick_depth = Some(0.0);
    }

    /// 是否在拾取模式
    pub fn is_picking(&self) -> bool {
        self.pick_depth.is_some()
    }
}
```

在拾取模式下，鼠标悬停在元素上会高亮显示。

### 选择元素

```rust
/// 选择一个元素（固定检查）
pub fn select(&mut self, id: InspectorElementId, window: &mut Window) {
    self.set_active_element_id(id, window);
    self.pick_depth = None;  // 退出拾取模式
}

/// 悬停到一个元素（临时高亮）
pub fn hover(&mut self, id: InspectorElementId, window: &mut Window) {
    if self.is_picking() {
        let changed = self.set_active_element_id(id, window);
        if changed {
            self.pick_depth = Some(0.0);
        }
    }
}
```

`select` 锁定检查目标，`hover` 只在拾取模式下响应。

## 61.4 活跃元素状态

Inspector 为选中的元素维护类型安全的状态存储：

```rust
pub fn with_active_element_state<T: 'static, R>(
    &mut self,
    window: &mut Window,
    f: impl FnOnce(&mut Option<T>, &mut Window) -> R,
) -> R {
    let Some(active_element) = &mut self.active_element else {
        return f(&mut None, window);
    };

    let type_id = TypeId::of::<T>();
    let mut inspector_state = active_element
        .states
        .remove(&type_id)
        .map(|state| *state.downcast().unwrap());

    let result = f(&mut inspector_state, window);

    // 状态写回
    if let Some(inspector_state) = inspector_state {
        active_element.states.insert(type_id, Box::new(inspector_state));
    }

    result
}
```

状态以 `TypeId` 为 key 存储在 `FxHashMap` 中，支持任意类型的扩展状态。

## 61.5 自定义 Inspector 渲染器

### InspectorRenderer

```rust
pub type InspectorRenderer =
    Box<dyn Fn(&mut Inspector, &mut Window, &mut Context<Inspector>) -> AnyElement>;
```

应用通过注册渲染器自定义 inspector 面板：

```rust
impl Render for Inspector {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        if let Some(inspector_renderer) = cx.inspector_renderer.take() {
            let result = inspector_renderer(self, window, cx);
            cx.inspector_renderer = Some(inspector_renderer);
            result
        } else {
            Empty.into_any_element()
        }
    }
}
```

没有注册渲染器时显示空元素。

### InspectorElementRegistry

注册类型特定的 inspector 渲染器：

```rust
pub(crate) struct InspectorElementRegistry {
    renderers_by_type_id: FxHashMap<
        TypeId,
        Box<dyn Fn(InspectorElementId, &dyn Any, &mut Window, &mut App) -> AnyElement>,
    >,
}

impl InspectorElementRegistry {
    pub fn register<T: 'static, R: IntoElement>(
        &mut self,
        f: impl Fn(InspectorElementId, &T, &mut Window, &mut App) -> R,
    ) {
        self.renderers_by_type_id.insert(
            TypeId::of::<T>(),
            Box::new(move |id, value, window, cx| {
                let value = value.downcast_ref().unwrap();
                f(id, value, window, cx).into_any_element()
            }),
        );
    }
}
```

注册后，选中元素的对应类型状态会自动使用注册的渲染器显示。

### 渲染活跃状态

```rust
pub fn render_inspector_states(
    &mut self,
    window: &mut Window,
    cx: &mut Context<Self>,
) -> Vec<AnyElement> {
    let mut elements = Vec::new();
    if let Some(active_element) = self.active_element.take() {
        for (type_id, state) in &active_element.states {
            if let Some(render_inspector) = cx
                .inspector_element_registry
                .renderers_by_type_id
                .remove(type_id)
            {
                let mut element = (render_inspector)(
                    active_element.id.clone(),
                    state.as_ref(),
                    window,
                    cx,
                );
                elements.push(element);
                // 重新注册渲染器
                cx.inspector_element_registry
                    .renderers_by_type_id
                    .insert(*type_id, render_inspector);
            }
        }
        self.active_element = Some(active_element);
    }
    elements
}
```

## 61.6 Inspector 实战

### 检查元素树

```rust
// 选中一个元素后，inspector 显示：
// - 元素类型（div, text, img 等）
// - 源码位置（点击跳转到代码）
// - 当前样式
// - 布局信息（bounds, content_size）
```

### 自定义元素检查器

```rust
// 为自定义类型注册 inspector
cx.inspector_element_registry.register::<MyCustomState, _>(
    |id, state, window, cx| {
        div()
            .child(format!("Element: {:?}", id))
            .child(format!("State: {:?}", state))
            .into_any_element()
    },
);
```

## 61.7 inspector_reflection

GPUI 提供反射支持用于 inspector 功能：

```rust
pub mod inspector_reflection {
    pub struct FunctionReflection<T> {
        pub name: &'static str,
        pub function: fn(Box<dyn Any>) -> Box<dyn Any>,
        pub documentation: Option<&'static str>,
        pub _type: PhantomData<T>,
    }

    impl<T: 'static> FunctionReflection<T> {
        pub fn invoke(&self, value: T) -> T {
            let boxed = Box::new(value) as Box<dyn Any>;
            let result = (self.function)(boxed);
            *result.downcast::<T>().expect("Type mismatch")
        }
    }
}
```

通过 `#[derive(InspectorReflection)]`（或手动实现）为类型生成反射信息，使 inspector 可以动态调用方法、读取字段。
