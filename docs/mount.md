---
id: mount
title: ui.mount
sidebar_label: ui.mount
sidebar_position: 5
description: Mount a root component into a viewport and start the render loop.
tags: [api]
---

# ui.mount

`ui.mount` starts the render loop for a root component. It performs the initial render, opens the viewport, binds keyboard interactions, and starts listening for state changes.

```lua
ui.mount(RootComponent)
-- or with an explicit viewport:
ui.mount(RootComponent, viewport)
```

---

## Reference

### `ui.mount(RootComponent, viewport?)`

#### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `RootComponent` | `FunctionalComponent` | Yes | The root component to render. This is a component created with `ui.createComponent`. |
| `viewport` | `ascii-ui.Viewport` | No | Where to render the UI. Defaults to a centered Neovim floating window. |

#### Returns

| Name | Type | Description |
|------|------|-------------|
| `bufnr` | `integer` | The Neovim buffer number used by the viewport. Returns `-1` for non-Neovim viewports (e.g. `StdoutViewport`). |

---

## Usage

### Default: Neovim floating window

The simplest call opens a centered, rounded floating window automatically sized to fit the rendered content:

```lua
local ui = require("ascii-ui")

local MyApp = ui.createComponent("MyApp", function()
  return {
    ui.components.Paragraph({ content = "Hello from a floating window!" }),
  }
end)

ui.mount(MyApp)
```

### StdoutViewport (headless / CLI)

Pass a `StdoutViewport` to render to the terminal instead of a Neovim window. Useful for headless scripts, CI pipelines, or animations:

```lua
local ui = require("ascii-ui")

local MyApp = ui.createComponent("MyApp", function()
  return {
    ui.components.Paragraph({ content = "Rendered to stdout!" }),
  }
end)

ui.mount(MyApp, ui.viewports.StdoutViewport.new())
```

---

## Lifecycle

`ui.mount` manages the full lifecycle of the UI:

1. **Initial render** — Calls `fiber.render(RootComponent)` to build the fiber tree and produce the first buffer.
2. **Open viewport** — Calls `viewport:open()`. For the default floating window this creates the Neovim buffer and window.
3. **Update viewport** — Calls `viewport:update(buffer)` to write the rendered content.
4. **Keyboard bindings** — Registers `vim.on_key` for `h`/`j`/`k`/`l` navigation between focusable segments. Horizontal keys (`h`/`l`) move to the previous/next focusable in linear order, while vertical keys (`j`/`k`) move spatially to the closest focusable above or below by column distance.
5. **State change listener** — On every `useState` setter call, rerenders and refreshes the viewport.
6. **Cleanup** — On `WinClosed`, detaches all bindings, calls `root:unmount()` to clean up effects and timers, calls `viewport:close()`, and runs `useEffect` cleanups.

---

## Error Handling

When a component throws an error during render, ascii-ui catches it and displays a formatted error message in the viewport instead of crashing silently.

### Render Errors

If your component throws during the initial render or a re-render, you'll see an error screen:

```
═══ RENDER ERROR ═══

Type: render
Component: MyApp > Counter
Message: attempt to index local 'state' (a nil value)

Hint: Components must return a list (table) of FiberNode or BufferLine objects.
      Did you forget to wrap your return value?
      
      Example:
        return { Segment:new({ content = "hello" }):wrap() }

═══════════════════
```

### Error Types

ascii-ui categorizes errors by type:

| Type | Description |
|------|-------------|
| `render` | Error during component render |
| `hook` | Error in a hook (useState, useEffect, etc.) |
| `effect` | Error in an effect or cleanup function |
| `interaction` | Error in user interaction handler |
| `viewport` | Error in viewport operations |

Each error type includes a contextual hint to help you debug the issue.

### Error Recovery

After a render error, the viewport remains open showing the error message. Fix the issue in your component code, and the error will disappear on the next successful render.

See [Error Handling](./error-handling.md) for more details on debugging and common error patterns.

---

## Default keyboard bindings

These bindings are active while the ascii-ui window is focused:

| Key | Action |
|-----|--------|
| `h` | Move focus left / to the previous focusable element (linear order) |
| `l` | Move focus right / to the next focusable element (linear order) |
| `k` | Move focus up (2D: closest focusable above by column distance) |
| `j` | Move focus down (2D: closest focusable below by column distance) |
| `<CR>` | Trigger the `SELECT` interaction on the focused element |
| `q` | Close the window (configurable via `ui.setup`) |

### Vertical (2D) navigation

`j` and `k` navigate spatially, not linearly. From the current cursor
position, ascii-ui searches line by line above (`k`) or below (`j`) and
lands on the focusable with the smallest column distance on the nearest
line that has one. If no focusable exists in that direction, focus stays
put.

```
Line 1: [A]  [B]   → A at col 0, B at col 3
Line 2: [C]  [D]   → C at col 0, D at col 3
```

- With focus on `D` (line 2, col 3), pressing `k` moves to `B`
  (line 1, col 3) — not to `C`, which would be previous in linear order.
- With focus on `B` (line 1, col 3), pressing `j` moves to `D`
  (line 2, col 3).
- When columns don't align exactly, the closest column wins. For example,
  with `A` at col 0 and `B` at col 11 on line 1 and `C` at col 6 on
  line 2, pressing `k` from `C` moves to `B` (distance 5 beats distance 6).

---

## Custom viewport

Any object satisfying the `ascii-ui.Viewport` interface can be passed as the second argument. The interface requires these methods:

| Method | Description |
|--------|-------------|
| `open()` | Initialize/open the output target |
| `close()` | Tear down |
| `update(buffer)` | Render the buffer frame |
| `is_focused()` | Returns `true` if the viewport has user focus |
| `enable_edits()` | Allow text input (for `Input` widgets) |
| `disable_edits()` | Revert to read-only |
| `get_id()` | Neovim window id, or `-1` |
| `get_bufnr()` | Neovim buffer number, or `-1` |
| `get_ns_id()` | Neovim namespace id, or `-1` |

See the [StdoutViewport](./viewports/stdout.md) documentation for a built-in example.
