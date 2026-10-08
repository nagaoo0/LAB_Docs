---
title: "Splines"
---

Queries against a `SplineComponent`, in world space. See [Lua Scripting](../../index.md) for how a script runs.

A spline is a curve an entity carries: a list of points, whether it closes, and a width profile. The same content a `SplineMeshComponent` gets extruded from. These calls read it, which is what a script needs to run something along a path, aim it, or ask how far along it is.

Everything here is in **world space**, distances in **metres** along the curve, and frames as world vectors. That is exactly what the same calls give a native module, so a path authored once behaves the same whichever language follows it.

The entity these are called on is the one carrying the spline, which is usually not the entity moving: a follower samples the rail.

[C++ equivalent](../../../03-cpp/api/spline/01-splines.md)

## `has_spline()`

```lua
if rail:has_spline() then
    ...
end
```

Whether the entity carries a spline at all. Every other call here answers a zero, a false or a plain frame for an entity that does not, so this is how you tell "this entity has no spline" from "the spline is somewhere unexpected".

## `spline_length()`

The curve's total length in metres, and `0` with no spline.

The length is **arc-length accurate**, not an estimate from the control points: the spline keeps a table of the real curve, so a sample at 30 metres is 30 metres along it, whatever the points do in between.

## `spline_closed()`

Whether the curve joins its last point back to its first. A closed spline's length includes the closing span, and sampling past its end wraps rather than clamps. Answers `false` with no spline, which is the same answer an open spline gives, so pair it with `has_spline()` where the difference matters.

## `spline_sample(distance)`

```lua
local position, forward, right, up, width = rail:spline_sample(12.5)
```

Position, forward, right, up and width at a distance along the curve, as five values.

The frame is an orthonormal set of **world** vectors: `forward` is the direction of travel along the curve, and `right` and `up` are the other two axes. "Forward" is the curve's own direction, not the sampled entity's, so a thing following a spline gets its heading without computing one. `width` is the per-point width profile the spline carries, the one a `SplineMeshComponent` extrudes from: a constant-width spline answers that constant, a varying one the interpolated value at this distance.

`distance` is in world metres, so `0` is the first point and `spline_length()` the last. On a **closed** curve it wraps, on an **open** one it clamps to the ends, so a caller advancing a distance does not have to handle the ends itself.

With no spline on the entity, the answer is a plain frame rather than nothing: position `(0, 0, 0)`, forward `(0, 1, 0)`, right `(1, 0, 0)`, up `(0, 0, 1)` and width `0`.

### Following a curve

The idiomatic shape: keep a distance, advance it by speed times the timestep, and sample.

```lua
local rail = scene.find("Rail")
local distance = 0
local speed = 6

function on_update(dt)
    if not rail or not rail:has_spline() then
        return
    end

    distance = distance + speed * dt
    local length = rail:spline_length()
    if rail:spline_closed() then
        distance = distance % length
    elseif distance > length then
        distance = length
    end

    local position, forward = rail:spline_sample(distance)
    entity:set_position(position)
end
```

`set_position` is local and the sample is world, so this is exact when the follower and the rail share a parent (a scene root, typically), and needs the follower's own parent transform accounted for otherwise.

Advancing by distance rather than by a per-segment index is what keeps the speed constant. A curve with a tight corner has more control points in the same length, so stepping by point would speed up in the corners and slow down on the straights.

## `spline_closest(point [, hint [, window]])`

```lua
local along, lateral, vertical = rail:spline_closest(entity:get_world_position())
```

The distance along the curve nearest `point`, then that point's offset from the curve **in the curve's own frame**: lateral (along `right`) and vertical (along `up`). Three values, all in metres.

The lateral and vertical outputs are the right values for a question like "how far off the centre line is this thing", and the wrong ones for "how far east of it is". The distance is the one to feed back into `spline_sample`.

`hint` and `window` are the cheap form, both in metres: pass the last answer as the hint and a window of a few metres, and only that stretch is searched first. That matters when this is called every frame for several entities, because the exact search walks the whole curve. It is what keeps a racer on its own part of a track that crosses over itself, where the nearest point of the whole curve is on the other lap.

The hint is a preference, not a restriction: a window that finds nothing falls back to a search of the whole curve, so the answer is never worse than an unhinted one, only usually cheaper. **Omitted, or negative, hint and window search the whole curve from the start**, which is what a caller with no previous answer passes.

With no spline on the entity, the answer is `0, 0, 0`.

## `spline_point_count()`

How many control points the spline was authored with, and `0` with no spline.

This is a structural count, not a measure of the curve's detail: the sampled curve between two control points is a smooth interpolation, so a spline with four points is still a smooth loop.

## Scaling

**A scaled entity is assumed to be scaled uniformly**, and its local `+X` axis length is taken as the scale. So a spline on an entity scaled `(2, 2, 2)` is twice as long and its frames are twice as wide; one scaled `(2, 1, 1)` is treated as scale 2, because a non-uniform scale would need a different curve, not a stretched one.

Distances and widths are reported in world metres with that scale folded in, so `spline_length`, a `spline_sample` distance and the lateral and vertical outputs of `spline_closest` all speak the same units whatever the entity's scale is.

## What is not here

There is no way to create, edit or re-point a spline from a script, and no way to read the control points themselves: the curve is queried, never authored, at run time. Splines are built in the editor's spline tool and saved with the scene.

There is also no per-point roll read. Roll exists in the data and shapes the frames a `SplineMeshComponent` extrudes, so it is already baked into `right` and `up` where following the curve needs it.
