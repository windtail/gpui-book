# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **《GPUI 从入门到精通》**, a Chinese-language technical book about GPUI (the GPU-accelerated UI framework from Zed). Built with mdBook.

- **Target audience**: Rust-experienced developers
- **Source GPUI code**: `/home/ljj/work/zed/crates/gpui`
- **Book output**: `book/` directory (generated HTML)

## Key Commands

```bash
# Build the book (generates HTML to book/)
mdbook build

# Serve locally with live reload (http://localhost:3000)
mdbook serve

# Build and open in browser
mdbook build --open
```

## Book Structure

| Path | Purpose |
|------|---------|
| `src/SUMMARY.md` | Table of contents — controls chapter ordering |
| `src/outline.md` | Detailed chapter planning document (reference only, not rendered) |
| `src/part1-intro/` | Chapters 1-3: Introduction, setup, first app |
| `src/part2-fundamentals/` | Chapters 4-7: Application, Entity, Context, data flow |
| `src/part3-views/` | Chapters 8-10: Render trait, View lifecycle, composition |
| `src/part4-elements/` | Chapters 11-17: Element trait, div, text, img, SVG, canvas, custom elements |
| `src/part5-layout/` | Chapters 18-21: Flexbox, Grid, sizing, Taffy internals |
| `src/part6-styling/` | Chapters 22-25: Styled trait, colors, effects, text styling |
| `src/part7-events/` | Chapters 26-29: Mouse, keyboard, gesture, propagation |
| `src/part8-action-keymap/` | Chapters 30-33: Actions, key bindings, key context, dispatch |
| `src/part9-focus-input/` | Chapters 34-35: Focus system, text input & IME |
| `src/part10-async/` | Chapters 36-38: Foreground executor, background executor, Task |
| `src/part11-state/` | Chapters 39-42: Observe, Subscribe, Global, Subscription |
| `src/part12-advanced-elements/` | Chapters 43-47: List, UniformList, Anchored, Deferred, Animation |
| `src/part13-text-system/` | Chapters 48-50: Fonts, shaping, line layout |
| `src/part14-rendering/` | Chapters 51-53: Scene graph, GPU pipeline, performance |
| `src/part15-platform/` | Chapters 54-56: Platform trait, macOS, Linux |
| `src/part16-testing/` | Chapters 57-60: gpui::test, TestAppContext, visual tests, property tests |
| `src/part17-debugging/` | Chapters 61-63: Inspector, debug styling, profiling |
| `src/part18-cookbook/` | Chapters 64-69: Complete app examples |
| `src/appendix/` | Appendices: Macros, API cheat sheet, Zed case study |
| `book.toml` | mdBook configuration |

## Source-of-Truth Rule

**ALL API descriptions, method signatures, type definitions, and enum variants MUST be verified against the actual GPUI source code at `/home/ljj/work/zed/crates/gpui/`.**

Never fabricate APIs, methods, types, or behavior based on assumed knowledge. Every technical claim in the book must have source code backing it.

Key source locations:
- **Core types**: `gpui/src/app.rs`, `gpui/src/element.rs`, `gpui/src/view.rs`, `gpui/src/window.rs`
- **Scene**: `gpui/src/scene.rs` (Primitive enum, Scene struct)
- **Geometry**: `gpui/src/geometry.rs`
- **Styling**: `gpui/src/styled.rs`
- **Events**: `gpui/src/interactive.rs`
- **Actions**: `gpui/src/action.rs`
- **Keymap**: `gpui/src/keymap.rs`
- **Colors**: `gpui/src/colors.rs`
- **Text**: `gpui/src/text_system.rs`
- **Elements**: `gpui/src/elements/` (div, img, svg, text, list, anchored, deferred, canvas, animation)
- **Platform backends** (separate workspace crates):
  - `gpui_macos/` → Metal renderer (`metal` crate)
  - `gpui_linux/` → wgpu renderer (`gpui_wgpu` crate)
  - `gpui_windows/` → DirectX 11 renderer (`ID3D11Device`)
  - `gpui_wgpu/` → Cross-platform wgpu renderer

## Writing Conventions

- All content in **Chinese** (Simplified)
- Code examples use **Rust 2024 edition**
- Import style: `use gpui::{prelude::*, *};`
- Cookbook examples use `gpui = "0.2.x"` from crates.io as primary, git dependency as alternative
- Playground/run buttons are **disabled** (GPUI is a GUI framework, cannot run in browser)

## Known Platform Backend Facts

| Platform | Renderer | Underlying API |
|----------|----------|----------------|
| macOS | `MetalRenderer` | Metal (`metal` crate) |
| Linux | `WgpuRenderer` | wgpu (auto-selects Vulkan/OpenGL) |
| Windows | `DirectXRenderer` | DirectX 11 (`ID3D11Device`) |

## Update Process (When GPUI Source Changes)

1. Review changed GPUI source at `/home/ljj/work/zed/crates/gpui/`
2. Re-verify any book chapters that reference changed APIs/types
3. Run `mdbook build` to confirm no errors
4. Do NOT invent new content — only update existing content to match source
