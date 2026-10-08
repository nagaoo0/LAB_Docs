---
title: "Post-process parameters"
---

Runtime values for the scene's Post Process Material, driven from a script. See
[Lua Scripting](../../index.md).

A scene can name a Post Process Material, a graph that runs over the finished image rather than
over a mesh. Its **Scalar** and **Vector3 Parameter** nodes are the values a game changes while
it plays: a fade, a tint, a distortion amount. Each parameter is addressed by the name typed on
its node.

The three calls on the `render` table read and write those values. They belong to the scene
rather than to the renderer, so a play session's values vanish when play stops and the graph's
authored defaults come back. For authoring the graph itself, see
[Post Process Materials](../../../../03-rendering/01-materials.md#post-process-materials).

## `set_post_param(name, value)`

Sets one parameter, by its node name.

- A **number** sets a Scalar, and is also splatted across a Vector3, so one number can drive a
  Vector3 parameter uniformly.
- A **`vec3`** sets a Vector3. A Scalar parameter reads its `x`, so the same call works for
  either node type.

Anything else is refused: a nil, a string or a table logs a warning naming the parameter and
stores nothing. The call answers nothing.

```lua
render.set_post_param("Fade", 0.5)
render.set_post_param("Tint", vec3.new(1, 0.8, 0.6))
```

The value travels to the shader in the renderer's push-constant block, so the graph is not
recompiled and setting a parameter every frame costs nothing beyond the write.

## `get_post_param(name)`

The parameter's value as the scene currently holds it: a number for a Scalar, a `vec3` for a
Vector3, whichever shape the last `set_post_param` stored there.

**Answers `nil` for a name that has not been set on this scene.** A parameter left at its
authored default is not set, so it answers `nil` too: the default itself lives in the graph's
node, not here.

This reads the scene's own store, not anything back from the GPU, so it cannot tell you what the
last frame drew with.

```lua
local fade = render.get_post_param("Fade")
if fade == nil then
    fade = 1        -- never set this session, so the authored default applies
end
```

## `clear_post_params()`

Drops every value the scene has stored, so the graph's authored defaults apply again. No
arguments, and it answers nothing. Values die with the play session anyway, so this is for
resetting mid-run without stopping play.

```lua
function on_create()
    render.clear_post_params()
end
```

## What can actually reach the shader

The parameter values travel in the renderer's push-constant block, which holds **12 float
slots**. A Scalar takes one slot and a Vector3 takes three, packed in the graph's own parameter
order, and the whole block is exactly the 128 bytes Vulkan guarantees.

**A parameter that does not fit keeps its authored default, and `set_post_param` cannot change
it.** The graph's compile writes one warning naming it: "parameter 'Name' does not fit the 12
runtime float slots; it keeps its authored default and render.set_post_param cannot change it".
The fix is authoring fewer or smaller parameter nodes, not calling differently.

Worth knowing, because it hides: setting such a parameter still stores it and `get_post_param`
still reads it back, so the round trip looks like it works while the shader never sees the
value.

A name the active graph does not declare behaves the same way in reverse. It is stored and
readable by `get_post_param`, but nothing in the shader is named that, so nothing visible
happens.

## A complete example

A hit flash driven from the scene's post-process graph, faded out over a second.

```lua
local flash = 0

function on_overlap_begin(other)
    flash = 1
end

function on_update(dt)
    if flash <= 0 then return end

    flash = math.max(0, flash - dt * 2)
    render.set_post_param("Flash", flash)
end
```

The graph side is a Scalar Parameter node named `Flash`, wired into whatever the effect needs,
with its authored default used for every frame the script has not set it.
