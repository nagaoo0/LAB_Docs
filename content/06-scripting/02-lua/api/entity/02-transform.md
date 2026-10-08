---
title: "Transform"
---

Where an entity is, which way it faces, how big it is, and how a write reaches the physics body.

See [Lua Scripting](../../index.md).

Every call is an entity method, called on a handle like `entity:get_position()`.

Rotations here are **degrees**. The transform component stores radians, and every call on this
page converts on the way in and out.

LAB is Z-up. An entity's local forward is **-Z**, its right is **+X**, and its up is **+Y**, the
convention the renderer's camera already uses. `get_forward`, `get_right` and `get_up` read that
basis out of the **world** matrix, so a parented entity reports where it actually points rather
than its local rotation.

## How a transform write reaches the rest of the engine

`set_position`, `translate`, `rotate`, `set_rotation` and `set_rotation_quat` write the component
and then notify the scene, and the notification is what makes the write stick.

- While the scene is playing, the entity's world transform is pushed onto its physics body, so a
  scripted move is not overwritten by the next solver step.
- Only the half of the pose that actually changed is honoured. The scene's transform is the
  interpolated pose, which trails the simulated one by up to a step, so pushing it back wholesale
  would drag the body backwards every frame and bleed its motion away. A position write leaves the
  body's own rotation to the simulation, and a rotation write leaves its position.
- A character-controlled entity is not teleported by a per-frame write. Its own collision sweep
  resolves each step, which is what driving a controller wants, so a character is moved with
  `move` and `set_character_velocity` rather than by writing its transform.
- The renderer's temporal history is not reset, because a value written every frame is exactly the
  case that pass already reprojects correctly.

`set_scale` is the exception: it writes the component and nothing else.

## `get_position()`

The entity's **local** position: relative to its parent when it has one, and to the world when it
does not.

```lua
local here = entity:get_position()
entity:set_position(here + vec3.new(0, 0, 1))
```

Answers `(0, 0, 0)` for an entity that does not resolve or carries no transform component.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `set_position(v)`

Writes the local position onto the entity, then notifies the scene, so a rigid body follows the
new value instead of discarding it. The mechanics are in
[How a transform write reaches the rest of the engine](#how-a-transform-write-reaches-the-rest-of-the-engine).

```lua
local origin = 0
local elapsed = 0

function on_create()
    origin = entity:get_position()
end

function on_update(dt)
    elapsed = elapsed + dt
    entity:set_position(origin + vec3.new(0, 0, math.sin(elapsed * 2) * 0.5))
end
```

Note that the snippet remembers `origin` rather than reading the position back every frame and
adding to it. Reading it back accumulates error, and it fights anything else that moves the same
entity.

A no-op for an entity that does not resolve or carries no transform component.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_world_position()`

The entity's position in world space, with every ancestor's transform applied.

```lua
local here = entity:get_world_position()
```

Answers `(0, 0, 0)` for an entity that does not resolve. There is no world position **setter**,
because the local value is the one that is stored: to place an entity at a world point, spawn it
there or account for its parent's transform yourself.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `translate(delta)`

Adds a delta to the entity's local position, then notifies the scene like `set_position` does. It
is the convenient form for a movement script, because the delta can come straight from an input
axis scaled by the frame time.

```lua
local speed = 5

function on_update(dt)
    local x = input.get_key_axis("A", "D")
    local y = input.get_key_axis("S", "W")
    entity:translate(vec3.new(x, y, 0) * speed * dt)
end
```

A no-op for an entity that does not resolve or carries no transform component.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_rotation()`

The entity's **local** rotation as Euler angles, in degrees, X then Y then Z as the engine
composes them: a child's rotation is relative to its parent's.

```lua
local yaw = entity:get_rotation().z
```

Answers `(0, 0, 0)` for an entity that does not resolve or carries no transform component.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `set_rotation(v)`

Writes the local rotation as Euler angles in degrees, then notifies the scene like `set_position`
does, so a physics body is turned to match.

A no-op for an entity that does not resolve or carries no transform component.

```lua
entity:set_rotation(vec3.new(0, 0, 90))
```

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `rotate(degrees)`

Adds the delta to the entity's current Euler angles, component by component, in degrees, then
notifies the scene.

```lua
local turn = 120

function on_update(dt)
    local yaw = input.get_key_axis("A", "D") * turn * dt
    entity:rotate(vec3.new(0, 0, -yaw))
end
```

Because the add is per component, a delta on one axis is exact, and a delta on two or three axes
at once is not the same thing as composing two rotations. Reach for the `quat` functions when a
rotation has to be composed rather than nudged.

A no-op for an entity that does not resolve or carries no transform component.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_rotation_quat()`

The same rotation as `get_rotation`, without the Euler round trip, as a `quat`.

Reading it as a quaternion and writing one back is lossless; going through degrees is not, and
composing two rotations through degrees is worse than lossy.

```lua
local q = entity:get_rotation_quat()
entity:set_rotation_quat(quat.slerp(q, quat.identity(), 0.1))
```

Answers the identity quaternion `(1, 0, 0, 0)` for an entity that does not resolve or carries no
transform component.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `set_rotation_quat(q)`

Normalises `q`, converts it to Euler angles, and writes those to the component, then notifies the
scene like `set_position` does. The component always holds the Euler form, so `get_rotation`
answers the same rotation expressed in degrees.

A no-op for an entity that does not resolve or carries no transform component.

```lua
entity:set_rotation_quat(quat.from_axis_angle(vec3.new(0, 0, 1), 45))
```

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_world_rotation()`

The entity's rotation in world space, in degrees, decomposed from the world matrix on every call.

Answers `(0, 0, 0)` for an entity that does not resolve, or one whose world matrix will not
decompose. Under an ancestor that is both rotated and non-uniformly scaled it is the closest
rotate+scale fit rather than an exact answer, because the transform has no shear term.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_scale()`

The entity's **local** scale.

Answers `(1, 1, 1)` for an entity that does not resolve or carries no transform component, rather
than zero, because one is the identity and zero is a degenerate object.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `set_scale(v)`

Writes the local scale. Unlike the other setters on this page it does not notify the scene, and
**the collider does not follow**: Jolt bakes the scale into the shape when the body is created, so
the mesh resizes and the collision shape stays where it was. An entity that needs a collider at a
different size has to be built that way.

A no-op for an entity that does not resolve or carries no transform component.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_world_scale()`

The entity's scale in world space, decomposed fresh from the world matrix on every call. There is
no cached world transform to keep in sync, so this is the value of the moment.

Under an ancestor that is both rotated and non-uniformly scaled this is the closest rotate+scale
fit to the true, sheared world shape, the same approximation the gizmo and the Properties panel's
World Scale field make.

Answers `(1, 1, 1)` for an entity that does not resolve, or one whose world matrix will not
decompose.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_forward()`

Which way the entity points, in world space: its world **-Z** basis vector.

```lua
local speed = 5

function on_update(dt)
    entity:translate(entity:get_forward() * speed * dt)
end
```

Answers `(0, 0, -1)` for an entity that does not resolve. The vector is normalised, so a scale of
zero leaves it nothing to normalise, and the same applies to `get_right` and `get_up`.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_right()`

The entity's world **+X** basis vector.

Answers `(1, 0, 0)` for an entity that does not resolve.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_up()`

The entity's world **+Y** basis vector.

This is the **entity's** up, not the world's: world up is `+Z`, because gravity is `-Z`, and an
unrotated entity's local `+Y` lies along world `+Y`.

Answers `(0, 1, 0)` for an entity that does not resolve.

[C++ equivalent](../../../03-cpp/api/entity/02-transform.md)

## `get_fov()`

A camera's vertical field of view in degrees. Speed effects widen it, and this is how one reads
what it currently is.

```lua
local fov = entity:get_fov()
-- ease toward a wider value while sprinting
entity:set_fov(fov + (85 - fov) * 0.1)
```

Answers `0.0` for an entity that does not resolve or carries no camera component.

[C++ equivalent](../../../03-cpp/api/entity/04-components.md)

## `set_fov(degrees)`

Writes the camera's vertical field of view. The value is clamped into the 1 to 170 degree range,
so a nonsense number is bounded rather than stored.

A no-op for an entity that does not resolve or carries no camera component.

```lua
function on_update(dt)
    if input.is_key_down("LeftShift") then
        entity:set_fov(entity:get_fov() + 60 * dt)
    end
end
```

[C++ equivalent](../../../03-cpp/api/entity/04-components.md)
