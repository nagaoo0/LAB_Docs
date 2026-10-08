---
title: "Character controller"
---

Driving an entity that carries a Character Controller: its velocity, its jump, and the state it
is standing in.

see [Lua Scripting](../../index.md).

A character controller is **not** a rigid body. It is Jolt's `CharacterVirtual`: a capsule swept
through the world under direct control, so it walks up Step Height kerbs, holds its footing on
slopes and never tips over, and nothing pushes it back. It replaces the Rigid Body/Collider pair
rather than joining it; [Components → Character Controller](../../../../02-building-worlds/02-components.md#character-controller)
has every field, and [Physics → Character controllers](../../../../04-gameplay/01-physics.md#character-controllers)
has the behaviour.

Two things about that shape run through every call below:

- **It is driven by velocity, not by force.** A script sets a velocity and the controller walks
  it, resolving contacts on the way. There is nothing to push and nothing to wake up.
- **The vertical component is the engine's.** Gravity is applied by the physics step and `jump`
  sets the upward speed, which is why `move` refuses to touch z. Vertical motion is why a
  character falls.

Most of these calls need a live physics world, and there is one only while the scene is playing.
`move`, `jump`, `get_character_velocity` and `set_character_velocity` are silent no-ops (or
answer zero) outside Play mode. `is_grounded` and `get_ground_normal` are the exception: they
read the component the physics step writes, so they answer in the editor as well, reporting the
last thing the character stood on.

## `has_character()`

Whether this entity carries a Character Controller. `false` for an entity with no component, and
`false` once the reference no longer resolves, so it is the test to make before anything else
here.

```lua
if not entity:has_character() then
    log.warn("this script drives a Character Controller")
end
```

## `move(velocity)`

Sets the character's horizontal velocity, in metres per second. The x and y components are
written; the z component keeps whatever gravity and `jump` last left it as.

> **Only x and y are taken, deliberately.** A move call that overwrote the vertical speed would
> cancel gravity, and the character would be unable to fall. Use `set_character_velocity` when
> the vertical component is yours to set.

Silent no-op on an entity with no character, and silent outside Play mode, where there is no
physics world to move in. It answers nothing either way.

```lua
-- On the frame clock: read input, drive, and let gravity own the rest.
local direction = vec3.new(input.get_key_axis("A", "D"), input.get_key_axis("S", "W"), 0)
entity:move(direction * speed)
```

## `jump(speed)`

Jumps if the character is standing on something, and answers whether it actually did. `speed` is
the upward velocity in metres per second, not a force and not an impulse: it is written straight
into the z component of the character's velocity, and the horizontal part is preserved.

Answers `false` in mid-air, which is the difference between a jump and a flight, and `false` on
an entity with no character or outside Play mode. Nothing is raised, so a jump that silently did
not happen is exactly what this answer is for.

There is no need to check `is_grounded` first. The refusal is here.

```lua
if input.is_key_pressed("Space") then
    entity:jump(6.0)
end
```

## `is_grounded()`

Whether the character is standing on something, read off the component the physics step writes.
`false` on an entity with no character.

Because it reads written state rather than asking the world, it also works outside Play mode,
reporting the last thing the character stood on; before the first step it is `false`.

```lua
if entity:is_grounded() then
    -- A landing sound, a coyote timer, a footstep: whatever the game wants.
end
```

## `get_ground_normal()`

The normal of the surface the character is standing on, as a world-space vector. It is what a
"can I walk up this" check reads, and what an alignment effect or a decal would use.

Answers `(0, 0, 1)` on an entity with no character, which is also the component's own default
before the first step.

## `get_character_velocity()`

The character's current velocity, in metres per second, world space, all three components. This
is the reader for the vertical component that `move` leaves alone, so it is how a script finds
out how fast something is falling.

Answers a zero vector on an entity with no character or outside Play mode.

## `set_character_velocity(v)`

Writes the whole velocity, vertical included. Unlike `move`, nothing is preserved, so this can
cancel a fall or start one: a dash, a launch, a respawn that must not keep the old momentum, or
a hard stop.

Silent no-op on an entity with no character or outside Play mode.

```lua
-- A respawn: no leftover falling speed.
player:set_character_velocity(vec3.new(0, 0, 0))
```

## A complete example

```lua
local speed = 6
local jumpSpeed = 6

function on_update(dt)
    -- Walk along the entity's own facing, with opposed keys cancelling.
    local wish = entity:get_forward() * input.get_key_axis("S", "W")
        + entity:get_right() * input.get_key_axis("A", "D")
    if wish:length() > 0 then
        entity:move(wish:normalized() * speed)
    else
        entity:move(vec3.new(0, 0, 0))   -- still a velocity: setting zero is how it stops
    end

    -- Jump, refused in mid-air by the call itself.
    if input.is_key_pressed("Space") then
        entity:jump(jumpSpeed)
    end
end
```

`LAB/assets/scripts/player_controller.Lscript` is a fuller version of this, with look and
ground handling already worked out.
