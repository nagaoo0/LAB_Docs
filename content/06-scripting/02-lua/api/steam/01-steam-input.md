---
title: "Steam Input (steam_input)"
---

Five calls that read controllers through Steam Input, all of them inert wherever Steam is not
there. See [Lua Scripting](../../index.md).

Steam Input is Steamworks' controller layer. Bindings are per player and per controller, and they
are configured by the player in Steam's own controller configurator against an in-game action
manifest, so the game reads named **actions** rather than buttons. This table is that surface.

It sits **beside** `input.gamepad_*`, deliberately not underneath it: a literal button name or a
raw pad index is a different binding model from an action resolved by Steam, and a game that wants
Steam's rebinding asks for it by name.

## Steam is a capability, never a requirement

The Steam library is resolved at run time and a checkout without it configures, compiles and runs
with no Steam at all. Failure to initialise is logged at info, not error, and **nothing here
fails when it is missing**: every call returns its inert default (`false`, or a zero vector) and
does nothing. That is the normal case in the editor, where Steam is usually not initialised at
all, so there is nothing to check first and nothing to guard against.

`steam_input.available()` is for branching behaviour, not for protecting a call. Checking it to
avoid a crash would be checking the wrong thing.

**With Steam present but no action manifest authored**, Steam Input still initialises
(`available()` answers true) and resolves no actions, so every query answers false or zero. That
is a legitimate state during development, before a manifest exists, not a broken one.

See [Steam integration](../../../../STEAM.md) for the manifest, the DLL loading and what the
editor's Diagnostics panel shows.

## `steam_input.available()`

```lua
if steam_input.available() then
    steam_input.activate_action_set("Gameplay")
end
```

True when Steam Input initialised. False with no Steam (library missing, or an export the loader
could not bind), and false for the whole session once it has failed: every other call then answers
its inert default.

## `steam_input.controller_connected()`

True when at least one controller is connected and tracked by Steam Input.

It takes no arguments, unlike `input.gamepad_connected(pad)`: Steam Input is a per-player API and
this is the single-player convenience, "is anything plugged in at all". False whenever Steam Input
is not up.

## `steam_input.activate_action_set(name)`

Makes the named action set the active one on every connected controller.

```lua
-- Switching modes: Steam Input is designed around swapping sets, not
-- toggling individual bindings.
steam_input.activate_action_set("Gameplay")
```

An action set is how a Steam Input game ships two layouts, say "Menu" and "Gameplay", and swapping
one is cheap enough to do whenever a game mode changes. Returns nothing. A name nothing resolves,
or no Steam Input: nothing happens, silently.

## `steam_input.action_down(name)`

True while the named digital action is held on **any** connected controller, the same
single-player convenience `input.gamepad_button` has with its pad index defaulted.

False when the name resolves to no action, when no controller is connected, when nothing is
pressed, and when Steam Input is not up. All four are one answer, so a name typo is a dead action
rather than an error, the same tolerance a mistyped input action name already gets.

```lua
if steam_input.action_down("Jump") then
    entity:jump(5)
end
```

## `steam_input.action_axis(name)`

```lua
local move = steam_input.action_axis("Move")
entity:move(vec3.new(move.x, move.y, 0) * speed)
```

The named analog action's two axes as a `vec3`, with `z` unused. It returns a vector rather than
two numbers so it composes with the `vec3` helpers and with multiple assignment, the same choice
`input.mouse_position` makes.

A one-axis action, a trigger for instance, drives `x` only: Steam Input does not distinguish 1D
from 2D analog actions at the API level, only in how many axes the binding drives.

With several controllers connected, the first one reporting the action active supplies the value.
No controller, no such action, or no Steam Input: `vec3.new(0, 0, 0)`.

## There is no `steam_input.action_pressed`

`action_down` is the only digital query on this table. There is no edge-triggered variant here, so
a script that needs "this frame the button went down" keeps the previous frame's value itself:

```lua
local wasDown = false

function on_update(dt)
    local down = steam_input.action_down("Jump")
    if down and not wasDown then
        entity:jump(5)
    end
    wasDown = down
end
```

`input.action_pressed` is that edge for the project's own Input Actions table, which is the local
binding table rather than Steam's, so the two are not interchangeable: a press read through Steam
Input is only ever a level here.

## What is not here

No glyphs (nothing renders a button icon yet), no haptics, no motion data, and no way to rebind
from Lua. **Rebinding happens in Steam's own controller configurator**, which is the entire point
of using Steam Input over a hand-rolled binding table for players who have Steam; there is no
LAB-side remap UI for it and none is planned.

The two other input surfaces are unchanged and still the right call most of the time:
`input.gamepad_*` reads a controller directly and needs no Steam, and `input.action_*` reads the
project's own Input Actions table for players who do not have it.
