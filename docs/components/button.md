---
id: button
title: Button
sidebar_label: Button
sidebar_position: 1
description: A focusable interactive button component for ascii-ui.
tags: [components, interactive]
---

# Button

`Button` renders a single focusable, interactive button. Pressing `<CR>` while the button is focused calls the `on_press` callback.

```lua
ui.components.Button({ label = "Click me", on_press = function() ... end })
```

---

## Reference

### Props

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `label` | `string` | Yes | The text displayed inside the button. |
| `on_press` | `fun()` | No | Callback invoked when the user presses `<CR>` on the button. |

---

## Usage

### Basic button

```lua
local ui = require("ascii-ui")
local Button = ui.components.Button

local MyApp = ui.createComponent("MyApp", function()
  return {
    Button({
      label = "Press me",
      on_press = function()
        print("Button pressed!")
      end,
    }),
  }
end)

ui.mount(MyApp)
```

### Button with state

Combine `Button` with `useState` to update the UI when the button is pressed:

```lua
local ui = require("ascii-ui")
local useState = ui.hooks.useState
local Button = ui.components.Button
local Paragraph = ui.components.Paragraph

local Counter = ui.createComponent("Counter", function()
  local count, setCount = useState(0)

  return {
    Paragraph({ content = "Count: " .. count }),
    Button({
      label = "Increment",
      on_press = function()
        setCount(count + 1)
      end,
    }),
  }
end)

ui.mount(Counter)
```

---

## Styling

The `Button` component uses the `AsciiUIButton` highlight group, which applies a gold background with dark text by default. You can override this group in your Neovim colorscheme configuration.

---

## Keyboard interaction

| Key | Action |
|-----|--------|
| `h` / `l` | Move focus to the previous / next focusable element (linear order) |
| `j` / `k` | Move focus to the closest focusable element below / above (2D, by column distance) |
| `<CR>` | Trigger `on_press` |

See [ui.mount](../mount.md#default-keyboard-bindings) for how vertical
(2D) navigation picks the nearest focusable by column distance.
