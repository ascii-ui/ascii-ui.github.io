---
sidebar_position: 6
id: error-handling
title: Error Handling
sidebar_label: Error Handling
description: How ascii-ui handles render errors and how to debug them.
tags: [errors, debugging]
---

# Error Handling

ascii-ui provides informative error messages when components fail to render. Instead of crashing silently or showing cryptic stack traces, the framework catches errors and displays them in the viewport with helpful context.

---

## How It Works

When a component throws an error during render (initial or re-render), ascii-ui:

1. Catches the error using `xpcall`
2. Parses the error to extract the component path and message
3. Displays a formatted error screen in the viewport
4. Provides a contextual hint based on the error type

The viewport remains open, allowing you to fix the issue and see the error disappear on the next successful render.

---

## Error Types

ascii-ui categorizes errors into five types, each with a specific hint:

### `render`

Error during component render. This is the most common error type.

**Common causes:**
- Returning a non-table value from a component
- Accessing nil values
- Syntax errors in the render function

**Hint:**
```
Components must return a list (table) of FiberNode or BufferLine objects.
Did you forget to wrap your return value?

Example:
  return { Segment:new({ content = "hello" }):wrap() }
```

### `hook`

Error in a hook (useState, useEffect, useReducer, etc.).

**Common causes:**
- Calling hooks inside callbacks or effects
- Calling hooks conditionally
- Calling hooks inside loops

**Hint:**
```
Hooks (useState, useEffect, useReducer) must be called during component render.
They cannot be called inside callbacks, effects, or conditionally.

Rules:
  - Only call hooks at the top level of your component
  - Don't call hooks inside loops, conditions, or nested functions
```

### `effect`

Error in an effect or cleanup function.

**Common causes:**
- Accessing nil values in useEffect
- Cleanup function errors
- Calling APIs that don't exist

**Hint:**
```
Effects run after render and should not throw errors.
Check your useEffect cleanup functions and effect logic.

Common causes:
  - Accessing nil values
  - Calling APIs that don't exist
  - Cleanup function errors
```

### `interaction`

Error in user interaction handler (on_press, on_select, etc.).

**Common causes:**
- Unhandled exceptions in callbacks
- Accessing state that no longer exists

**Hint:**
```
User interaction handlers (on_press, on_select, etc.) should not throw.
Add error handling to your callbacks.

Example:
  on_press = function()
    local ok, err = pcall(function() ... end)
    if not ok then log(err) end
  end
```

### `viewport`

Error in viewport operations (open, update, close).

**Common causes:**
- Invalid window/buffer state
- Neovim API permission issues
- Floating window configuration errors

**Hint:**
```
Viewport operations (open, update, close) failed.
This may be a Neovim API issue or invalid window state.

Check:
  - Window/buffer validity
  - Neovim API permissions
  - Floating window configuration
```

---

## Error Screen Format

When an error occurs, you'll see a formatted message like this:

```
═══ RENDER ERROR ═══

Type: render
Component: MyApp > Counter > Display
Message: attempt to index local 'state' (a nil value)

Hint: Components must return a list (table) of FiberNode or BufferLine objects.
      Did you forget to wrap your return value?
      
      Example:
        return { Segment:new({ content = "hello" }):wrap() }

═══════════════════
```

The error screen shows:
- **Type** — The error category (render, hook, effect, interaction, viewport)
- **Component** — The component path where the error occurred (e.g., `App > Counter > Display`)
- **Message** — The actual error message
- **Hint** — Contextual guidance for fixing the issue

Multi-line messages and hints are split on newlines into separate display
lines, so the error screen itself never contains embedded newline
characters (single-line `Segment` content cannot hold newlines).

---

## Debugging Tips

### 1. Check the Component Path

The component path tells you where the error occurred. If you see `App > Counter > Display`, the error is in the `Display` component.

### 2. Read the Hint

Each error type includes a hint with common causes and solutions. Start by following the hint.

### 3. Use Logging

Enable debug logging to see more details:

```lua
local ui = require("ascii-ui")
ui.setup({ log_level = "debug" })
```

Check the logs with `:messages` or by viewing the log file.

### 4. Test in Isolation

Use the testing library to test components in isolation without a viewport:

```lua
local testing = require("ascii-ui.testing")

describe("MyComponent", function()
  it("renders without errors", function()
    local screen = testing.render(MyComponent)
    -- If this throws, you'll see the error in the test output
  end)
end)
```

### 5. Wrap Interactions in pcall

To prevent interaction errors from crashing your UI:

```lua
Button({
  label = "Click me",
  on_press = function()
    local ok, err = pcall(function()
      -- Your interaction logic here
    end)
    if not ok then
      vim.notify("Error: " .. err, vim.log.levels.ERROR)
    end
  end,
})
```

---

## Common Errors

### "attempt to call a nil value"

**Cause:** Trying to call a function that doesn't exist or is nil.

**Solution:** Check that all required modules are imported and that you're calling the correct function names.

```lua
-- Wrong
local ui = require("ascii-ui")
ui.createComponent("MyComponent", function()
  return { ui.components.UnknownComponent({}) }  -- nil value
end)

-- Right
local ui = require("ascii-ui")
ui.createComponent("MyComponent", function()
  return { ui.components.Paragraph({ content = "Hello" }) }
end)
```

### "attempt to index a nil value"

**Cause:** Accessing a property on a nil object.

**Solution:** Check that all variables are initialized before use.

```lua
-- Wrong
local count, setCount = useState(0)
ui.createComponent("Counter", function()
  return { Paragraph({ content = count.value }) }  -- count is a number, not a table
end)

-- Right
local count, setCount = useState(0)
ui.createComponent("Counter", function()
  return { Paragraph({ content = tostring(count) }) }
end)
```

### "hook called outside of render"

**Cause:** Calling a hook inside a callback, effect, or conditional.

**Solution:** Move hooks to the top level of your component.

```lua
-- Wrong
ui.createComponent("MyComponent", function()
  Button({
    on_press = function()
      local state = useState(0)  -- Hook inside callback!
    end,
  })
end)

-- Right
ui.createComponent("MyComponent", function()
  local state, setState = useState(0)  -- Hook at top level
  return {
    Button({
      on_press = function() setState(state + 1) end,
    }),
  }
end)
```

---

## See Also

- [ui.mount](./mount.md) — Mount lifecycle and error handling
- [Testing](./testing/testing.md) — Test components in isolation
- [Architecture](./architecture.md) — How the fiber reconciler works
