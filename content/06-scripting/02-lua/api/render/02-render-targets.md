---
title: "Render targets"
---

Globals for the `.Lrt` [render target assets](../../../../03-rendering/01-materials.md#render-targets-live-feeds-on-materials-and-the-hud) a Scene Capture draws into. Paths are asset relative. The entity side
(`set_capture_target`, `set_ui_texture`, `set_material_texture`) is in [HUD](../entity/10-hud.md).

## `render_target_info(path)`

```lua
local info = render_target_info("rendertargets/lobby.Lrt")
if info.exists then print(info.width, info.height, info.format, info.filter, info.live, info.renders) end
```

A table: `exists` (false when the file is missing or not a render target, and then nothing else is set),
`width`, `height`, `format` (`"LDR"` or `"HDR"`), `filter` (`"Linear"` or `"Nearest"`), `live` (a capture
currently draws into it, so its image exists) and `renders` (how many times it was drawn since the image
was made).

## `render_target_resize(path, w, h)` / `render_target_set(path, fields)`

```lua
render_target_resize("rendertargets/lobby.Lrt", 640, 360)
render_target_set("rendertargets/lobby.Lrt", { format = "HDR", filter = "nearest", clear = { 0, 0, 0.1, 1 } })
```

Changes the running description without touching the file; the image is replaced at the top of the next
frame, so a sprite or monitor shows the old size for one more frame and the new picture after the next
render. `fields` takes any of `width`, `height`, `format`, `filter` and `clear` (an `{r, g, b, a}` list).
Sizes are clamped to 16..2048. Both return false for a missing target.

## `render_target_save(path)` / `render_target_reload(path)` / `render_target_create(path, w, h [, format])`

`save` writes the running description to the file. `reload` drops runtime changes and re-reads the file
within half a second. `create` writes a new `.Lrt` (`format` is `"LDR"` by default) and returns whether it
was written; a file that was missing when first asked about is picked up within half a second of being written.
