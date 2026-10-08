---
title: "HUD interaction"
---

Hover, press and click for HUD elements, by tag or by entity, plus the scripted pointer a test or a
replay drives. See [Lua Scripting](../../index.md).

Hit testing is updated **once per frame from the pointer position**, by whichever layer owns the
game viewport (the editor's Play-mode viewport, or the runtime's window). The flag calls below have
no callbacks: a script asking whether a button was clicked is asking about this frame, the same way
it asks about a key. The hit test runs after the scripts for that frame have had their turn, so what
a call here answers is the state the last hit test left behind. For elements that carry a widget
component (a button), the same hit test also fills a per-frame event queue: [`events()`](#events)
and [`changed(name)`](#changedname) below, described in [UI Widgets](../../../../05-ui/03-ui-widgets.md).

Three rules decide who wins it:

- **Only visible, interactive elements take part.** An element with `Interactive` off is never
  hovered, pressed or clicked, whatever the pointer does, which is what makes a decorative panel
  not swallow a click meant for the button behind it. A label on a button is the usual case for
  turning it off, so that the button underneath is the thing the pointer is over.
- **Only the topmost element wins** where two overlap: a higher `Layer` wins, and at the same layer
  the deeper element wins. That is the same tie-break draw order uses, so what is on top is what
  is clickable.
- **Names are element tags**, the same lookup `scene.find` uses, and the first entity carrying that
  tag wins. An unknown name answers `false` rather than an error, the same tolerance a mistyped key
  has, so a button that never fires is usually a name that does not match.

## `is_hovered(name)`

`true` while the element with that tag is the one the pointer is over, and `false` for a name no
entity carries.

It is the element itself, not a parent: if the pointer is over a label inside a button, the label
is hovered and the button is not, unless the label is set non-interactive. (An entity's own
`entity:ui_hovered()` is the ancestor-tolerant form, which is what a button wants when it counts
its label as part of itself.)

[C++ equivalent](../../../03-cpp/api/ui/02-hud-interaction.md)

## `is_pressed(name)`

`true` while the element is under the pointer and the left mouse button is down, and `false` for a
name no entity carries.

Note what this is made of: the hover test, plus the button. It does **not** ask where the press
started, so a press that began elsewhere and was dragged onto the element reads pressed as well.
The by-entity `pressed()` below is the one that remembers where a press landed.

```lua
if ui.is_pressed("SprintButton") then
    speed = runSpeed
end
```

## `is_clicked(name)`

`true` for **exactly one frame**: the frame on which a press and a release both landed on that
element. `false` on every other frame, and `false` for a name no entity carries.

Pressing on one element and releasing on another is not a click on either, which is what stops a
click being stolen by whatever the pointer passed over on the way. The flag is cleared at the start
of each frame's hit test, so a script reading it every frame sees the click once.

```lua
function on_update(dt)
    if ui.is_clicked("PlayButton") then
        scene.open_scene("scenes/level1.Lscene")
    end
end
```

## `hovered()`

The element the pointer is over, as an entity, or `nil` when there is none. It is the same answer
`is_hovered` compares a name against, and the form to use when the script already has the element
in hand rather than a name for it.

```lua
local el = ui.hovered()
if el then
    el:set_ui_color(vec3.new(1, 1, 1))
end
```

[C++ equivalent](../../../03-cpp/api/ui/02-hud-interaction.md)

## `pressed()`

The element a press landed on, as an entity, while the button is still down. `nil` when no button
is down.

This is the press-start element, and it stays that element even if the pointer has since been
dragged off it, which is the pair of answers a drag wants: `pressed()` says what is being held and
`hovered()` says what is under the hand.

## `clicked()`

The element a click just completed on, as an entity, for that one frame. `nil` on every other
frame, and `nil` when the click landed on nothing.

This is the form a dispatcher wants, one script handling every button rather than one per button:

```lua
local el = ui.clicked()
if el then
    log.info("clicked " .. el:get_name())
end
```

The three entity forms are the raw hit-test answers, so a parent is not returned for a click that
landed on its child. An entity's own `entity:ui_clicked()` is the one that counts a descendant, so
a button with an interactive label still reports the click on itself.

## `pointer_over()`

`true` while the pointer is over any interactive HUD element at all, **including a disabled button**: it is deaf to the pointer but it is not a hole, so a click on it does not reach what is behind it.

This is the test a world click makes before acting, so that clicking a button does not also click
what is behind it in the 3D scene:

```lua
if input.is_mouse_pressed(0) and not ui.pointer_over() then
    local h = scene.pick()
    if h.hit then
        h.entity:set_color(vec3.new(1, 0, 0))
    end
end
```

`scene.pick()` makes the same test itself, and answers a miss while the pointer is over a HUD
element, so a script using it needs no guard. This call is for a script doing its own ray work
where that guard would be lost.

[C++ equivalent](../../../03-cpp/api/ui/02-hud-interaction.md)

## `events()`

Everything that happened to a widget this frame, oldest first, as an array of tables:

```lua
for _, e in ipairs(ui.events()) do
    if e.type == "clicked" then
        log.info(e.name .. " clicked")        -- e.entity is the entity, e.value a number
    end
end
```

`type` is one of `"hover_enter"`, `"hover_exit"`, `"pressed"`, `"released"`, `"clicked"`, `"value_changed"`, `"submitted"`, `"focus_gained"`, `"focus_lost"`, `"cancel"`, `"page_prev"`, `"page_next"`, `"text_changed"`, `"complete"` and `"link"`; `name` is the element's tag. A terminal's `submitted` and `complete` also have `text` (the line) and, for `complete`, `caret`; a `link` (a click on a `<l=payload>..</l>` span in a terminal's output) has the payload in `text`. Only elements with a widget component (a button, toggle, slider, scrollbar or text field) produce events, so an empty table is the normal answer for a HUD with none. The queue is rebuilt once per frame, right after the hit test, and stays untouched until the next one: a script sees each event **exactly once**, on the first `on_update` after it happened, and every script sees the same list. `value` is documented per type in [UI Widgets](../../../../05-ui/03-ui-widgets.md#the-event-queue). Returns a new table each call.

## `features()` / `hand_cursor()`

```lua
if ui.features().rich_links then ... end
```

`features()` is a table of what this engine's HUD can do, so a game can run on an older build: `rich_links` (terminal link spans raise `"link"`
events, `term_link_at` exists) and `key_passthrough` (F-keys, Alt+key and Ctrl+Tab still reach key reads while a terminal is focused). A key
the build does not know is `nil`. `hand_cursor()` is true while the pointer is over a terminal link. See [Terminal](../../../../05-ui/03-ui-widgets.md#links).

## `changed(name)`

`true` when the element tagged `name` raised a `value_changed` this frame, `false` for one that did not and for a name nothing carries. The call for sliders and toggles; always `false` for a button.

## `size()`

The game viewport's own size **in UI units** (width, height), then the scale from UI units to
pixels. Three values, so take what you need:

```lua
local w, h, s = ui.size()
log.info("the HUD is " .. w .. " by " .. h .. " UI units, at " .. s .. "x")
```

`w * s` is the viewport in real pixels, which is what a resolution readout wants. Everything else
here works in UI units, so layout maths wants `w` and `h` as they are. Before the first frame's hit
test has run the size is not known yet and this answers `0, 0, 1`.

[C++ equivalent](../../../03-cpp/api/ui/02-hud-interaction.md)

## `focused()` / `set_focus(target)` / `navigate(direction)` / `accept()` / `cancel()` / `cancelled()`

```lua
local f = ui.focused()                   -- the focused entity, or nil
ui.set_focus("StartButton")              -- a tag name, an entity, or nil to clear
ui.navigate("down")                      -- what the ui_down action does
if ui.cancelled() then close_menu() end  -- a cancel event this frame (the ui_cancel action, or ui.cancel())
```

Keyboard and gamepad focus, for projects that turn on **UI focus navigation** ([UI Widgets](../../../../05-ui/03-ui-widgets.md#focus-and-navigation));
with it off they answer `nil` / `false` and do nothing. `set_focus` and `navigate` answer whether they did anything;
`accept()`, `cancel()`, `page_prev()` and `page_next()` do what their input actions do, which is how a test or a
game with its own input scheme drives the focus. Focus changes at once; the events they raise (`focus_gained`,
`focus_lost`, `clicked`, `submitted`, `cancel`) join the next frame's queue.

## `set_virtual_pointer(u, v [, buttons [, wheel [, wheel_h [, shift]]]])`

Installs a scripted pointer standing in for the mouse, so a test or a replay can press real buttons
through the real UI code with no mouse at all.

| Argument | |
|---|---|
| `u`, `v` | Across the viewport, `0` to `1`. Not pixels, and not UI units |
| `buttons` | A bit mask: `1` left, `2` right, `4` middle. Defaults to `0`, no button down |
| `wheel` | Wheel steps, in notches, positive up. Defaults to `0` |
| `wheel_h` | A horizontal wheel notch, positive toward the left (the content moves right), as the real device's. Defaults to `0` |
| `shift` | Whether Shift is held; a scroll list turns vertical notches sideways with it. Defaults to `false` |

A pointer over the middle of the screen is `ui.set_virtual_pointer(0.5, 0.5)`, and holding the left
button there is `ui.set_virtual_pointer(0.5, 0.5, 1)`:

```lua
ui.set_virtual_pointer(0.5, 0.5)
ui.set_virtual_pointer(0.5, 0.5, 1)
```

Four things to know:

- **It takes effect on the next frame**, like a real mouse move, so a script that sets it and then
  tests a click in the same call sees the previous frame's state.
- **It replaces the mouse entirely** while it is installed: the HUD hit test, `ui.hovered`,
  `ui.clicked`, `input.pointer` and `input.pointer_uv`, `input.is_mouse_down` and its two edge
  forms, `input.mouse_wheel`, and the ray `scene.pointer_ray` casts all read it rather than the
  real device. That is what makes a replayed session reproducible. (`input.mouse_position()` is the
  exception: it keeps reporting the real cursor.)
- **A click is two frames.** The button edges come from comparing that frame's mask against the
  previous frame's, so the press is a frame whose mask holds the left button (`1`) and the release
  is the frame after it whose mask has cleared it. Both frames have to land on the same element,
  exactly as a real click does. The HUD's own press and click read the left button only, so a
  script driving a button keeps `1` in the mask for the press.
- **Wheel steps passed before the next frame are added up**, and the total is what that frame's
  hit test sees.

**Clear it when the replay is over.** A virtual pointer left installed is a session whose mouse no
longer works, with nothing on screen to explain it.

[C++ equivalent](../../../03-cpp/api/ui/02-hud-interaction.md)

## `clear_virtual_pointer()`

Removes the scripted pointer, so the real mouse takes over again on the next frame. Nothing is
returned, and clearing when none is installed does nothing.

## A replayed click, end to end

Hover, press, release, then read the click. The press and the release are separate frames, and the
click is visible to the script on the frame after the release:

```lua
local frame = 0

function on_create()
    ui.set_virtual_pointer(0.5, 0.62)
end

function on_update(dt)
    frame = frame + 1
    if frame == 3 then
        ui.set_virtual_pointer(0.5, 0.62, 1)
    elseif frame == 5 then
        ui.set_virtual_pointer(0.5, 0.62, 0)
    elseif frame == 6 then
        log.info("clicked: " .. tostring(ui.clicked() ~= nil))
        ui.clear_virtual_pointer()
    end
end
```
