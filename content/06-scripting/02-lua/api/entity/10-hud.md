---
title: "HUD"
---

An element's text, its layout and its interaction, by entity. See [Lua Scripting](../../index.md) for how a script reaches an entity and when it runs.

A HUD element is an entity carrying a `UITransformComponent` plus a `TextComponent` or a `SpriteComponent`, or both. These are entity methods: a script that holds the element calls them on it, the same way it calls `get_position`.

**Every call here is a silent no-op on an entity missing the component it needs**, and every reader answers a neutral value rather than raising. A HUD script does not necessarily know what it is attached to, and asking the wrong entity should not be a failure. `set_text` is the one exception worth knowing: the first time it meets an entity with no `TextComponent` it writes a warning to `LAB.log`, then stays quiet.

`get_ui_rect`, `ui_hovered`, `ui_pressed` and `ui_clicked` reach further than one component: they read the HUD's resolved layout and its hit test, which is also what the `ui` table's by-name calls read.

[C++ equivalent](../../../03-cpp/api/ui/01-hud-elements.md), and [HUD interaction](../../../03-cpp/api/ui/02-hud-interaction.md) for the hit-testing side.

## Units

Positions and sizes are **UI units**: the pixels a HUD is authored in, scaled to the real viewport by the project's reference height when one is set (`ProjectConfig::UIReferenceHeight`). A layout authored for a 1080p reference height therefore looks the same on a Steam Deck, with no per-resolution maths in the script.

An element is placed by an anchor rectangle and offsets from it:

| Field | Meaning |
|---|---|
| `Anchor` (min) | A fraction of the **viewport**, or of the parent's rect for a Relative element. x: 0 left, 1 right. y: 0 top, 1 bottom |
| `Anchor` max | The far corner of the anchor rectangle. Where it equals the min on an axis, that axis is a point; where it differs, the element **stretches** with its parent on that axis |
| `Pivot` | A fraction of the element's **own size**: the point of the element that sits at the anchor. Point axes only |
| `Offset` | UI units from the anchor point, applied before the pivot correction; on a stretched axis, from the min anchor to the left/top edge |
| `OffsetMax` | Stretched axes only: UI units from the max anchor to the right/bottom edge, negative to pull it inward |

**+y is down.** Anchor `(0, 0)` is the top-left corner and `(0.5, 1)` is bottom centre, which is where a subtitle belongs. A positive offset y moves the element further down. The y axis runs opposite to the world's because it is screen space.

`Offset` and `Size` are the two a script normally animates: a bar that fills, a marker that slides out. Anchor and pivot are authored in the Properties panel and left alone.

An axis stretches when its two anchors differ: the element then spans from the min anchor plus `Offset` to the max anchor plus `OffsetMax`, and `Size` and `Pivot` do not place it on that axis. See [Anchors](../../../../05-ui/01-hud.md#anchors-points-and-stretching) for the formula and the editor's presets.

## `set_text(text)`

```lua
entity:set_text("1240")
```

The `TextComponent`'s text. UTF-8, and a newline in the string breaks a line.

Silent no-op on an entity with no `TextComponent`, with the one-time warning above.

## `get_text()`

```lua
local current = entity:get_text()
```

The element's own text, or `''` when there is no `TextComponent`. It always answers a string, so a comparison needs no nil check.

A script that formats its own text every frame rarely needs this, but it is how an authored string is read back, and how a script confirms what it just set.

## `set_text_rich(enabled)` / `get_text_rich()`

```lua
log:set_text_rich(true)
log:set_text("<c=#ff4040>ERROR</c> disk <c=#40ff80>ok</c>")
```

Turns on colour spans in the text: `<c=#rrggbb>..</c>` and `<c=#rrggbbaa>..</c>`, nestable, with `<<` for a literal `<`. A tag that is not one of these stays in the text as written. `measure_text` leaves the tags out, so a panel sized from it fits what is drawn. See [Rich text](../../../../05-ui/01-hud.md#rich-text).

`get_text_rich` answers `false` on an entity with no `TextComponent`. `set_text_rich` is a no-op there, with the one-time warning above.

## `get_font_size()` / `set_font_size(size)`

```lua
local size = entity:get_font_size()
entity:set_font_size(size * 1.5)
```

The text's font size in UI units, `24` as authored. `set_font_size` clamps anything below 1 up to 1, and `get_font_size` answers `0` for an entity with no `TextComponent`.

Animating the font size grows the glyphs in place, while animating the element's size moves the layout instead. A pop or a scale effect wants the first.

## `set_text_outline(width [, rgb [, alpha]])`

```lua
entity:set_text_outline(2)
entity:set_text_outline(2, vec3.new(0, 0, 0), 0.8)
```

An outline around the glyphs, for text that has to stay readable over a moving background.

The **width always applies**, clamped at 0. The colour and the alpha only apply when that argument is given, so changing the width alone never recolours the outline by accident. That is the same has-argument rule `set_ui_color` and `set_ui_border` follow.

A text component is authored with no outline and a black outline colour underneath it, so a call that gives a width and nothing else produces black edges.

No-op with no `TextComponent`.

## `measure_text([override])`

```lua
local width, height = entity:measure_text()
local overWidth = entity:measure_text(message)
```

The laid-out size of the text in UI units: **width, height**. Wrapped at the element's own width when the component's `Wrap` is on. Passing a string measures that string instead of the component's own text, which is how a script sizes a panel around text it is about to set.

Needs a `TextComponent` and a live renderer's default font, so it answers `0, 0` on an entity without one, and before there is anything to lay text out with.

```lua
local width, height = panel:measure_text(message)
panel:set_ui_size(width + 24, height + 16)
panel:set_text(message)
```

## `set_ui_visible(visible)` / `is_ui_visible()`

A hidden element is not drawn and **does not receive interaction**: hover, press and click all report nothing for it. So showing and hiding is the whole mechanism for a menu.

`is_ui_visible` answers `false` for an entity with no `UITransformComponent`, which is the same answer a hidden element gives. An element is visible until something hides it.

## `set_ui_anchor(x, y)` / `get_ui_anchor()`

Where the element is pinned in the viewport, as fractions.

```lua
local ax, ay = entity:get_ui_anchor()
entity:set_ui_anchor(0.5, 1)
```

`get_ui_anchor` answers `0, 0` with no `UITransformComponent`, which is also the component's authored default. The anchor above is bottom centre.

`set_ui_anchor` always makes a **point** on both axes (min and max both `(x, y)`), and clears `OffsetMax`, which a point has no use for; an element that was stretched keeps the size it had. `get_ui_anchor` answers the min corner.

## `set_ui_anchors(minx, miny, maxx, maxy)` / `get_ui_anchors()`

The anchor rectangle, as fractions. An axis where min and max differ stretches.

```lua
-- Span the parent's full width, pinned to its top edge.
entity:set_ui_anchors(0, 0, 1, 0)
entity:set_ui_offsets(10, 8, -30, 0)
local minx, miny, maxx, maxy = entity:get_ui_anchors()
```

A raw write, like `set_ui_anchor`: the offsets are not adjusted to keep the element where it was, so set them afterwards. `get_ui_anchors` answers `0, 0, 0, 0` with no `UITransformComponent`.

## `set_ui_offsets(minx, miny, maxx, maxy)` / `get_ui_offsets()`

`Offset` (the first two) and `OffsetMax` (the last two) together, in UI units. On a stretched axis they are the distances from the min and max anchors to the element's edges; the max ones are **negative to inset**, so `-30` on x is a 30-unit margin on the right. On a point axis only `Offset` counts. `get_ui_offsets` answers zeros with no `UITransformComponent`.

## `set_ui_pivot(x, y)`

The point about which the element is positioned, as fractions of its own size. `(0.5, 0.5)` is its centre, `(0, 0)` its top-left.

An element that should grow from its centre needs a centre pivot, and one that should grow rightwards needs a left one.

There is no getter. The pivot is an authored value, and the Properties panel is where to read it.

## `set_ui_offset(x, y)` / `get_ui_offset()`

The offset from the anchor, in UI units, and +y down. This is the call for animation: nudging a label, sliding a panel in, shaking a health bar.

```lua
local x, y = entity:get_ui_offset()
entity:set_ui_offset(x, y - 40 * dt)
```

`get_ui_offset` answers `0, 0` with no `UITransformComponent`. Since +y is down, the line above slides the element up.

On a stretched axis `Offset` is the left/top inset, so `set_ui_offset` moves that edge and resizes the element rather than sliding it. Use `set_ui_offsets` to move both edges.

## `set_ui_size(width, height)` / `get_ui_size()`

The element's size in UI units, `100` by `100` as authored. Negative values are clamped to 0, so a bar mid-animation cannot invert. `get_ui_size` answers `0, 0` with no `UITransformComponent`.

On an axis that is **stretched** (see `set_ui_anchors`) the parent decides the size. `set_ui_size` leaves that axis alone, changes the other one, and writes a warning to `LAB.log` the first time it does so for an element; `get_ui_size` reports what the axis resolves to in UI units, as of the last frame's viewport. To size a stretched axis, change its offsets, or make it a point with `set_ui_anchor` first.

## `set_ui_layer(layer)` / `get_ui_layer()`

Draw order, and the tie-break for which element a pointer hit lands on where two overlap. Higher is on top. `get_ui_layer` answers `0` with no `UITransformComponent`.

## `set_ui_color(rgb [, alpha])`

```lua
entity:set_ui_color(vec3.new(1, 0.2, 0.2))
entity:set_ui_color(vec3.new(1, 0.2, 0.2), 0.5)
```

Colours the element's text **and** its sprite, whichever of the two it has, both if both.

The alpha only replaces the component's own when it is given, so a script can recolour without knowing the current opacity. That has-argument rule runs through every colour call in this area, and the reason is the same each time: a caller changing one of two things should not have to read the other one first. It is also why there is no getter for the colour here.

No-op on an entity with neither component.

## `set_ui_alpha(alpha)`

```lua
entity:set_ui_alpha(1 - t)
```

The alpha of the sprite and the text, and of their border, outline and shadow: the whole-element fade. Clamped to 0 to 1.

Unlike `set_ui_color`, there is no has-argument here: an alpha is always an alpha, and it writes both components' own alpha directly. So one owner per element is the shape to use rather than a fade written from two call sites. No-op with neither component.

## `set_ui_radius(radius)`

The corner radius of a sprite element, in UI units, clamped at 0. `0` is a plain rectangle.

Sprites are drawn as a signed-distance rounded rectangle, so a panel, a pill or a circle (radius = half the size) needs no texture and stays crisp at any size. No-op with no `SpriteComponent`.

## `set_ui_line(from, to [, thickness])` / `set_ui_line_color(r, g, b [, a])`

```lua
line:set_ui_line(nodeA, nodeB, 3)                    -- the centres of two elements
line:set_ui_line({ 100, 100 }, { x = 400, y = 250 })  -- two points
line:set_ui_line_color(0.2, 1, 0.4, 0.8)
```

Sets the ends of a `UILineComponent` (add it with `entity:add_component("UI Line")`). Each end is an **entity**, in which case the line follows the centre of its rect, or a **point** `{x, y}` (or `{x = .., y = ..}`) in UI units from the top-left corner of this entity's own rect. The two can be mixed. The thickness only changes when it is given.

`set_ui_line_color` takes the colour channels `0..1` and an alpha that defaults to 1. `set_ui_color` and `set_ui_alpha` reach the line too. All of it is a silent no-op on an entity without a `UILineComponent`. See [Lines](../../../../05-ui/01-hud.md#lines).

## `set_ui_capture(camera)` / `set_ui_cctv(enabled [, noise [, scanlines [, pixelated]]])`

```lua
monitor:set_ui_capture(scene.find("LobbyCam"))   -- show the camera's live picture
monitor:set_ui_cctv(true, 0.2, 0.4)              -- security-camera look, grain 0.2, scanlines 0.4
monitor:set_ui_capture(nil)                      -- back to the texture
```

`set_ui_capture` makes the sprite show what a [Scene Capture](../../../../02-building-worlds/02-components.md#scene-capture) camera sees, live; `nil` (or an entity that is not valid) clears it. `camera` is the entity that carries the Scene Capture component. `set_ui_cctv` turns the scanlines, grain and vignette on or off; `noise` and `scanlines` are `0..1`, and `pixelated` picks nearest filtering; each only changes when given. Both are silent no-ops on an entity without a `SpriteComponent`. See [Live camera feeds](../../../../05-ui/01-hud.md#live-camera-feeds-and-the-cctv-look).

## `set_capture_active(active)` / `capture_now()` / `set_capture_size(w, h)` / `capture_render_count()`

```lua
cam:set_capture_active(false)    -- freeze its feed on the last frame
cam:capture_now()                -- render it once at the next opportunity, even when inactive
cam:set_capture_size(640, 360)   -- a sharper picture
local n = cam:capture_render_count()
```

On the entity with the Scene Capture component. `set_capture_active` is the component's Active switch: off, it stops rendering and its sprites keep the last picture. `capture_now` takes the next frame's capture slot (at most one capture renders a frame, and this one goes first). `set_capture_size` changes the image size (16 to 2048 a side); the image is replaced at the start of the next frame, so the feed shows black for that one frame and the next render is at the new size. `capture_render_count` is how many times it has rendered since its image was made, for tests and tools. No-ops (0 for the count) on an entity without a `SceneCaptureComponent`.

## `set_capture_target(path)` / `get_capture_target()`

```lua
cam:set_capture_target("rendertargets/lobby.Lrt")   -- draw into the shared render target asset
cam:set_capture_target(nil)                          -- back to the capture's private image
```

On the entity with the Scene Capture component. The picture then goes to the [render target](../../../../03-rendering/01-materials.md#render-targets-live-feeds-on-materials-and-the-hud) and the capture takes its size and format from the asset. `capture_render_count` counts that target's renders. `set_ui_texture` takes a `.Lrt` path too, and `set_material_texture(slot, path)` (slot is `"albedo"`, `"emissive"`, `"normal"` or `"metallic_roughness"`, on the material entity) does the same for a 3D material.

## `set_ui_border(width [, rgb [, alpha]])`

```lua
entity:set_ui_border(4)
entity:set_ui_border(4, vec3.new(1, 1, 1), 0.9)
```

The border of a sprite element: a width in UI units, plus a colour and an alpha only when they are given. Same rule as `set_ui_color`: the width always applies, the other two only where theirs do.

A border is authored with width 0 and a black colour, so giving a width and nothing else shows black. No-op with no `SpriteComponent`.

## `set_ui_texture(path)`

```lua
entity:set_ui_texture("textures/icons/health.png")
```

Points a sprite element at a texture, project-relative, the same string form a `.Lscene` holds.

A new path is loaded through the texture cache on first draw, so switching between a small set of icons is fine and a generated path per frame is a disk and upload cost. A path that fails to load draws the plain white fallback rather than skipping the draw, so a typo shows as a white rectangle rather than a hole.

No-op with no `SpriteComponent`.

## `set_ui_slice(l, t, r, b [, scale])` / `get_ui_slice()`

```lua
entity:set_ui_slice(16, 16, 16, 16)        -- 9-slice with 16-texel borders
entity:set_ui_slice(16, 16, 16, 16, 2)     -- and each border texel drawn 2 UI units wide
local l, t, r, b, scale = entity:get_ui_slice()
```

The 9-slice borders of a sprite element: how far in from the left, top, right and bottom edges of its **texture** the corners end, in **texels of the image** (not UI units). The four corners are then drawn at their natural size and the edges and centre fill the rest, so a frame stays crisp at any size. Any border above 0 turns slicing on; all four at 0 turns it off. Negative values are 0. See [9-slice sprites](../../../../05-ui/01-hud.md#9-slice-sprites).

`scale` is the on-screen size of one border texel in UI units (default 1, clamped to at least 0.01) and only changes when it is given, the same has-argument rule as `set_ui_border`; `set_ui_slice(16, 16, 16, 16)` leaves an earlier scale alone. `get_ui_slice` answers `l, t, r, b, scale`, and `0, 0, 0, 0, 1` for an entity with no `SpriteComponent`.

A sliced sprite ignores `set_ui_radius` and `set_ui_border` (and the Softness field) and needs a texture; with none it draws as a plain quad. No-op with no `SpriteComponent`.

## `set_ui_slice_fill(fill [, draw_center])`

```lua
entity:set_ui_slice_fill("tile")          -- repeat the edges and the centre
entity:set_ui_slice_fill("stretch", false) -- stretch them, and leave the middle out
```

How a 9-slice sprite fills its edges (along their length) and its centre: `"stretch"` (one quad per cell), `"tile"` (the image's cell repeated at its natural size, the last repeat cropped) or `"tile_fit"` (the repeat count rounded to a whole number and the tiles stretched to fit). An unknown name writes a warning to `LAB.log` and leaves the fill as it was.

`draw_center` is applied only when given: `false` leaves the middle cell out, a hollow frame; `true` draws it. Has no visible effect until the sprite has borders (`set_ui_slice`). No-op with no `SpriteComponent`.

## `set_ui_layout_order(n)` / `get_ui_layout_order()`

```lua
tile:set_ui_layout_order(index)
```

A child's place in its layout container's order: children are laid out by ascending order, ties by their id (see [UI Widgets](../../../../05-ui/03-ui-widgets.md#layout-containers)). `set_ui_layout_order` creates the entity's `UILayoutElementComponent` on first use; `get_ui_layout_order` answers `0` for an entity that has none.

## `set_ui_layout_spacing(x, y)` / `set_ui_layout_padding(left, top, right, bottom)` / `set_ui_layout_columns(n)`

A layout container's Spacing, Padding (UI units) and, for a Grid, its column count (at least 1). They do nothing on an entity that is not a layout container. The container re-lays out at once; nothing is cached.

## `get_ui_rect_at(width, height)`

```lua
local x, y, w, h = panel:get_ui_rect_at(1920, 1080)
```

The rect `get_ui_rect` would answer if the HUD were laid out at a render target of `width` x `height` pixels, in that size's UI units: what a layout looks like at a resolution the game is not running at. Answers `0, 0, 0, 0` for an entity with no `UITransformComponent`.

## `set_ui_interactive(interactive)`

Whether the element takes part in hit testing at all.

An element that is not interactive is never hovered, pressed or clicked, whatever the pointer does. That is what lets a label sit on a button without stealing its clicks, and what stops a decorative panel swallowing a click meant for whatever is behind it.

## `get_ui_rect()`

```lua
local x, y, width, height = entity:get_ui_rect()
```

The element's final rect on screen in UI units: **x, y, width, height**. x and y are its top-left corner, after anchors, pivots, offsets, parents and the viewport scale have all had their say.

It reads as of the last frame's viewport and **resolves the whole HUD per call**, so read it on an event rather than for every element every frame. Answers `0, 0, 0, 0` with no `UITransformComponent`, and before a viewport has been laid out.

This is the value a script needs for something that is not a HUD element itself: drawing a cursor to a slot, or aiming a world-space marker at a screen position.

## `ui_hovered()` / `ui_pressed()` / `ui_clicked()`

```lua
if entity:ui_clicked() then
    log.info("button pressed")
end
```

| Call | True when |
|---|---|
| `ui_hovered` | The pointer is over this element right now |
| `ui_pressed` | A press started on it and the button is still down |
| `ui_clicked` | For exactly one frame, when a press and a release both landed on it |

Hit testing is updated once per frame from the pointer, the same way `input.*` is: there is no separate event or signal system, so a script asking about a click is asking about this frame. **The element's own relative children count too**, so a button reports a click that landed on its label.

Only visible, interactive elements are tested, and where two overlap **only the topmost wins**: draw order, a `UITransformComponent`'s `Layer`, and at the same layer the deeper one, so a button on a panel is hit rather than the panel. Pressing on one element and releasing on another is not a click on either, which is what stops a click being stolen by whatever the pointer passed over on the way.

All three answer `false` for an entity that does not exist or has no `UITransformComponent`.

The same three questions by name rather than by held entity are on the `ui` table, and there they use the element's tag: `ui.is_clicked("PlayButton")`. See [the `ui` table](../../index.md#ui).

## `set_ui_disabled(disabled)` / `is_ui_disabled()`

```lua
button:set_ui_disabled(true)
if button:is_ui_disabled() then ... end
```

The Disabled flag of an entity's widget component: a `UIButtonComponent`, `UIToggleComponent` or `UISliderComponent` (see [UI Widgets](../../../../05-ui/03-ui-widgets.md)). A disabled button is drawn with its Disabled tint, emits no events, is neither hovered nor clicked, and still blocks the pointer. Takes effect with the next frame's hit test. A no-op on an entity that has no button, and `is_ui_disabled` answers `false` for it.

## `ui_changed()`

`true` when the widget raised a `value_changed` event this frame (the same queue `ui.events()` reads). Always `false` for a Button, which has no value.

## `get_value()` / `set_value(value [, notify])`

```lua
local v = slider:get_value()        -- nil for an entity with no widget value
slider:set_value(0.5)               -- raises value_changed next frame
slider:set_value(0.5, false)        -- quietly
```

A widget's value, for the widgets that have one. A toggle's value is `1` (on) or `0` (off); a slider's is its value between Min and Max; a **text field's is its text, a string** (`set_value("text")`, or a number, which becomes its text, held to the field's Filter and Max Length like typing; it raises `value_changed` and not `text_changed`). `get_value` answers `nil` for a widget with none, which includes a Button, and for any entity that is not a widget. `set_value` takes a number, or a boolean for a toggle (`true` is 1), stores it (a slider clamps it to [Min, Max] and snaps it to Step; a toggle in a radio group turns the rest of the group off, and cannot switch off the last one on unless Allow Switch Off) and, unless `notify` is `false`, raises `value_changed` so the next frame's scripts see it like a change the player made. It works on a disabled widget (Disabled is for the pointer), and does nothing for a widget without a value.

## `get_scroll()` / `set_scroll(x, y [, notify])`

```lua
local x, y = list:get_scroll()      -- UI units: how far the content has moved left and up
list:set_scroll(0, 120)             -- returns true when it moved; clamped to the content
```

A UI Scroll List's offset ([UI Widgets](../../../../05-ui/03-ui-widgets.md#scroll-lists)). The content moves up as `y` grows. `set_scroll`
clamps to `0 .. get_scroll_max()` unless the list has Clamp To Content off, ignores an axis the list does not scroll,
and unless `notify` is `false` raises `value_changed` (the normalized position of the main axis) like a change the
player made. Both answer zeros / `false` for an entity that is not a list. The offset is run-time state: not saved, and 0 in the editor outside Play.

## `scroll_to(x, y)`

```lua
list:scroll_to(0, 600)           -- glides there when the list has Smoothing, jumps when it has none
```

Sends a UI Scroll List to an offset the way the wheel does ([UI Widgets](../../../../05-ui/03-ui-widgets.md#scroll-lists)): clamped to the content, and with **Smoothing** above 0 only the target moves and the content glides to it (`get_scroll()` is where it is on the way). With Smoothing 0 it is `set_scroll(x, y)`. Returns whether the list has somewhere to go. A list that is not one answers `false`.

## `get_scroll_max()` / `get_scroll_normalized()` / `set_scroll_normalized(x, y [, notify])`

```lua
local mx, my = list:get_scroll_max()           -- the most it can move; 0 when the content fits
local nx, ny = list:get_scroll_normalized()    -- 0 to 1 of that distance
list:set_scroll_normalized(0, 1)               -- to the end
```

`get_scroll_max` follows the content as it grows, so a list fed from a script can be scrolled to its end after adding items.
`get_value()` / `set_value(v)` on a list are the normalized position of its main axis (vertical, or horizontal for a list that does not scroll vertically).

## `ui_scroll_into_view(child)`

```lua
list:ui_scroll_into_view(row)    -- the least movement that shows `row` completely
```

Called on a scroll list with one of its descendants. Returns `true` when it scrolled; `false` for a child already fully visible, one that is not inside
the list, or an entity that is not a list. A child larger than the window lines its start up with the window's.

## `ui_focus()` / `ui_focused()`

```lua
button:ui_focus()               -- true when it took focus
if button:ui_focused() then ... end
```

Focus this widget / whether it has keyboard or gamepad focus ([UI Widgets](../../../../05-ui/03-ui-widgets.md#focus-and-navigation)). `ui_focus`
is `ui.set_focus` on this entity: `false` unless the project turns on UI focus navigation and the widget can take focus now.

## `ui_begin_edit()` / `ui_end_edit([commit])` / `ui_is_editing()`

```lua
if field:ui_begin_edit() then ... end   -- start typing into it
field:ui_end_edit()                     -- keep the text (value_changed if it changed)
field:ui_end_edit(false)                -- put the text from before the edit back
if field:ui_is_editing() then ... end
```

A [text field](../../../../05-ui/03-ui-widgets.md#text-field) is being edited, or not. `ui_begin_edit` starts it (all the text selected when the field says Select All On Focus) and answers `false` for a field that is disabled or hidden and for an entity that is not a text field; whatever else was being edited ends first, kept. `ui_end_edit` ends **this** field's edit (`commit` defaults to `true`), answers whether it was being edited. While an edit is under way the game's own key reads are silent (`input.is_key_down` and the key bindings answer as if nothing were held).

## Terminal

```lua
local sh = scene.find("Shell")
sh:term_print("<c=#4fd8ff>10.0.0.1</c> connected\nready")   -- two lines, one coloured span
sh:term_set_prompt("root@box:~# ")
sh:term_focus()                        -- take the keyboard (false if read only or disabled)
sh:term_set_input("help", 2)           -- the typed line and the caret
local line = sh:term_get_input()
sh:term_clear()
sh:term_scroll_to_end()
for i = 1, sh:term_line_count() do log.info(sh:term_get_line(i)) end   -- 1 based, no colour tags
sh:term_set_history({ "ls", "cat notes.txt" })
local hist = sh:term_get_history()
sh:term_unfocus()
local on = sh:term_is_focused()
```

A [terminal](../../../../05-ui/03-ui-widgets.md#terminal) (UI Terminal component). Every call does nothing, and logs a warning, on an entity that has none
(`term_is_focused` just answers `false`). `term_print` splits at newlines (a newline at the very end is a terminator, not a blank line) and reads
`<c=#rrggbb>..</c>` colour tags and `<l=payload>..</l>` link spans while the terminal is Rich. The typed line and Tab come back through
`ui.events()` as `"submitted"` and `"complete"` with `text`, and `caret` for `complete`; a click on a link is `"link"` with the payload in `text`.
`sh:term_link_at(x, y)` answers the payload of the link under a HUD pixel, or `nil`. Ctrl+Tab does not raise `"complete"`, and Alt+key and F-keys
keep reaching `input.is_key_down` while a terminal is focused; see [Terminal](../../../../05-ui/03-ui-widgets.md#terminal).

## `ui_adjust(steps)`

```lua
slider:ui_adjust(-1)    -- one notch down
```

Steps a slider by `steps` notches (negative goes back): a notch is the slider's Step, or a tenth of Min to Max when Step is 0. The result is clamped and snapped, raises `value_changed` like a drag and answers `true` when the value changed. `false` for a disabled slider, for a step that changes nothing and for an entity that is not a slider. This is what keyboard and gamepad focus will call.
