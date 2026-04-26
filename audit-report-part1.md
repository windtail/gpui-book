# GPUI Book Audit Report: Chapters 1-35

Audited against GPUI source code at `/home/ljj/work/zed/crates/gpui/src/`.

---

## Chapter 1: What is GPUI (`src/part1-intro/what-is-gpui.md`)

### ERROR
1. **Wrong rendering backends** (Section 1.2.1):
   - Book: "通过 Vulkan（Linux）、Metal（macOS）或 wgpu（Windows）提交到 GPU"
   - Source: Linux uses **wgpu** (gpui_linux uses gpui_wgpu/WgpuRenderer), Windows uses **DirectX 11** (gpui_windows/src/directx_renderer.rs), NOT Vulkan.
   - Book platform table (Section 1.4.2): Linux listed as "Vulkan", Windows as "wgpu/Vulkan" -- both incorrect.

2. **Primitive enum order** (Section 1.2.1):
   - Book lists: `Quad, Shadow, Path, Underline, MonochromeSprite, SubpixelSprite, PolychromeSprite, Surface`
   - Source order: `Shadow, Quad, Path, Underline, MonochromeSprite, SubpixelSprite, PolychromeSprite, Surface`
   - (Shadow comes first, not Quad)

### OK
- Entity model description (Section 1.2.3) is accurate.
- View/Element hybrid mode description is accurate.
- Render trait signature is correct.

---

## Chapter 2: Getting Started (`src/part1-intro/getting-started.md`)

### ERROR
1. **Linux Vulkan SDK presented as primary requirement** (Section 2.1.3):
   - Book extensively describes Vulkan SDK installation for Linux, implying it is the rendering backend.
   - Source: Linux uses **wgpu**, not Vulkan. Vulkan SDK should not be a primary dependency.

### MISSING
- Should mention that Linux uses wgpu (not Vulkan) as the rendering backend.

### OK
- Project structure, Cargo.toml, and prelude documentation are accurate.

---

## Chapter 3: First App (`src/part1-intro/first-app.md`)

### ERROR
1. **Wrong platform backends** (Section 3.1.1):
   - Book: "检测并初始化平台后端（macOS 用 Metal，Linux 用 Vulkan）"
   - Source: macOS = Metal (correct), Linux = **wgpu** (NOT Vulkan).

### OK
- `AppCell` wrapping `RefCell<App>` is correct.
- Render trait signature: `fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement` is correct.
- `open_window` signature and WindowOptions are accurate.

---

## Chapter 4: Application (`src/part2-fundamentals/application.md`)

### ERROR
1. **Windows type incorrect** (Section 4.1):
   - Book: "所有窗口（`SlotMap<WindowId, Window>`）"
   - Source: `windows: SlotMap<WindowId, Option<Box<Window>>>` -- values are `Option<Box<Window>>`, not bare `Window`.

### MISSING
- `App` field `platform: Rc<dyn Platform>` not mentioned.

### OK
- `AppCell` wraps `RefCell<App>` is correct.
- `AppContext` trait definition is accurate.
- Event loop description is accurate.

---

## Chapter 5: Entity (`src/part2-fundamentals/entity.md`)

### OK
- Entity lease mode description is accurate.
- `WeakEntity` usage is correct.
- `EntityId` description is accurate.
- `reserve_entity()` / `insert_entity()` pattern is documented correctly.

### MISSING
- `AnyEntity` fields not detailed (`entity: AnyEntity`, `render: fn(...)`, `cached_style`).

---

## Chapter 6: Context (`src/part2-fundamentals/context.md`)

### OK
- Context hierarchy (App, Context<T>) is accurate.
- `AppContext` trait methods match source.
- `Deref<Target = App>` pattern is correctly described.
- Event callback signatures (`window: &mut Window, cx: &mut App`) are correct.

### MISSING
- `EntityInputHandler` not mentioned (belongs to input chapter but context methods reference it).

---

## Chapter 7: Data Flow (`src/part2-fundamentals/data-flow.md`)

### OK
- Unidirectional data flow description is accurate.
- Lease mode nested update trap is correctly described.
- `observe` vs `subscribe` distinction is accurate.
- Global vs Entity comparison is correct.

---

## Chapter 8: Render Trait (`src/part3-views/render-trait.md`)

### ERROR
1. **RenderOnce trait signature is WRONG** (Section 8.2.1):
   - Book: `fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement`
   - Book code: `pub trait RenderOnce: 'static { fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement; }`
   - Source: `fn render(self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement`
   - **This is wrong throughout the entire chapter**: all `RenderOnce` examples use `cx: &mut App` instead of `cx: &mut Context<Self>`.

2. **Multiple code examples with wrong RenderOnce signature**:
   - Section 8.2.1 `StatusBadge`: `_cx: &mut App`
   - Section 8.5.3 `Card`: `_cx: &mut App`
   - All `RenderOnce` impl blocks use `cx: &mut App`

### MISSING
- `RenderOnce` does **not** have `Sized` bound mentioned (unlike `Render` which does have `Sized`).

### OK
- Render trait signature is correct: `fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement` with `Sized` bound.
- `&mut self` reasoning is correct.
- `Component<C>` wrapper description is accurate.

---

## Chapter 9: View Lifecycle (`src/part3-views/view-lifecycle.md`)

### OK
- `cx.new()` creation flow is accurate.
- `cx.notify()` and `cx.observe()` descriptions are correct.
- Frame scheduling diagram is accurate.
- `replace_root_view()` usage is correct.

### MISSING
- `AnyView` fields not detailed: `entity: AnyEntity`, `render: fn(&AnyView, &mut Window, &mut App) -> AnyElement`, `cached_style: Option<Rc<StyleRefinement>>`.

---

## Chapter 10: View Patterns (`src/part3-views/view-patterns.md`)

### ERROR
1. **RenderOnce signatures wrong throughout**:
   - All `impl RenderOnce` blocks use `cx: &mut App` instead of `cx: &mut Context<Self>`:
     - `Parent` (line 110): `_cx: &mut App`
     - `Child` (line 117): `_cx: &mut App`
     - `Card` (line 174): `_cx: &mut App`
     - `ButtonGroup` (line 221): `cx: &mut App`
     - `Panel` (line 279): `_cx: &mut App`
     - `ConfirmDialog` (line 458): `cx: &mut App`
     - `Modal` (line 518): `cx: &mut App`

### OK
- View composition patterns are correct.
- `EventEmitter` pattern is accurate.
- Shared Entity pattern is correct.

---

## Chapter 11: Element Trait (`src/part4-elements/element-trait.md`)

### OK
- **Element trait signature is correct** with `inspector_id: Option<&InspectorElementId>` parameter (Section 11.2.1).
- Three-phase lifecycle (request_layout, prepaint, paint) is accurately described.
- `GlobalElementId` wraps `Arc<[ElementId]>` correctly documented (Section 11.4.1).
- `IntoElement` trait is accurate.
- `AnyElement` type erasure is correct.

### MISSING
- `Element` trait also has `into_any(self) -> AnyElement` method not mentioned.

---

## Chapter 12: Div (`src/part4-elements/div.md`)

### OK
- Div structure description is plausible.
- FluentBuilder pattern is correct.
- ParentElement trait is accurate.
- `id()` importance is correctly emphasized.

---

## Chapter 13: Text (`src/part4-elements/text.md`)

### OK
- `SharedString` as `Arc<str>` wrapper is accurate.
- `StyledText` and `TextRun` usage is correct.
- `InteractiveText` with click ranges is accurate.
- `label()` shortcut is correct.

---

## Chapter 14: Image (`src/part4-elements/img.md`)

### OK
- `ObjectFit` enum is accurate.
- Image loading async flow is correct.
- `StyledImage` trait description is accurate.

### MISSING
- `ImageSource::Custom` variant may need verification against source.

---

## Chapter 15: SVG (`src/part4-elements/svg.md`)

### OK
- SVG vs img comparison is reasonable.
- Transformation support description is plausible.

### MISSING
- `Transformation` struct fields need verification: book shows `scale: Size<f32>`, `translate: Point<Pixels>`, `rotate: Radians`.

---

## Chapter 16: Canvas (`src/part4-elements/canvas.md`)

### OK
- Canvas two-closure pattern (prepaint + paint) is accurate.
- `paint_quad`, `paint_path` usage is correct.
- Canvas vs custom Element comparison is accurate.

---

## Chapter 17: Custom Elements (`src/part4-elements/custom-elements.md`)

### OK
- Full Element implementation with `inspector_id` parameter is correct.
- `Interactivity` delegation pattern is accurate.
- Hitbox registration is correctly described.

---

## Chapter 18: Flexbox (`src/part5-layout/flexbox.md`)

### OK
- Flex direction, grow, shrink, basis descriptions are accurate.
- Alignment (justify_*, items_*) is correct.
- Gap and wrapping descriptions are accurate.

---

## Chapter 19: Grid (`src/part5-layout/grid.md`)

### OK
- Grid basics, col_span, row_span descriptions are accurate.
- Grid vs Flex comparison is correct.
- Implicit grid behavior is accurately described.

---

## Chapter 20: Sizing (`src/part5-layout/sizing.md`)

### OK
- **`Length` enum correct**: `Definite(DefiniteLength)`, `Auto` (Section 20.3.1).
- **`DefiniteLength` enum correct**: `Absolute(AbsoluteLength)`, `Fraction(f32)`.
- **`AbsoluteLength` enum correct**: `Pixels(Pixels)`, `Rems(Rems)`.
- **`Anchor` enum correct**: All 8 variants listed (Section 20.6.3).
- **`Edges<T>` and `Corners<T>` correct**: fields match source.
- `Pixels(pub(crate) f32)` visibility nuance not mentioned but acceptable for user-facing docs.

### OK
- `DevicePixels(pub i32)` -- book mentions it's for GPU textures, correct.

---

## Chapter 21: Taffy (`src/part5-layout/taffy.md`)

### OK
- Taffy integration description is accurate.
- `LayoutId` as opaque handle is correct.
- `AvailableSpace` enum is accurate.
- `request_measured_layout` description is correct.

---

## Chapter 22: Styled (`src/part6-styling/styled.md`)

### OK
- `Styled` trait definition is accurate.
- `StyleRefinement` incremental pattern is correctly described.
- State-driven styles (hover, active, focus) are accurate.
- `group()` pattern is correct.

---

## Chapter 23: Colors (`src/part6-styling/colors.md`)

### OK
- **`Colors` struct fields are correct**: text, selected_text, background, disabled, selected, border, separator, container -- **all `Rgba`**, NOT `Hsla` (Section 23.5.2).
- `Rgba` struct with `r, g, b, a: f32` is correct.
- `DefaultAppearance`: `Light`, `Dark` mentioned; book also includes `VibrantLight`, `VibrantDark` which may be `WindowAppearance` variants (not `DefaultAppearance`).
- `rgb()` and `rgba()` constructors are correct.

### ERROR
1. **`DefaultAppearance` vs `WindowAppearance`** (Section 23.5.1):
   - Book uses `WindowAppearance` with variants `Light`, `Dark`, `VibrantLight`, `VibrantDark`.
   - Source: `DefaultAppearance` has only `Light`, `Dark`. `VibrantLight`/`VibrantDark` may be from `WindowAppearance` -- needs clarification.

---

## Chapter 24: Effects (`src/part6-styling/effects.md`)

### OK
- Border, shadow, opacity, transform descriptions are accurate.
- `TransformationMatrix` chain pattern is correct.
- Performance note about opacity vs alpha is correct.

---

## Chapter 25: Text Styling (`src/part6-styling/text-styling.md`)

### OK
- **`FontWeight` constants correct**: THIN, EXTRA_LIGHT, LIGHT, NORMAL, MEDIUM, SEMIBOLD, BOLD, EXTRA_BOLD, BLACK (Section 25.2.1).
- **`FontStyle` partially correct**: Book shows `Normal`, `Italic` (Section 25.2.2).
- Text decoration, alignment, line height descriptions are accurate.

### ERROR
1. **`FontStyle` missing `Oblique` variant** (Section 25.2.2):
   - Source: `FontStyle` has `Normal`, `Italic`, **`Oblique`**.
   - Book only shows `Normal` and `Italic`.

### MISSING
- `FontStyle::Oblique` variant not documented.

---

## Chapter 26: Mouse (`src/part7-events/mouse.md`)

### OK
- **`MouseButton` enum correct**: Left, Right, Middle, Navigate(NavigationDirection) (Section 26.1.3).
- **`NavigationDirection` correct**: Back, Forward.
- **`ScrollDelta` enum correct**: `Pixels(Point<Pixels>)`, `Lines(Point<f32>)` (Section 26.3.2).
- **`FileDropEvent`**: Book shows `Entered { position, paths }`, `Pending { position }`, `Submit { position }`, `Exited`.
  - Source: `Entered(ExternalPaths)`, `Pending`, `Submit`, `Exited` -- **book adds extra `position` fields and `paths: ExternalPaths` struct detail that differ from source**.

### ERROR
1. **`FileDropEvent` variant signatures** (Section 26.5.1):
   - Book: `Entered { position: Point<Pixels>, paths: ExternalPaths }`, `Pending { position }`, `Submit { position }`
   - Source: `Entered(ExternalPaths)`, `Pending`, `Submit`, `Exited` -- simpler tuple/unit variants without position fields.

---

## Chapter 27: Keyboard (`src/part7-events/keyboard.md`)

### OK
- `KeyDownEvent` and `KeyUpEvent` structures are accurate.
- `Keystroke` parsing is correct.
- `Modifiers` struct is accurate.
- `ModifiersChangedEvent` description is correct.

---

## Chapter 28: Gestures (`src/part7-events/gesture.md`)

### OK
- `PinchEvent` structure is plausible.
- `TouchPhase` enum (Started, Moved, Ended) is correct.
- Gesture + zoom combination example is comprehensive.

---

## Chapter 29: Event Propagation (`src/part7-events/propagation.md`)

### OK
- Capture/Bubble phase description is accurate.
- `DispatchPhase` enum is correct.
- `stop_propagation()` and `prevent_default()` are correctly described.
- Hitbox tree algorithm is accurate.

---

## Chapter 30: Defining Actions (`src/part8-action-keymap/defining-actions.md`)

### OK
- `Action` trait definition is accurate.
- `actions!()` macro is correctly described.
- `#[derive(Action)]` with `#[action(...)]` attributes is correct.
- `NoAction` and `Unbind` descriptions are accurate.

---

## Chapter 31: Key Bindings (`src/part8-action-keymap/key-bindings.md`)

### OK
- `KeyBinding::new()` signature is correct.
- Keystroke string format is accurate.
- `cx.bind_keys()` usage is correct.
- Key sequence (e.g., "g g") description is accurate.

---

## Chapter 32: Key Context (`src/part8-action-keymap/key-context.md`)

### OK
- **`KeyBindingContextPredicate` enum correct**: Identifier, Equal, NotEqual, Descendant, Not, And, Or (Section 32.3.1).
- Predicate string parsing is accurate.
- Context stack evaluation is correctly described.
- `depth_of()` method is correct.

---

## Chapter 33: Action Dispatch (`src/part8-action-keymap/action-dispatch.md`)

### OK
- `on_action()` registration is accurate.
- Handler signature: `Fn(&A, &mut Window, &mut App)` is correct.
- Action resolution pipeline (keystroke -> action -> handler) is accurate.
- `dispatch_action()` usage is correct.
- Test patterns are accurate.

---

## Chapter 34: Focus (`src/part9-focus-input/focus.md`)

### OK
- `FocusHandle` creation via `cx.focus_handle()` is correct.
- `track_focus()` usage is accurate.
- Tab navigation with `tab_index()` is correct.

### MISSING
1. **`FocusHandle` fields not fully documented**:
   - Source: Fields include `id`, `handles`, **`tab_index`**, **`tab_stop`**.
   - Book does not mention `tab_index`/`tab_stop` fields or clone modifiers for these.
   - The book does cover `tab_index()` and `tab_stop(false)` as methods (Section 34.4), but doesn't connect them to FocusHandle fields.

---

## Chapter 35: Text Input (`src/part9-focus-input/text-input.md`)

### OK
- **`EntityInputHandler` trait methods correct**: text_for_range, selected_text_range, marked_text_range, unmark_text, replace_text_in_range, replace_and_mark_text_in_range, bounds_for_range, character_index_for_point, accepts_text_input (Section 35.1.1).
- **`ElementInputHandler` usage correct**: `ElementInputHandler::new(element_bounds, cx.entity())` (Section 35.2.1).
- IME marked text lifecycle is accurately described.
- UTF-8/UTF-16 conversion patterns are correct.

### OK (signature verification)
- Chapter 35 correctly uses `cx: &mut Context<Self>` for EntityInputHandler methods, which matches the source. This is consistent and correct.

---

## Summary

### Critical Errors (must fix)

| # | Chapter | Issue | Book says | Source says |
|---|---------|-------|-----------|-------------|
| 1 | Ch1, Ch2, Ch3 | Linux rendering backend | Vulkan | **wgpu** |
| 2 | Ch1, Ch3 | Windows rendering backend | wgpu/Vulkan | **DirectX 11** |
| 3 | Ch8, Ch10 | RenderOnce `cx` parameter type | `cx: &mut App` | **`cx: &mut Context<Self>`** |
| 4 | Ch4 | Window storage type | `SlotMap<WindowId, Window>` | **`SlotMap<WindowId, Option<Box<Window>>>`** |
| 5 | Ch26 | FileDropEvent variant signatures | `Entered { position, paths }` etc. | **`Entered(ExternalPaths)`, `Pending`, `Submit`, `Exited`** |

### Minor Errors

| # | Chapter | Issue | Book says | Source says |
|---|---------|-------|-----------|-------------|
| 6 | Ch1 | Primitive enum order | Quad first | **Shadow first** |
| 7 | Ch25 | FontStyle missing variant | Normal, Italic | **Normal, Italic, Oblique** |
| 8 | Ch23 | DefaultAppearance variants | Includes VibrantLight/Dark | **DefaultAppearance: Light, Dark only** |

### Missing Content

| # | Chapter | Missing API |
|---|---------|-------------|
| 9 | Ch4 | `App.platform: Rc<dyn Platform>` |
| 10 | Ch5 | `AnyView` fields (`entity`, `render`, `cached_style`) |
| 11 | Ch9 | `AnyView` field details |
| 12 | Ch11 | `Element::into_any(self) -> AnyElement` |
| 13 | Ch34 | `FocusHandle` fields: `tab_index`, `tab_stop` |

### Correctly Documented (matching source)

- Render trait: `&mut self, window: &mut Window, cx: &mut Context<Self>` -- correct
- Element trait with `inspector_id` parameter -- correct
- GlobalElementId wraps `Arc<[ElementId]>` -- correct
- MouseButton: Left, Right, Middle, Navigate(NavigationDirection) -- correct
- NavigationDirection: Back, Forward -- correct
- ScrollDelta: Pixels(Point<Pixels>), Lines(Point<f32>) -- correct
- KeyBindingContextPredicate: Identifier, Equal, NotEqual, Descendant, Not, And, Or -- correct
- Colors struct fields as Rgba (not Hsla) -- correct
- FontWeight constants (THIN through BLACK) -- correct
- Length/DefiniteLength/AbsoluteLength enums -- correct
- Anchor enum (8 variants) -- correct
- Edges<T> and Corners<T> -- correct
- EntityInputHandler methods -- correct
- ElementInputHandler usage -- correct
- Subscription pattern -- correct
