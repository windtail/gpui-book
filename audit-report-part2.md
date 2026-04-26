# GPUI Book Audit Report: Chapters 36-69 + Appendices

Audited against GPUI source code at `/home/ljj/work/zed/crates/gpui/src/`.

---

## Chapter 36: Foreground Executor (`src/part10-async/foreground.md`)

### OK
- `ForegroundExecutor` / `cx.spawn()` / `Task<T>` descriptions are accurate.
- `WeakEntity` safety pattern (`view.update(&mut cx, |this, cx| ...).ok()`) is correct.
- `Task::detach()` and lifecycle management descriptions are accurate.
- All code examples use correct `cx: &mut Context<Self>` signatures.

---

## Chapter 37: Background Executor (`src/part10-async/background.md`)

### OK
- `BackgroundExecutor` vs `ForegroundExecutor` distinction is accurate.
- `Send` requirements for background tasks correctly described.
- `timer()`, `scoped()`, `block_on()`, `FallibleTask` descriptions match source.
- Front/back coordination pattern is accurately described.

---

## Chapter 38: Task (`src/part10-async/task.md`)

### OK
- `Task` lifecycle description is accurate.
- Serial/parallel task coordination patterns are correct.
- `Priority` enum (`Low`, `High`, `RealtimeAudio`) matches source.
- Error handling with `Task<Result<T, E>>` is correct.

---

## Chapter 39: Observe (`src/part11-state/observe.md`)

### OK
- `cx.observe()`, `cx.observe_self()`, `cx.observe_global::<T>()` signatures match source.
- Callback signatures are correct.
- Performance considerations are well described.

---

## Chapter 40: Subscribe (`src/part11-state/subscribe.md`)

### OK
- `EventEmitter<E>` marker trait description is accurate.
- `cx.emit()`, `cx.subscribe()`, `cx.subscribe_self()` signatures match source.
- Event vs Observe comparison table is correct.

---

## Chapter 41: Global (`src/part11-state/global.md`)

### OK
- `Global` marker trait, `cx.set_global()`, `cx.global()`, `cx.try_global()`, `cx.update_global()` all accurate.
- `ReadGlobal`/`UpdateGlobal` blanket implementations correctly described.

---

## Chapter 42: Subscription (`src/part11-state/subscription.md`)

### OK
- `Subscription` RAII pattern with `unsubscribe: Option<Box<dyn FnOnce()>>` matches source.
- `detach()`, `join()` descriptions are correct.
- `SubscriberSet` internal structure description is plausible.

---

## Chapter 43: List (`src/part12-advanced-elements/list.md`)

### OK
- `ListState::new(item_count, alignment, overdraw)` signature matches source.
- `list(state, render_item)` with `FnMut(usize, &mut Window, &mut App) -> AnyElement` is correct.
- `splice()`, `reset()` descriptions are accurate.
- `SumTree` for height tracking is correctly described.

---

## Chapter 44: Uniform List (`src/part12-advanced-elements/uniform-list.md`)

### OK
- `uniform_list(id, count, render_fn)` description is accurate.
- `UniformListScrollHandle` and `scroll_to_item()` are correct.
- `ScrollStrategy` (Top/Center/Bottom/Nearest) matches source.

---

## Chapter 45: Anchored (`src/part12-advanced-elements/anchored.md`)

### OK
- `anchored()` and `Anchor` enum descriptions are accurate.
- Fit modes and position modes are correctly described.

---

## Chapter 46: Deferred (`src/part12-advanced-elements/deferred.md`)

### OK
- `deferred(child)` and `with_priority()` descriptions match source.
- Draw deferral mechanism via `window.defer_draw()` is accurately described.

---

## Chapter 47: Animation (`src/part12-advanced-elements/animation.md`)

### OK
- `Animation::new(duration)`, `.repeat()`, `.with_easing()` signatures are correct.
- `with_animation(id, animation, closure)` is accurate.
- Easing functions (`linear`, `ease_in_out`, `ease_out_quint`, `bounce`, `pulsating_between`) match source.

---

## Chapter 48: Fonts (`src/part13-text-system/fonts.md`)

### OK
- `Font` struct, `FontWeight` constants, `FontStyle` (Normal, Italic, Oblique) all correct.
- `FontId`, `FontMetrics`, `FontFeatures`, `FontFallbacks` descriptions are accurate.
- Font resolution with fallback stack is correctly described.

---

## Chapter 49: Shaping (`src/part13-text-system/shaping.md`)

### OK
- `TextSystem` architecture description is plausible.
- `ShapedGlyph` with `index: usize` (UTF-8 byte index) is correct.
- Glyph rasterization, Atlas textures, subpixel variants description is accurate.

---

## Chapter 50: Line Layout (`src/part13-text-system/line-layout.md`)

### OK
- `LineWrapper`, `WrappedLine`, `LineLayoutCache` descriptions are accurate.
- UTF-8/UTF-16 conversion patterns are correct.
- Bidi support via `rustybuzz` is correctly mentioned.

---

## Chapter 51: Scene (`src/part14-rendering/scene.md`)

### OK
- `Scene` struct with typed Vecs description is accurate.
- `Primitive` enum order (Shadow first, then Quad) matches source.
- `TransformationMatrix` with `rotation_scale: [[f32; 2]; 2]` and `translation: [f32; 2]` is correct.
- `BoundsTree`, `PrimitiveBatch` descriptions are accurate.

---

## Chapter 52: GPU Rendering (`src/part14-rendering/gpu.md`)

### ERROR
1. **Linux rendering backend** (multiple places):
   - Book: "Linux 使用 Vulkan API 进行渲染", "macOS 使用 Metal，Linux 使用 Vulkan"
   - Source: Linux uses **wgpu** (gpui_linux uses gpui_wgpu/WgpuRenderer), NOT Vulkan.
   - This is the same recurring error from chapters 1-3 (critical error #1 in part1 report).

---

## Chapter 53: Performance (`src/part14-rendering/performance.md`)

### OK
- `AnyView::cached()`, `cx.notify()` precision, `window.refresh()` descriptions are accurate.
- `request_animation_frame()` is correct.
- Profiler module with `TaskTiming`, `ThreadTaskTimings`, `ProfilingCollector` matches source.
- Ring buffer (16MiB) description is correct.

---

## Chapter 54: Platform (`src/part15-platform/platform.md`)

### OK
- Full `Platform` trait methods description is accurate.
- `guess_compositor()`, `ThermalState`, `WindowDecorations`, `Tiling` struct are correct.
- Feature flags (wayland, x11, screen-capture) are accurately described.

---

## Chapter 55: macOS (`src/part15-platform/macos.md`)

### OK
- Metal rendering pipeline description is correct.
- Native menus, dock menu, trackpad gestures, pressure sensing descriptions are plausible.
- Native file dialogs and find pasteboard are accurately described.

---

## Chapter 56: Linux (`src/part15-platform/linux.md`)

### ERROR
1. **Linux GPU backend** (recurring error):
   - Book: "使用 Vulkan 作为 GPU 后端" with Vulkan code examples
   - Source: Linux uses **wgpu**, not Vulkan.
   - Same root cause as chapters 1, 2, 3, 52.

### OK
- Wayland vs X11, `guess_compositor()`, layer shell, headless mode descriptions are accurate.
- XDG portals, primary selection descriptions are correct.

---

## Chapter 57: Testing (`src/part16-testing/test.md`)

### OK
- `#[gpui::test]` macro, sync/async test signatures are correct.
- Multi-context tests, `TestDispatcher`, SEED/ITERATIONS env vars are accurate.
- Retry mechanism description is plausible.

---

## Chapter 58: Test App (`src/part16-testing/test-app.md`)

### OK
- `TestApp` struct, `TestApp::new()`, `TestApp::with_seed()` are correct.
- `update()`/`read()`, entity shortcut methods are accurate.
- Input simulation (`simulate_keystroke`, `simulate_click`, etc.) is correct.

---

## Chapter 59: Visual Testing (`src/part16-testing/visual.md`)

### OK
- Visual testing framework (macOS only) description is accurate.
- `VisualTestContext`, screenshot capture, tolerance-based comparison are correct.
- `UPDATE_VISUAL_TESTS` env var for baseline management is correct.

---

## Chapter 60: Property Testing (`src/part16-testing/property.md`)

### OK
- `#[gpui::property_test]` macro and proptest integration are accurately described.
- `seed_strategy()`, `apply_seed_to_proptest_config()` match source.

---

## Chapter 61: Inspector (`src/part17-debugging/inspector.md`)

### OK
- `Inspector`, `InspectorElementId` with `path` and `instance_id` are correct.
- Picking modes, `with_active_element_state<T>()` are accurately described.
- `InspectorElementRegistry`, `inspector_reflection` descriptions match source.

---

## Chapter 62: Debug Styling (`src/part17-debugging/debug-styling.md`)

### OK
- `.debug()` and `.debug_below()` methods are correctly described.
- `DebuggingStyle` struct with `color` and `label` fields matches source.
- Zero-cost in release builds via `#[cfg(debug_assertions)]` is correct.

---

## Chapter 63: Profiling (`src/part17-debugging/profiling.md`)

### OK
- `TaskTiming`, `ThreadTaskTimings`, `ProfilingCollector` structures match source.
- Ring buffer size calculation `(16 * 1024 * 1024) / size_of::<TaskTiming>()` is correct.
- Serialization to JSON is accurate.

---

## Chapter 64: Hello World (`src/part18-cookbook/hello-world.md`)

### OK
- `Application::new()`, `app.run()`, `cx.open_window()` pattern is correct.
- `Render` trait with `cx: &mut Context<Self>` is correct.
- `div()` container with flexbox centering is accurate.

### ERROR
1. **Application::new() description** (line 65):
   - Book: "初始化平台后端（macOS 用 Metal，Linux 用 Vulkan）"
   - Source: Linux uses **wgpu**, NOT Vulkan.
   - Same recurring backend error.

---

## Chapter 65: Counter (`src/part18-cookbook/counter.md`)

### OK
- `cx.listener()`, `cx.notify()` pattern is correct.
- Four-parameter order (`this`, event, `window`, `cx`) is accurate.
- State management with `count: i32` is correct.
- `button("id", "label")` and `.on_click()` usage is accurate.

---

## Chapter 66: Todo App (`src/part18-cookbook/todo.md`)

### OK
- `ListState::new(0, ListAlignment::Top, px(400.))` signature matches source.
- `ListState::splice(range, new_count)` usage is correct.
- `list(state, render_item)` pattern is accurate.
- `cx.focus_handle()`, `track_focus()` usage is correct.
- `.when(condition, |div| ...)` conditional styling is correct.
- `cx.listener` capturing `index` in closures works correctly.

---

## Chapter 67: Text Editor (`src/part18-cookbook/editor.md`)

### OK
- `actions!` macro usage is correct.
- `EntityInputHandler` trait implementation methods match source signatures.
- `ElementInputHandler::new(bounds, cx.entity().clone())` is correct.
- `window.set_input_handler(input_handler)` in `render()` is accurate.
- Focus management with `track_focus()` and `cx.focus()` is correct.
- UTF-8 byte offset handling with `char.len_utf8()` is correct.
- `KeyBinding::new()` registration is accurate.

---

## Chapter 68: Image Gallery (`src/part18-cookbook/gallery.md`)

### ERROR
1. **`ImageSource::uri()` does not exist** (multiple places):
   - Book uses: `img(ImageSource::uri(&url))`
   - Source: `ImageSource` has variants `Resource`, `Render`, `Image`, `Custom` but NO `uri()` constructor.
   - Correct usage: `img(url)` (via `From<&str>` impl) or `ImageSource::from(url)`.

2. **`ImageSource::resource()` does not exist** (key takeaways):
   - Book mentions: `img(ImageSource::resource(path))`
   - Source: No `resource()` constructor. Use `ImageSource::from(path)` or `img(path)`.

3. **`PinchEvent` field name wrong**:
   - Book: `event.scale` (line 259: `self.zoom_level * event.scale as f32`)
   - Source: `PinchEvent` has `delta: f32`, NOT `scale`.
   - Should be: `event.delta`.

4. **`NavigationDirection::Zoom` does not exist**:
   - Book: `MouseButton::Navigate(NavigationDirection::Zoom)`
   - Source: `NavigationDirection` only has `Back` and `Forward` variants.
   - There is no `Zoom` variant.

5. **`image_cache()` element description is misleading**:
   - Book shows: `div().image_cache().child(/* 图片内容 */)`
   - Source: `div().image_cache(cache)` requires an `ImageCacheProvider` parameter, NOT a no-arg method.
   - `image_cache()` free function also requires an `ImageCacheProvider` argument.

### OK
- `ObjectFit::Cover` / `ObjectFit::Contain` descriptions are accurate.
- `flex_wrap()` for auto-layout grid is correct.
- `ScrollWheelEvent` with `delta.pixel_delta(px(1.))` is correct.
- `cx.spawn()` + `cx.background_executor().timer()` pattern is correct.

---

## Chapter 69: Dashboard (`src/part18-cookbook/dashboard.md`)

### OK
- `actions!` macro and parameterized action (`SwitchTab { index: 0 }`) are correct.
- `cx.spawn(|view, mut cx| async move { ... view.update(&mut cx, |this, cx| ...).ok() })` pattern is correct.
- `cx.open_window()` in callback for multi-window is correct.
- Tab navigation via View state is correct.
- `cx.bind_keys()` with `KeyBinding::new()` is accurate.
- `.when(condition, |btn| btn.enabled(false))` conditional disabling is correct.

---

## Appendix A: Macros (`src/appendix/macros.md`)

### ERROR
1. **`RenderOnce` signature in examples** (lines 382-389, 436-437):
   - Book examples: `fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement`
   - Source: `fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement` (element.rs line 151)
   - **This is actually CORRECT** -- `RenderOnce` does use `cx: &mut App`, NOT `cx: &mut Context<Self>`. The previous audit report (part1) was wrong about this.

### OK
- `actions!` macro syntax and generated code are accurate.
- `#[derive(Action)]` attributes (`namespace`, `name`, `no_json`, `no_register`, `deprecated`) match source.
- Manual `Action` trait implementation with `register_action!()` is correct.
- `#[gpui::test]` macro syntax and function signatures are accurate.
- `#[gpui::property_test]` description is correct.
- `#[derive(IntoElement)]` + `RenderOnce` pattern is accurate.
- Macro decision tree is helpful and accurate.

---

## Appendix B: API Cheat Sheet (`src/appendix/api-cheat.md`)

### OK
- Entity operation table (`cx.new()`, `entity.read()`, `entity.update()`, `downgrade()`, `upgrade()`) is accurate.
- Context methods table is comprehensive and correct.
- Styled method tables are accurate and well-organized.
- Div event registration tables are correct.
- Animation and easing function descriptions match source.
- List and UniformList tables are accurate.
- `ImageSource` table lists `uri`, `resource`, `file` constructors -- but `ImageSource::uri()` and `ImageSource::resource()` do not exist as methods (see Ch68 errors above). They should be described as `From` trait conversions.

### MINOR
1. **ImageSource constructor table**:
   - Book: `ImageSource::uri(url)`, `ImageSource::resource(path)`, `ImageSource::file(path)`
   - Source: No such methods. `ImageSource` uses `From` trait implementations:
     - `From<&str>` for URI strings and embedded resource paths
     - `From<&Path>` / `From<PathBuf>` for file paths
     - `From<Arc<RenderImage>>` / `From<Arc<Image>>` for direct image data
     - `From<F>` for custom loading functions

---

## Appendix C: Zed Case Study (`src/appendix/zed-case-study.md`)

### OK
- Three-layer View architecture (Workspace -> PaneGroup -> Pane -> Item) is an accurate description of Zed's design.
- `Workspace` struct fields match Zed source code.
- `Item` trait and `Panel` trait abstractions are accurately described.
- Action system with `actions!` and namespace organization is correct.
- Key binding JSON configuration format matches Zed's actual keymap system.
- Editor multi-buffer architecture (Editor -> MultiBuffer -> Buffer) is accurately described.
- Custom Element rendering for Editor is correctly described.
- `EntityInputHandler` implementation for IME is accurate.
- Observer/subscribe pattern with `cx.subscribe()` is correct.
- `cx.spawn` + `WeakEntity` + `update` pattern is correctly described.
- `Global` state for `ThemeSettings` is accurate.

### OK
- All seven engineering practices extracted from Zed are sound and well-justified.

---

## Summary

### Critical Errors (must fix)

| # | Chapter | Issue | Book says | Source says |
|---|---------|-------|-----------|-------------|
| 1 | Ch52, Ch56, Ch64 | Linux rendering backend | Vulkan | **wgpu** |
| 2 | Ch68 | `ImageSource::uri()` and `ImageSource::resource()` | Methods exist | **`From` trait impls only** -- use `img(url)` directly |
| 3 | Ch68 | `PinchEvent` field name | `event.scale` | **`event.delta`** |
| 4 | Ch68 | `NavigationDirection::Zoom` | Exists as variant | **Only `Back` and `Forward`** exist |
| 5 | Ch68 | `image_cache()` no-arg | `div().image_cache()` | **Requires `ImageCacheProvider` parameter** |

### Minor Errors

| # | Chapter | Issue | Book says | Source says |
|---|---------|-------|-----------|-------------|
| 6 | App B | ImageSource constructors | `uri()`, `resource()`, `file()` methods | **`From<&str>`, `From<&Path>`, etc.** |

### Corrected from Previous Audit (Part 1 was wrong)

| # | Issue | Part1 report said | Actual source says |
|---|-------|-------------------|-------------------|
| 7 | `RenderOnce` cx parameter | "Should be `cx: &mut Context<Self>`" | **`cx: &mut App` is correct** -- element.rs line 151 |

This means chapters 8 and 10 from the part1 audit were flagged incorrectly for `RenderOnce` -- their use of `cx: &mut App` is actually correct. The error in chapters 8/10 was only that the book's stated signature in the *explanation text* may differ from the code examples. The code examples themselves using `cx: &mut App` are correct.

### Correctly Documented (matching source)

- Async system: `cx.spawn()`, `WeakEntity`, `Task`, `FallibleTask`, `Priority` enum
- State: `observe`, `subscribe`, `Global`, `Subscription` RAII
- Lists: `ListState::new(count, alignment, overdraw)`, `splice()`, `uniform_list`, `ScrollStrategy`
- Advanced elements: `anchored()`, `deferred()`, `with_priority()`
- Animation: `Animation::new()`, easing functions, `with_animation()`
- Text system: `Font`, `FontWeight`, `FontStyle` (with Oblique), `FontId`, `ShapedGlyph`, `LineWrapper`
- Rendering: `Scene`, `Primitive` enum order (Shadow first), `TransformationMatrix`
- Platform: `Platform` trait, `guess_compositor()`, `ThermalState`, `WindowDecorations`
- Testing: `#[gpui::test]`, `TestApp`, visual testing, `#[gpui::property_test]`
- Debugging: `Inspector`, `.debug()`, `.debug_below()`, profiler module
- Cookbook: Counter (`cx.listener`, `cx.notify`), Todo (`ListState`, `splice`), Editor (`EntityInputHandler`, `ElementInputHandler`), Dashboard (`cx.spawn` + `view.update`)
- Macros: `actions!`, `#[derive(Action)]`, `register_action!`, `#[gpui::test]`, `#[derive(IntoElement)]`
- Zed case study: All architecture descriptions, trait abstractions, and engineering practices are accurate
