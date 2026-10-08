---
title: "Physics"
---

Rays, shape casts, overlaps and the world's gravity settings. See
[Lua Scripting](../../index.md).

`physics` is one of the globals a script is handed, alongside `entity` and `scene`, so there is
nothing to require. Everything that asks the world a question (the ray, the three casts, the
three overlaps) needs a physics world, which exists only while the scene is playing: outside play
mode a ray answers a miss and an overlap comes back empty rather than failing. The gravity and
terminal velocity calls are scene settings rather than world state, so no world is needed to read
or write them, and a change reaches the simulation at its next step.

## `raycast(origin, direction, maxDistance [, ignore])`

Casts a ray and answers the first thing it hits, as one table.

```lua
local h = physics.raycast(entity:get_world_position(), vec3.new(0, 0, -1), 100, entity)
if h.hit then
    log.info("hit " .. h.entity:get_name() .. " at " .. h.distance)
end
```

| Field | |
|---|---|
| `hit` | `true` when something was hit. Every other field is absent when it is `false` |
| `entity` | The thing that was hit, an entity like any other with all the usual methods |
| `position` | The world-space point of impact |
| `normal` | The surface normal at that point |
| `distance` | How far along the ray it was, in world units |

A miss is `{ hit = false }` and nothing else, so `if h.hit then` is the whole idiom and there is
nothing meaningless to read on a miss. A miss is also the answer outside play mode, where there
is no world to hit anything in.

**The direction does not have to be normalised.** It is divided by its own length and the ray is
scaled by `maxDistance`, so `vec3.new(0, 0, -1)` and `vec3.new(0, 0, -9.81)` cast the same ray
with the same reach. A zero-length direction, or a `maxDistance` of zero or less, answers a miss.

`ignore` is the optional entity to leave out of the cast, and it is almost always the caster. A
ray fired from an entity's own centre hits its own collider at distance zero, and answers that
rather than the ground below. The ignored body is filtered out **during** the cast rather than
afterwards, so everything behind it is still found.

[C++ equivalent](../../../03-cpp/api/physics/01-physics.md)

## `sphere_cast(radius, origin, direction, maxDistance [, ignore])`

Sweeps a sphere along a ray and answers the first thing its surface would touch, in the same
table shape `raycast` returns.

A ray is a shape of zero width: it slips through gaps nothing could fit through and misses
ledges a foot would land on. A cast answers the question a ray cannot, whether a body of some
size fits and what it would run into on the way.

The shape stops when its **surface** touches something rather than its centre, so a sphere cast
always reports a shorter distance than a ray fired the same way, by exactly its radius.

**A shape that is already overlapping something reports a hit at distance 0**, rather than
answering clear. The alternative would be to answer "nothing in the way" for a character stuck
inside a wall, which is the one moment the caller most needs to be told otherwise.

```lua
-- Is there room for a body this size ahead of us?
local sweep = physics.sphere_cast(0.5, entity:get_world_position(), entity:get_forward(), 3, entity)
if sweep.hit then
    log.info("something in the way at " .. sweep.distance)
end
```

`ignore` works exactly as it does for `raycast`: pass the caster, or the sweep starts inside it.

## `capsule_cast(radius, height, origin, direction, maxDistance [, ignore])`

The same sweep with a capsule, which is the shape a character is. `height` is the whole capsule
including its caps, and it stands up along the world Z axis, the same way a body's does: a query
capsule lying on its side would report clearances no character using the matching body could
pass through. A capsule whose `height` is not more than twice its `radius` has no cylinder left
in the middle, and is swept as a sphere.

The result table, the miss and the ignore argument are the ones `sphere_cast` above describes.

## `box_cast(halfExtent, origin, direction, maxDistance [, ignore])`

The same sweep with a box. `halfExtent` is a `vec3` half-size, so a two-metre cube is
`vec3.new(1, 1, 1)`. A half-extent of zero or less is floored at a small positive value rather
than refused.

The result table, the miss and the ignore argument are the ones `sphere_cast` above describes.

## `overlap_sphere(radius, position [, ignore])`

Asks what is inside a sphere sitting still at a world position. An overlap has no direction and
no distance: nothing is swept, so the answer is not one thing but everything the shape is
touching.

The answer is a plain array of entities, so `#found` and `ipairs` both work on it:

```lua
local found = physics.overlap_sphere(2, entity:get_world_position())
for i = 1, #found do
    log.info(found[i]:get_name())
end
```

The array is empty (not `nil`) when nothing is inside, and also when there is no world to ask.
It holds one entry per body rather than per contact, so a compound or mesh collider reports the
thing it belongs to once, and the order is whatever the world happened to hand back, so do not
read anything into it. `ignore` leaves one entity out of the answer, and a shape placed inside
the caster finds the caster without it.

## `overlap_capsule(radius, height, position [, ignore])`

The same question with a capsule: `radius` and a total `height` including the caps, standing up
along the world Z axis, and swept as a sphere when `height` is not more than twice the `radius`.
It is the overlap a character uses to ask what it is standing in.

## `overlap_box(halfExtent, position [, ignore])`

The same question with a box, `halfExtent` being a `vec3` half-size. It is the overlap for
anything box-shaped: a doorway, a spawn volume, the footprint a building would occupy.

## `get_gravity()`

The gravity the world is actually applying: the scene's gravity vector multiplied by its gravity
scale. Z is up in LAB, so the default is `vec3(0, 0, -9.81)`.

Note that this answers the **effective** vector rather than the one `set_gravity` last wrote: with
a gravity scale of 2 it answers `vec3(0, 0, -19.62)`.

## `set_gravity(gravity)`

Sets the scene's gravity, in metres per second squared, before the scale is applied. The next
physics step picks it up, so a script can change it mid-fall rather than at the next play. A
moon-ish world is one line:

```lua
physics.set_gravity(vec3.new(0, 0, -3.7))
```

## `get_gravity_scale()`

The multiplier on gravity, `1` by default. Kept separate from the vector so a scene can be made
floatier without retyping a direction.

## `set_gravity_scale(scale)`

Ramps the whole world up or down. `0` is weightless and a value above 1 is heavy, and the change
applies at the next physics step. Prefer this over rewriting the gravity vector when a script
wants to vary the fall between two places: the direction stays authored.

## `get_terminal_velocity()`

The hard cap on linear speed, in world units per second, as `set_terminal_velocity` last set it.
`0` means no cap, which is the default.

## `set_terminal_velocity(speed)`

Caps every dynamic body's speed after each physics substep, so a fast body cannot outrun the cap
during a catch-up burst. `0` disables it.

The cap is applied to the whole velocity vector, so a body at the cap keeps falling, just no
faster. Real terminal velocity comes from drag, which a rigid body's own linear damping models;
this is the blunt instrument, for when a fast body must not tunnel through the level.

```lua
-- A plummet the level geometry can keep up with.
physics.set_terminal_velocity(40)
```

## What is not here

Forces, impulses, torque and velocity are per-entity calls rather than `physics` table calls, and
they are documented with the rest of the entity's own methods: see [`entity/`](../entity/). This
table is the world's settings and the queries you fire at it.
