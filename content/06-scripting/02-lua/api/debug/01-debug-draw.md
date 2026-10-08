---
title: "Debug draw (debug)"
---

Six calls that queue wireframe shapes into the viewport, for one frame or for a few seconds. See
[Lua Scripting](../../index.md).

## How the calls work

They all have the same shape: they queue a shape onto the scene and return **nothing**. What is
drawn is a wireframe, drawn over the frame's geometry and depth tested against it, so a shape
behind a wall is hidden by the wall.

Three rules cover every one of them:

- **The last argument is `duration`, the one before it is `color`**, and both are optional. The
  colour is a `vec3` with components in 0..1 (`vec3.new(1, 0, 0)` is red). The duration is
  seconds. A shape with no duration is drawn for the single frame it was queued on, which is why
  the usual idiom is calling it from `on_update` every frame; a shape with a duration keeps
  drawing until the seconds run out.
- **`box` and `capsule` take a `rotation` between the colour and the duration.** It is a `quat`
  (`quat.from_euler(vec3.new(0, 0, 45))`), and it orients the shape. This is the one irregularity
  in the argument lists, and it means the third argument of those two calls is a colour, so
  `debug.box(center, extent, 1.0)` is a mistake rather than a one-second box.
- **Nothing is remembered between frames.** The queue is drained every tick, so a shape queued
  once from `on_create` is drawn on that first frame and never again. Per-frame shapes are queued
  from `on_update`; a shape with a duration is the one exception, surviving on its own until its
  seconds run out.

```lua
function on_update(dt)
    debug.line(entity:get_position(), target, vec3.new(1, 0, 0))
    debug.sphere(target, 0.5)
end
```

## `debug.line(a, b [, color] [, duration])`

A line from `a` to `b`. Default colour **white**. One line segment.

```lua
debug.line(vec3.new(0, 0, 0), vec3.new(0, 0, 2), vec3.new(0, 0, 1), 0.5)
```

## `debug.arrow(from, to [, color] [, duration])`

A shaft from `from` to `to` with a four-line head at the `to` end. Default colour **yellow**.

The head is sized from the arrow's own length (a quarter of it, capped at 0.25 world units), so a
short arrow still reads as an arrow rather than a bare line. Five line segments.

## `debug.box(center, halfExtent [, color] [, rotation] [, duration])`

A wireframe box centred on `center`. Default colour **green**.

`halfExtent` is the half size on each axis, so `vec3.new(1, 1, 1)` is a box two units across in
every direction. `rotation` orients it; without one it is axis aligned. Twelve edges.

```lua
debug.box(entity:get_world_position(), vec3.new(0.5, 0.5, 0.5),
    vec3.new(1, 0.5, 0), entity:get_rotation_quat())
```

## `debug.sphere(center, radius [, color] [, duration])`

A wireframe sphere centred on `center`. Default colour **cyan**.

Three rings of 24 segments, one per axis: 72 line segments, which makes this the most expensive
shape here.

## `debug.capsule(center, radius, halfHeight [, color] [, rotation] [, duration])`

A wireframe capsule centred on `center`. Default colour **orange**.

`radius` is the radius of the round ends and `halfHeight` is half of the cylindrical section, so
the whole shape is `2 * (halfHeight + radius)` long. The capsule's axis is its own local Z, which
for an unrotated shape is world Z (the engine is Z up), and `rotation` tilts it from there. Its
wireframe is two end circles, four side lines and two swept domes: 84 line segments.

## `debug.diamond(center, radius [, color] [, duration])`

A wireframe octahedron centred on `center`, with its six points on the axes at `radius`.
Default colour **magenta**. Twelve edges.

```lua
-- A hit marker: a diamond where the projectile landed, for two seconds.
debug.diamond(hit.position, 0.25, vec3.new(1, 1, 1), 2)
```

## What they cost

Every call adds line segments to the frame's debug batch, which is uploaded and drawn as part of
the viewport's frame. A shape queued from `on_update` is that cost every frame, and the shapes are
not small: 72 segments for a sphere, 84 for a capsule, 12 for a box or a diamond, 5 for an arrow,
1 for a line. A handful of markers is nothing; a hundred spheres per frame is real frame time.

The batch is capped at 65536 vertices, 32768 line segments, shared with every other debug overlay
the viewport is drawing (colliders, the navmesh, the skeleton). Past the cap, lines are dropped
**silently**, beginning with the ones past it, so a script that draws enough shapes can lose the
last ones it queued without any error.

**These are development aids.** They are drawn by the editor's viewport layer, which drains the
scene's queue each frame while the game plays (`BuildScriptDebugDrawOverlay` in
`VulkanEngineLayer.cpp`); the standalone runtime layer opens no debug line pass at all, so this
output does not exist in a packaged build. A picture on screen, not a value any script or test can
read back.

The node editor's own **Debug Draw Line**, **Debug Draw Arrow**, **Debug Draw Box**,
**Debug Draw Sphere**, **Debug Draw Capsule** and **Debug Draw Diamond** nodes queue the same
shapes from a visual script, so a graph and a script can share a debug picture.
