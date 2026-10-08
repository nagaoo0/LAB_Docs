---
title: "Keyboard and mouse"
---

Raw key, mouse button, cursor and viewport-pointer reads, plus the capture a first-person camera
needs. See [Lua Scripting](../../index.md).

Input reaches gameplay through ImGui's event queue, which both the editor and the runtime already
pump, so a script reads the keyboard only when no UI widget has captured it: typing a name into a
text field does not walk the player around. That is true of the editor's own text fields **and** of a HUD
[text field](../../../../05-ui/03-ui-widgets.md#text-field) being edited in the game, in the editor's Play mode and the standalone runtime alike. Every keyboard call below answers its inert value for
the whole time a text field has the keyboard, and an unrecognised key name is simply never down,
so neither case is an error and neither is logged.

Prefer the project's [actions](03-actions.md) over raw keys where you have the choice, so the game
stays remappable.

## `is_key_down(name)`

`true` while the key is held, for every frame it stays held, and `false` once it comes up.

```lua
if input.is_key_down("W") or input.is_key_down("UpArrow") then
    entity:translate(entity:get_forward() * 4 * dt)
end
```

Key names are case-insensitive, and they resolve through one shared function (`LAB::KeyFromName`)
rather than through a list kept in the docs or in each caller. That sharing is the point: a key
name in a project's binding, a name a script asks for, and a name the editor's own input tooling
sends all mean the same key, so a spelling that works in one place works in all of them. Names are
the ones ImGui produces; the common ones:

| | |
|---|---|
| Letters and digits | `"A"` to `"Z"`, `"0"` to `"9"` |
| Arrows | `"UpArrow"`, `"DownArrow"`, `"LeftArrow"`, `"RightArrow"` |
| Modifiers | `"LeftShift"`, `"RightShift"`, `"LeftCtrl"`, `"LeftAlt"` |
| Common keys | `"Space"`, `"Enter"`, `"Escape"`, `"Tab"`, `"Backspace"`, `"Delete"` |
| Function keys | `"F1"` to `"F12"` |
| Numpad | `"Keypad0"` to `"Keypad9"`, `"KeypadEnter"` |
| Punctuation | `"Minus"`, `"Equal"`, `"Comma"`, `"Period"`, `"Slash"`, `"Semicolon"` |

**An unknown name is inert, not an error.** It is never down, nothing is logged, and
`is_key_down`, `is_key_pressed` and `is_key_released` all answer `false`. If a key never triggers,
check its spelling rather than looking for an error, and remember that a name resolving to no key
at all is also why `get_key_axis` can read 0 with a key visibly held.

[C++ equivalent](../../../03-cpp/api/input/01-input.md)

## `is_key_pressed(name)`

`true` for the one frame the key went down, `false` on every frame after it, including while the
key is held and while the keyboard's own auto-repeat is firing. `false` for a name that resolves
to no key.

This is the edge a jump, a shot or a menu toggle wants: a script that reads it every frame sees
the press once, on the frame it happened, and sees nothing while the key stays down.

## `is_key_released(name)`

`true` for the one frame the key came up, `false` otherwise and `false` for a name that resolves
to no key.

Useful for charged actions and for anything that should happen when the player lets go: aiming
down sights, releasing a grapple, ending a drag.

## `get_key_axis(negative, positive)`

Two key names collapsed to a single number: `-1`, `0` or `1`.

```lua
local move = input.get_key_axis("S", "W")
entity:translate(entity:get_forward() * move * speed * dt)
```

| Keys | Answer |
|---|---|
| Neither held | `0` |
| The negative key held | `-1` |
| The positive key held | `1` |
| Both held | `0`, so opposed movement keys cancel instead of fighting |

This is Unity's `GetAxis` shape, and it exists so a two-key movement axis is one call rather than
two key reads and a subtract. An unknown name on either side counts as not held, so a typo in the
negative name gives an axis that can only ever read `0` or `1`, and `0` is also the answer for the
whole time a text field has the keyboard.

## `is_mouse_down(button)`

`true` while that mouse button is held. `button` is a number: `0` left, `1` right, `2` middle. Any
other index answers `false`, so an out of range button is inert rather than an error.

While a [virtual pointer](../ui/01-hud-interaction.md) is installed the buttons come from it
rather than from the real device, which is what lets a recorded session click through a HUD.

[C++ equivalent](../../../03-cpp/api/input/01-input.md)

## `is_mouse_pressed(button)`

`true` for the one frame the button went down, `false` otherwise and `false` for an index outside
`0` to `2`.

This is a press **anywhere**, including over the editor's own panels, so it is not by itself a
world click. A script that means "the player clicked the game world" wants one of: the press
inside the game viewport (`input.pointer_in_view()`), the pointer clear of HUD
([`ui.pointer_over()`](../ui/01-hud-interaction.md)), or `scene.pick()`, which answers a miss
while the pointer is over a HUD element and is the whole guard in one call.

## `is_mouse_released(button)`

`true` for the one frame the button came up, `false` otherwise and `false` for an index outside
`0` to `2`. It fires wherever the pointer happens to be, so it does not tell you the press started
on the same thing. For that, a HUD element is asked with
[`ui.is_clicked`](../ui/01-hud-interaction.md).

## `mouse_position()`

The OS cursor position, as a `vec3` with `z = 0`, in window pixels. It comes back as one vector
rather than as two numbers, so it composes with the vector maths everywhere else in the API:

```lua
local m = input.mouse_position()
entity:set_position(entity:get_position() + vec3.new(m.x, 0, 0))
```

This is the raw cursor, not the game viewport's own coordinates, which matters in the editor,
where the viewport is one panel among several. Use `pointer()` for a position inside the game
viewport that the HUD and the world picking also agree on.

A virtual pointer does **not** move this value: the scripted pointer replaces the pointer position
and the buttons for everything that reads the game viewport, while this call keeps reporting the
real cursor.

## `mouse_delta()`

The cursor's movement this frame, as a `vec3` with `z = 0`. It is the value a mouse-look camera
wants, and that is exactly what it is scoped to: it answers `vec3(0, 0, 0)` while the cursor is
**not captured**, rather than the raw movement.

That zero is deliberate. An uncaptured cursor is being used to click panels, and a look script
reading raw movement during that would spin the camera out from under the click. Capture is the
signal that the movement is look input.

## `mouse_wheel()`

The wheel's movement this frame, in notches: one per click of the wheel, the same value the
editor's own viewport reads for zooming. It is `0` on a frame with no scrolling, and it is not
clamped, so a fast flick can report more than one.

While a virtual pointer is installed this answers that pointer's wheel instead, which is how a
script scrolls a list with no mouse involved.

## `set_mouse_captured(captured)`

Turns capture on or off. While captured the cursor is hidden and locked to the window, so
`mouse_delta` keeps reporting movement instead of stopping at the edge of the screen, and the
cursor cannot wander onto another window or the close button.

```lua
function on_create()
    input.set_mouse_captured(true)
end
```

Three things set it back to `false`, so a script that forgets cannot trap the cursor:

- **Escape**, always.
- **Losing the window's focus**, which is an Alt-Tab away.
- **In the editor, losing the Play-mode viewport's focus.** Capture is scoped there so that
  clicking the Properties panel to tweak a light mid-play still works, and a click back on the
  viewport takes it again.

There is no error and no return value: a call while the window is not up yet or with capture
already in the state you asked for does nothing.

## `is_mouse_captured()`

Whether the cursor is captured right now, so a pause menu can tell that Escape has handed the
cursor back. It answers `false` after any of the three releases above, not only after its own
`set_mouse_captured(false)`.

## `pointer()`

Where the HUD's own hit test last saw the pointer, in **UI units**, as a `vec3` with `z = 0`. One
position, in the game viewport (the editor's viewport panel or the runtime window), that a script,
the HUD and the 3D pick all agree on.

UI units are HUD pixels, scaled by the project's UI reference height when one is set, so a label
drawn at the value `scene.world_to_screen` returns lands on the point.

Outside the game viewport the value is meaningless, so test `pointer_in_view()` first.

## `pointer_uv()`

The same position as a fraction of the viewport: `(0, 0)` is its top-left corner and `(1, 1)` its
bottom-right, as a `vec3` with `z = 0`. It is **not clamped**, so a pointer over another editor
panel answers a value outside the range, and before the first frame's hit test has run it answers
`(-1, -1)`.

This is the coordinate `scene.screen_ray` and the editor's own picking take, which is why it is
here rather than on the `ui` table.

## `pointer_in_view()`

`true` while the pointer is inside the game viewport at all: not over another editor panel, and
not before the viewport has a size. `false` is the guard a world click wants, since it is the
cheapest way to say "the pointer is somewhere this game cares about".
