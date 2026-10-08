---
title: "quat"
---

A rotation held as a quaternion: composing, interpolating and aiming, which Euler degrees cannot
do. See [Lua Scripting](../../index.md).

Euler angles stay the authoring representation. `entity:get_rotation` and `entity:set_rotation`
speak degrees, and a script that only ever sets an angle need never touch this type. Reach for a
quaternion when a rotation has to be combined with another one, blended between two, or aimed
along a direction.

**There is no component-wise constructor.** `quat.new(w, x, y, z)` does not exist: a quaternion
typed by hand is either not normalised or not the rotation the author had in mind, and every
legitimate way to make one has a name. They are `quat.identity()`, `quat.from_euler(degrees)`,
`quat.from_axis_angle(axis, degrees)`, `quat.slerp(a, b, t)`, `quat.between(from, to)` and
`quat.look_rotation(forward [, upHint])`.

Angles on this boundary are degrees, the same as the rest of the Lua API. The engine's C++ side
works in radians underneath.

## `quat.identity()`

```lua
local q = quat.identity()
```

The rotation that changes nothing. Its components are `w = 1`, `x = 0`, `y = 0`, `z = 0`, and it
is what the named constructors answer when their input is degenerate: a zero-length axis to
`from_axis_angle`, a zero vector to `between`, a zero `forward` to `look_rotation`.

## `quat.from_euler(degrees)`

```lua
local facing = quat.from_euler(vec3.new(0, 0, 90))
```

The rotation those Euler degrees describe, in the units and the conversion `entity:get_rotation`
and `entity:set_rotation` themselves deal in. So `quat.from_euler(entity:get_rotation())` is the
rotation the entity's local transform holds, and `entity:set_rotation(q:to_euler())` writes an
equivalent one back.

## `quat.from_axis_angle(axis, degrees)`

A rotation of `degrees` about `axis`, right-handed about that axis. The axis does not have to be
unit length, it is normalised for you, and a zero-length axis answers the identity rotation rather
than a rotation full of NaNs.

## `quat.slerp(a, b, t)`

```lua
local q = quat.slerp(from, to, math.min(1, dt * turn_speed))
```

Shortest-arc interpolation between two rotations: `t = 0` answers `a`, `t = 1` answers `b`, and
everything between turns smoothly rather than sliding along a straight line between the four
components. This is the reason the type is here: interpolating Euler angles between, say, 350 and
10 degrees sweeps 340 degrees the wrong way, and there is no fix for that short of special-casing
every component. Both ends are expected to be unit rotations, which is what every constructor on
this page returns.

## `quat.between(from, to)`

The rotation that turns `from` into `to`, in the plane containing both: the short way round, not a
spin about some third axis. Both vectors are normalised for you, so their lengths do not matter,
and a zero-length `from` or `to` answers the identity rather than a broken rotation.

## `quat.look_rotation(forward [, upHint])`

The rotation that points along `forward`, with up as close to `upHint` as the forward allows:
facing a direction without rolling over, which is what vehicles and chase cameras want. `upHint`
defaults to world `+Z`, and a zero-length `forward` answers the identity.

The axis this points along is the quaternion's own forward, `q:forward()`, which is its **+Y**
axis, and not the entity convention: an entity's local forward is **-Z**. A camera entity, and any
entity whose own -Z should face the direction, composes the result with
`quat.from_euler(vec3(90, 0, 0))`:

```lua
-- An unparented camera entity looking at a target point.
local aim = quat.look_rotation(target - entity:get_world_position())
entity:set_rotation_quat(aim * quat.from_euler(vec3(90, 0, 0)))
```

## `quat.angle(a, b)`

The angle between two rotations, in degrees, from 0 to 180: the "am I aimed at it yet" test. Both
sides are normalised first, and because a rotation and its own negation describe one orientation,
the answer is the same whichever sign either side is carrying.

```lua
local remaining = quat.angle(entity:get_rotation_quat(), target_rotation)
if remaining < 5 then
    log.info("lined up")
end
```

## `q.w`, `q.x`, `q.y`, `q.z`

The four components, in the order `tostring(q)` prints them: `quat(w, x, y, z)`. `w` is the scalar
part, so `quat.identity()` reads `w = 1` with the rest zero.

They are ordinary fields: reading them is how a rotation is inspected, and an assignment is
accepted. A hand-edited component leaves four numbers that are probably not a unit rotation, which
is exactly what the missing constructor exists to prevent. Build rotations with the named
constructors, and use `q:normalized()` if one has drifted.

## `q:to_euler()`

The rotation as Euler degrees, the shape `entity:get_rotation` answers and `entity:set_rotation`
takes, so a quaternion can be handed back to the parts of the API that speak degrees.

It is the inverse of `quat.from_euler`. That pair is worth preferring over the lossy routes:
reading a rotation as a quaternion and writing one back is exact, going through degrees is not,
and composing two rotations through degrees is worse than lossy. Use this for authoring and
display, and `q * q2` for combining.

## `a * b` and `q * v`

```lua
local combined = a * b
local rotated = q * vec3.new(0, 0, 1)
```

Composition, and rotating a vector. The order matters and is the usual one: `a * b` applies `b`
first, then `a`. `q * v` answers the vec3 the rotation moves `v` to, which is the compact way to
turn an offset or a direction.

There is no `quat + quat`, no scaling and no equality operator, the same as `vec3`: two separately
built rotations describing one orientation are different values under `==`, so `quat.angle(a, b)`
is the comparison to use instead.

## `q:inverse()`

The rotation that undoes this one: for the unit rotations every constructor here returns, it is
the conjugate, the same rotation with the vector components negated. Rotating by `q` and then by
`q:inverse()` gives the original vector back.

## `q:normalized()`

The same rotation as a unit quaternion. Rotations built by multiplying one into another drift off
unit length after enough of them, and the unit assumption is what `q:forward()`, rotating a vector
and `quat.slerp` all rest on.

## `a:dot(b)`

The raw four-component dot product, with neither side normalised. For unit rotations it is the
cosine of half the angle between them, so `1` is the same rotation, `-1` is the same rotation
written the other way round (a rotation and its negation are one orientation), and `0` is 180
degrees apart. The sign is not meaningful on its own here: `quat.angle` is the call for how far
apart two rotations are.

## `q:forward()`

The direction this rotation maps its own forward axis to. That axis is **+Y**, which is not the
convention an entity's forward uses: an entity's local forward is **-Z**, so `q:forward()` and
`entity:get_forward()` answer different vectors for the same orientation. Use
`entity:get_forward()` for where an entity is pointing, and this one when the rotation itself is
what is being reasoned about.

## `q:right()`

The rotation's **+X** axis, mapped by the rotation. For an unparented entity holding this
rotation, that is the same direction `entity:get_right()` reports.

## `q:up()`

The rotation's **+Z** axis, mapped by the rotation. World up is **+Z**, so this is world up for an
unrotated rotation, and it is not an entity's own up axis: an entity's local up is **+Y**
(`entity:get_up()`).

The native module ABI has no quaternion type at all, so a module works in Euler degrees and
vectors (see [Transform](../../../03-cpp/api/entity/02-transform.md)).
