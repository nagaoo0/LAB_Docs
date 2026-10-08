---
title: "Gamepad"
---

Buttons, sticks and triggers on a connected pad. See [Lua Scripting](../../index.md).

The pad index is optional on every call and defaults to `0`, so a one-pad game never has to pass
one. The index is GLFW's joystick slot, so a second controller is `1` and a pad that reports on a
higher slot is reached by saying so. Every call below answers its inert value (`false`, `0` or a
zero vector) for an index with nothing on it, for a stick GLFW has no mapping for, and for a name
it does not recognise, so a missing pad or a misspelled button is never an error.

Unlike the keyboard, the pad is read straight from GLFW rather than out of ImGui's event queue, so
it keeps working while a UI text field has the keyboard. Typing a name does not freeze a pad.

Prefer the project's [actions](03-actions.md) for anything the player might rebind: one
`input.action_axis("Move Forward")` covers a keyboard binding and a stick binding for the same
action, and the remap panel covers both.

## `gamepad_connected([pad])`

`true` when GLFW recognises a pad at that index and can read its state. `false` for an index with
nothing plugged in, and `false` for a joystick GLFW has no mapping for: without a mapping there are
no meaningful button or axis names to read, and treating one as a gamepad would answer junk.

```lua
if input.gamepad_connected() then
    log.info("a pad is on slot 0")
end
```

Every other call here is safe to make without asking first, since an unconnected pad answers the
inert value rather than raising. This one is for a UI that has to say whether a pad was found.

## `gamepad_button(button [, pad])`

`true` while that button is held. `false` for an unrecognised button name and for a pad that does
not answer.

Button names follow the Xbox layout, which is what GLFW's mapping database normalises every pad
to. The PlayStation aliases are accepted too, since that is what is printed on a lot of hardware.
Matching ignores case, spaces, underscores and hyphens, so `"D Pad Up"`, `"dpad-up"` and
`"dpadup"` are one name:

| Button | Aliases |
|---|---|
| `"a"` | `"cross"` |
| `"b"` | `"circle"` |
| `"x"` | `"square"` |
| `"y"` | `"triangle"` |
| `"leftbumper"` | `"lb"` |
| `"rightbumper"` | `"rb"` |
| `"back"` | `"select"` |
| `"start"` | |
| `"guide"` | |
| `"leftthumb"` | The stick click, not the stick |
| `"rightthumb"` | |
| `"dpadup"` `"dpaddown"` `"dpadleft"` `"dpadright"` | |

## `gamepad_button_pressed(button [, pad])`

`true` for the one frame the button went down, `false` otherwise, and `false` for an unrecognised
button name or a pad that does not answer.

GLFW reports the pad's *state*, not its events, so this edge is derived by the engine: it
remembers the previous frame's state for every pad and compares. Two consequences worth knowing:

- A pad that is already connected when play starts, with a button held, reports a press on the
  first frame it is seen, because there is no previous frame to compare against.
- A pad unplugged mid-run forgets its buttons rather than latching one down, so plugging the same
  slot back in does not report a phantom release.

```lua
if input.gamepad_button_pressed("a") then
    entity:jump(6.5)
end
```

## `gamepad_axis(axis [, pad])`

One axis as a number. Sticks travel `-1` to `1`, and triggers are remapped to `0` at rest and `1`
fully pulled, rather than the `-1` to `1` the hardware reports. An unrecognised axis name and a
pad that does not answer both give `0`.

| Axis | Aliases |
|---|---|
| `"leftx"` `"lefty"` | Left stick, `x` right is positive, `y` **down** is positive |
| `"rightx"` `"righty"` | Right stick, the same convention |
| `"lefttrigger"` | `"lt"` |
| `"righttrigger"` | `"rt"` |

Sticks get a **0.15 deadzone with rescaling**: anything inside it reads `0`, and the rest is
stretched back over the full range rather than starting part way along, which is what stops a
resting stick from drifting a player across the room.

Note the vertical direction: `"lefty"` is positive when the stick is pushed **down**, matching
the hardware. Movement code wants `gamepad_stick` below, which flips it.

## `gamepad_stick(which [, pad])`

A whole stick as a `vec3`, `x` and `y` with `z = 0`, so it drops straight into movement maths:

```lua
local move = input.gamepad_stick("left")
entity:translate(entity:get_forward() * move.y * speed * dt)
entity:rotate(vec3.new(0, 0, -move.x * turn * dt))
```

`which` picks the stick: any name starting with `r` or `R` is the right stick, and anything else is
the left one, so `"left"` and `"right"` are all most scripts ever need to write. Both axes come
through the same 0.15 deadzone with rescaling as `gamepad_axis`, and **y is negated** so that up
is positive, which is what every movement and camera script wants.

A pad that does not answer gives `vec3(0, 0, 0)` rather than nil, so a script can read the stick
without checking whether anything is plugged in.

## What is not here

There is no gamepad call in the C++ ABI. A native module reaches the pad through the project's
actions, which is the better answer anyway: a gamepad binding and a keyboard binding for the same
action are one `ActionAxis` call. The ABI's own inputs are described in
[C++ input](../../../03-cpp/api/input/01-input.md).
