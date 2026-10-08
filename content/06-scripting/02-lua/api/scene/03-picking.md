---
title: "Picking and screen projection"
---

Turning the pointer into a world ray, hitting rendered geometry with it, and projecting a world
point back onto the screen. See [Lua Scripting](../../index.md).

Two things separate these from the physics casts. They hit **rendered triangles**, not colliders,
so a pick needs no collider on anything, and a mesh is hit exactly where it is drawn. And the
pointer they use is where the HUD's own hit-test last saw it, so a script, the HUD and the 3D pick
all agree on one position in the game viewport, which is the viewport panel in the editor and the
window in a build.

## `scene.pointer_ray()`

```lua
local origin, direction = scene.pointer_ray()
if origin then
    local hit = scene.raycast_meshes(origin, direction, 100)
    if hit.hit then
        log.info("pointing at " .. hit.entity:get_name())
    end
end
```

The ray through the pointer, as two vec3s: an origin at the camera, and a normalised direction.

**Answers nil when there is no camera to shoot from**, so the check above is on `origin` before
`direction` is used.

## `scene.screen_ray(u, v)`

The same ray, through a point given as `u` and `v` across the game viewport in 0..1, with the
top-left corner as the origin. Answers origin, direction, or nil when there is no camera.

This is the form to build a ray from a UI position or a scripted click: `input.pointer_uv()` is
the pointer in exactly these coordinates, while `world_to_screen` below answers in the other set
of units. There is no vertical flip to apply, `v = 0` is the top of the viewport, which is what
the renderer's own projection already accounts for.

```lua
local origin, direction = scene.screen_ray(0.5, 0.5)
if origin then
    local hit = scene.raycast_meshes(origin, direction, 200)
    log.info(tostring(hit.hit))
end
```

## `scene.world_to_screen(point)`

```lua
local x, y, in_front = scene.world_to_screen(target:get_world_position())
if in_front then
    log.info(x .. ", " .. y)
end
```

Three values: an `x` and a `y` in **UI units** (HUD pixels, scaled by the project's UI reference
height when one is set), then whether the point is in front of the camera. UI units are the units
the HUD is placed in, so a marker positioned at the answer lands on the point.

**When the third answer is false, `x` and `y` are `0`**, not a clamped or off-screen position.
Test the boolean first and never read the coordinates without it. False covers a point behind the
camera and a scene with no camera at all.

## `scene.raycast_meshes(origin, direction [, maxDistance])`

A ray against the scene's rendered meshes, from an origin and a direction you choose. Answers the
closest hit as a table:

```lua
local hit = scene.raycast_meshes(origin, direction, 60)
if hit.hit then
    log.info(hit.entity:get_name() .. " at " .. tostring(hit.distance))
end
```

| Field | |
|---|---|
| `hit` | `true` when something was hit. Everything below is absent on a miss |
| `entity` | The entity whose mesh was hit |
| `position` | The world-space point of impact |
| `normal` | The surface normal at that point |
| `distance` | How far along the ray it is, in world units |

A miss is `{ hit = false }` with the rest absent, the same shape `physics.raycast` reports, so
`if hit.hit then` is the whole idiom and there is nothing meaningless to read on a miss.

The direction does not have to be normalised, and omitting `maxDistance` searches the whole scene.
A zero-length direction is a miss. There is no ignore argument: the ray is tested against every
mesh in the scene, the caster's own included, so a ray that starts inside a mesh meets the far
side of it.

**Skeletal meshes are tested against their bind pose**, not the pose they are currently animated
into. That is correct for picking and much cheaper than skinning every vertex on the CPU per call,
so a pick that lands on the model's rest shape rather than on the shape on screen is the expected
behaviour.

The module ABI's own `Raycast` is the physics world's ray and needs colliders (see
[Physics](../../../03-cpp/api/physics/01-physics.md)); there is no mesh-ray counterpart there, so this is a
Lua-facade call.

## `scene.pick([throughUI])`

The mesh under the pointer, as the same hit table:

```lua
function on_update(dt)
    if input.is_mouse_pressed(0) then
        local hit = scene.pick()
        if hit.hit then
            log.info("clicked " .. hit.entity:get_name())
        end
    end
end
```

Answers `{ hit = false }` when there is nothing under the pointer, when the pointer is outside the
game viewport, and **when the pointer is over a HUD element**. A click on a button therefore does
not also click whatever is behind it, which is what almost every caller wants. Pass `true` to see
through the HUD and pick the world regardless.

It is the same mesh ray `raycast_meshes` casts, over an unlimited distance, through the pointer's
own ray, so no collider is needed on anything and the pick lands exactly where the mesh is drawn.
