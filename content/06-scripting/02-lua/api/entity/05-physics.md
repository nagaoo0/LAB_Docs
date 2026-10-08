---
title: "Physics"
---

Forces, impulses, velocities and sleep on an entity's rigid body, and the collision layer it sits on.

See [Lua Scripting](../../index.md).

Every call is an entity method, called on a handle like `entity:add_force(v)`.

Everything here needs a running physics world, and a world exists only while the scene is playing.
Outside play mode every call on this page is a no-op or a neutral value rather than an error: a
zero vector, `false`, or nothing happening at all.

The push calls all want a **dynamic** body. A static body (a wall, the ground) shrugs them off,
and a kinematic body takes a direct velocity write but no force, the same split the solver makes.
Every force, impulse and velocity call also activates the body first, so one that has gone to sleep
still takes the push without a separate `wake_up()`.

## `add_force(force)`

A continuous force in newtons, accumulated over the next physics step and cleared by it.

That means it has to be applied every frame to keep acting, and that calling it once barely
registers. The force is mass scaled: the same push moves a heavier body less. It is the right
call for thrust, wind, a hover, a conveyor.

```lua
-- Held thrust, applied every frame while the key is down.
function on_update(dt)
    if input.is_key_down("W") then
        entity:add_force(vec3.new(0, 200, 0))
    end
end
```

Apply it from `on_update`, every frame the push should last. A force accumulates into the step the
solver is about to take rather than being dosed by the physics clock, so the same push produces
slightly different results at different frame rates. That is a property of driving a fixed step
from a variable one, not a fault in the call.

A silent no-op on an entity that does not resolve, that has no body in the world, or whose body is
not dynamic.

[C++ equivalent](../../../03-cpp/api/physics/01-physics.md)

## `add_force_at(force, point)`

The same continuous force, applied at a world space point instead of at the centre of mass, so an
off-centre point also spins the body. Pushing the top edge of a crate tips it as well as moving it.

`point` is in world space, not local to the entity.

A silent no-op on an entity that does not resolve, that has no body in the world, or whose body is
not dynamic.

## `add_torque(torque)`

A continuous spin, in world axes, accumulated over the next step and cleared by it like
`add_force`. This is what a fan, a thruster offset from the centre, or a re-entry tumble uses.

A silent no-op on an entity that does not resolve, that has no body in the world, or whose body is
not dynamic.

## `add_impulse(impulse)`

A one-off change of motion, applied immediately rather than accumulated over a step.

It is in newton-seconds, and the velocity change it produces is the impulse divided by the body's
mass, so a heavier body moves less for the same number. Use it for a jump, a hit, a knockback, and
anything else that should compose with the motion the body already has.

```lua
if input.is_key_pressed("Jump") then
    entity:add_impulse(vec3.new(0, 0, 6))
end
```

A silent no-op on an entity that does not resolve, that has no body in the world, or whose body is
not dynamic.

[C++ equivalent](../../../03-cpp/api/physics/01-physics.md)

## `add_impulse_at(impulse, point)`

The same one-off change, applied at a world space point, so an off-centre hit both moves the body
and spins it. A shot into the edge of a plank is this call, not `add_impulse`.

A silent no-op on an entity that does not resolve, that has no body in the world, or whose body is
not dynamic.

## `add_angular_impulse(impulse)`

A one-off change to the body's spin alone, applied immediately, with no linear motion to go with
it. What an explosion at a distance uses when it should turn something rather than shove it.

A silent no-op on an entity that does not resolve, that has no body in the world, or whose body is
not dynamic.

## `get_velocity()`

Linear velocity in metres per second, in world space.

```lua
local max_speed = 8

local v = entity:get_velocity()
local flat = vec3.new(v.x, v.y, 0)
if flat:length() > max_speed then
    local scale = max_speed / flat:length()
    entity:set_velocity(vec3.new(v.x * scale, v.y * scale, v.z))
end
```

Answers `(0, 0, 0)` for an entity that does not resolve, or one with no body in the running world,
which is also every entity outside play mode.

[C++ equivalent](../../../03-cpp/api/physics/01-physics.md)

## `set_velocity(velocity)`

A direct write of the body's velocity rather than a push: it cancels whatever motion the body had,
ignores mass, and activates the body on the way in.

It is the right call for a hard clamp, a respawn or a dash; `add_impulse` is the right one for
anything that should compose with the body's current motion. Note that it does not bound gravity,
so clamping the horizontal speed of a falling body leaves the fall untouched and the total speed
can still be whatever the fall made it.

Refused on a **static** body, which leaves the value alone. A kinematic body takes it.

[C++ equivalent](../../../03-cpp/api/physics/01-physics.md)

## `get_angular_velocity()`

The body's angular velocity, in world axes.

Answers `(0, 0, 0)` for an entity that does not resolve, or one with no body in the running world.

## `set_angular_velocity(velocity)`

A direct write of the body's spin, the same shape as `set_velocity`: it cancels the spin the body
had, ignores mass, and activates the body.

Refused on a **static** body. A kinematic body takes it.

The length is **clamped to 46 rad/s**, under Jolt's own limit of 47.12 rad/s (above which a debug build asserts), keeping the direction; a component that is not a finite number reads as no spin. The native `SetAngularVelocity` goes through the same call.

## `get_acceleration()`

The acceleration the body is about to be given, derived rather than stored: the engine keeps
forces, and acceleration is what force and mass produce during a step. This answers the accumulated
force divided by the body's mass, plus the gravity this body sees.

Answers `(0, 0, 0)` for an entity that does not resolve, one with no body, and one whose body is
not dynamic.

There is no setter on purpose. Setting an acceleration means applying a force of mass times that
acceleration, and `add_force` says that plainly rather than hiding a multiply.

## `is_sleeping()`

Whether the body has gone to sleep, which is the solver's own state: an inactive body is skipped by
the simulation until something rouses it.

```lua
if entity:is_sleeping() then
    log.info("at rest")
end
```

Answers `false` for an entity that does not resolve, one with no body, and every entity outside
play mode.

## `wake_up()`

Activates the body, which is what brings a sleeping one back into the simulation.

A silent no-op for an entity that does not resolve or has no body. The force, impulse and velocity
calls above already activate the body they are given, so this is for rousing one that has nothing
pushing it, before reading or writing something that needs it awake.

## `get_layer()`

The entity's collision layer, as a number. What that layer collides with is the scene's own
collision matrix, not something an entity decides for itself.

The value is read off the rigid body component, so the answer is the same outside play mode.

Answers `0` for an entity that does not resolve or carries no rigid body component.

## `set_layer(layer)`

Sets the collision layer, clamped into the 0 to 15 range the scene's matrix has.

While the scene is playing, the live body's layer is updated in place, so the change applies to the
running simulation rather than at the next play, and the body's own motion type is left alone.

A silent no-op for an entity that does not resolve or carries no rigid body component.

## Where the C++ side stops

The native module ABI's physics surface is smaller than this page: rays, `AddForce`, `AddImpulse`
and linear velocity, documented in [Physics](../../../03-cpp/api/physics/01-physics.md). Torque, angular
velocity, point forces, sleep and the collision layer have no counterpart there yet.
