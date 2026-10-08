---
title: "Project actions"
---

Named input bindings from the project, the form a remappable game should use. See
[Lua Scripting](../../index.md).

An action is a name the project declares, paired with the keys, mouse buttons, pad buttons and
pad axes that trigger it: `"Jump"`, `"Move Right"`, `"Fire"`, whatever the project defines. They
are authored in the editor's remap panel and stored in the project's `.lab` file, so they are the
same names a C++ module's `ActionDown` sees.

```lua
if input.action_pressed("Jump") then
    entity:jump(6.5)
end
```

Three properties run through all three calls below, and they all come from the same design:

- **The name is resolved live, on every call.** There is no compiled binding cache, so editing the
  project's binding table takes effect immediately, with no reload and no rebuild.
- **An unbound or misspelled name is inert, not an error.** It is just an action nobody has bound
  yet, so the calls below answer `false` or `0`, exactly the way a typo'd raw key does. A project
  that is not open at all resolves the same way.
- **The name must match the project's own spelling.** "Jump" and "jump" are two different names and
  only one of them is bound, so a script that reads nothing should be checked against the remap
  panel before anything else is suspected.

A project that turns on **UI focus navigation** also gets `ui_up`, `ui_down`, `ui_left`, `ui_right`, `ui_accept`,
`ui_cancel`, `ui_page_prev` and `ui_page_next`, added when it is turned on or loaded and only if no action of that name
exists ([UI Widgets](../../../../05-ui/03-ui-widgets.md#the-input-actions)); they are ordinary actions, remappable like any.

The raw calls on [keyboard and mouse](01-keyboard-and-mouse.md) keep working untouched. Actions sit
alongside them rather than replacing them.

[C++ equivalent](../../../03-cpp/api/input/01-input.md)

## `action_down(action)`

`true` while the action is being held, and `false` for a name that does not resolve.

An action counts as held when any of its bindings is contributing something: a key held, a mouse
button held, a pad button held, a stick pushed past its deadzone, a trigger pulled off its rest.

```lua
if input.action_down("Sprint") then
    speed = runSpeed
end
```

## `action_pressed(action)`

`true` for the one frame the action went from up to down, `false` otherwise, and `false` for a name
that does not resolve.

The edge is tracked by the engine and advanced once per frame for **every** action the project
defines, not only the ones a script happened to ask about. That matters: an action nobody queried
this frame still has its "was down" state advanced, so a later query does not see a stale edge from
while nobody was looking.

## `action_axis(action)`

The action's value as one number, in `-1` to `1`, and `0` for a name that does not resolve.

It is the sum of the action's bindings, clamped to that range. A pair of keys gives `-1`, `0` or
`1`; a gamepad stick gives everything in between. Each binding also carries a **scale**, which is
how one action's opposite directions are wired without a second action: a `"Move Forward"` action
with `W` at `+1` and `S` at `-1` reads `1` forward, `-1` back, `0` with both held or neither.

```lua
local forward = input.action_axis("Move Forward")
local right = input.action_axis("Move Right")
if forward ~= 0 or right ~= 0 then
    local dir = entity:get_forward() * forward + entity:get_right() * right
    entity:translate(dir * speed * dt)
end
```

What each kind of binding contributes:

| Binding | Reads as |
|---|---|
| Key | Its scale while the key is held, `0` otherwise. Read through the same name table as `input.is_key_down`, so a binding to `"Space"` and a script asking for `"Space"` mean the same key. A key binding contributes nothing while a text field has the keyboard |
| Mouse button | Its scale while the button is held |
| Pad button | Its scale while the button is held, on the pad the binding names |
| Pad axis | The axis value times its scale, with triggers remapped to `0` at rest and `1` fully pulled, and sticks read through the same 0.15 deadzone with rescaling that [`gamepad_axis`](02-gamepad.md) uses |

## A complete binding, end to end

The project defines `"Move Forward"` (`W` at `+1`, `S` at `-1`, and the left stick's Y axis at `-1`
so pushing it up drives the same value), `"Move Right"` (`D` at `+1`, `A` at `-1`, left stick X at
`+1`) and `"Jump"` (`Space`, and pad `A`). A script then reads only the names, and every one of
those bindings can be rebound in the editor without the script changing:

```lua
local speed = 4
local turn = 120

function on_update(dt)
    local forward = input.action_axis("Move Forward")
    local right = input.action_axis("Move Right")
    if forward ~= 0 or right ~= 0 then
        local dir = entity:get_forward() * forward + entity:get_right() * right
        entity:translate(dir * speed * dt)
    end

    if input.action_pressed("Jump") then
        entity:jump(6.5)
    end
end
```
