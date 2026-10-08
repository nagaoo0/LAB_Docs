---
title: "Particles"
---

Emitting a burst from an entity's emitter, and asking how many particles it is holding. See [Lua Scripting](../../index.md) for how a script runs and when its callbacks fire.

A `ParticleEmitterComponent` is authored in the Properties panel: a rate or a burst count, a lifetime range, an initial velocity and its spread, gravity, drag, a colour over life and a size over life. These two calls are the whole scripting surface of it, the same surface a native module gets.

[C++ equivalent](../../../03-cpp/api/animation/01-animation-and-particles.md)

## `emit_particles(count)`

```lua
entity:emit_particles(24)
```

Spawns `count` particles from this entity's `ParticleEmitterComponent` right now, at the entity's world position, ignoring `EmissionRate` and `BurstCount`. A hit reaction or an event-driven puff is not something a continuous or burst emitter can express on its own, and this is the call that adds it.

Each new particle gets the emitter's `InitialVelocity` plus up to its `VelocitySpread` on each axis, a lifetime between `LifetimeMin` and `LifetimeMax`, and the emitter's colour and size ramps. Emission is skipped once the emitter holds `MaxParticles` (256 by default), so a burst into a full emitter adds less than was asked, possibly nothing.

A no-op with no `ParticleEmitterComponent`, and a count of zero or less does nothing. That is the same convention `play_sound` follows with no source.

Two things worth knowing:

- **A paused emitter still emits.** `Enabled` off stops the emitter's own rate; an explicit `emit_particles` is a request rather than the emitter's cadence, and it is honoured.
- **An emitter set to GPU simulation has no CPU-side particle list.** Its particles live in a GPU buffer that only the emitter's own rate and burst feed, so a burst here is not visible and `get_particle_count` reads `0` for it.

Emit from `on_update` or a contact callback. The scene steps particles once per frame in its own update, before it draws, so a particle emitted this frame is drawn this frame. A scene with no rigid bodies takes no physics steps at all, so the physics clock is the wrong clock for this either way.

An emitter with a rate of zero and no burst count is silent on its own; an emitter like that, poked from a script, is a purely event-driven effect, which is the usual shape for impacts, dust and celebrations.

## `get_particle_count()`

```lua
local alive = entity:get_particle_count()
```

How many particles this emitter is holding right now. It rises with a burst and falls as particles expire, so it is the honest way to ask "is this emitter still busy?" without assuming anything about the emitter's lifetime settings.

Answers `0` for an entity with no `ParticleEmitterComponent`, and for a GPU-simulation emitter, which keeps no CPU-side list.

```lua
local emitter = scene.find("Burn")
local fuse = 3
local draining = false

function on_update(dt)
    if not emitter then
        return
    end

    if not draining then
        fuse = fuse - dt
        if fuse <= 0 then
            emitter:emit_particles(40)
            draining = true
        end
    elseif emitter:get_particle_count() == 0 then
        draining = false
        log.info("smoke cleared")
    end
end
```

Particles are the one effect here with no per-particle access: there is no way to read or move a single particle, and no way to set the emitter's own fields. "Emit a burst" and "how many are alive" are a script's two tools, exactly as they are a module's.
