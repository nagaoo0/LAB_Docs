---
title: "Identity"
---

Whether an entity handle still points at anything, what the entity is called, and which components it carries.

See [Lua Scripting](../../index.md).

Every call on this page is an entity method, called on a handle the way `entity:get_name()`
shows. Each one answers a neutral value rather than raising when the entity does not resolve:
`false`, `0`, `""` or an empty table. That is the tolerance the whole `entity` object
has, and it is deliberate. A script generally does not know what it is attached to, and an error
every frame would be worse than nothing happening.

## `valid()`

False once the entity has been destroyed, and false for a handle that never named one.

An entity reference in Lua is a copy of the scene pointer and the registry handle, so it outlives
the entity it names rather than becoming unusable. This call is how a script notices, and it is
the one place on this page where a neutral answer is worth acting on.

```lua
function on_update(dt)
    if m_Target and not m_Target:valid() then
        m_Target = nil
        log.info("the target is gone")
    end
end
```

Answers `false` for a handle that carries no scene at all.

[C++ equivalent](../../../03-cpp/api/entity/01-entity-queries.md)

## `get_id()`

The stable numeric id, the same value a scene file stores and `scene.find_by_id` looks up.

An id survives save, load, play mode and undo, which a name does not. This is the handle worth
writing down: keep it in a variable or on the shared `globals` table, and find the same entity
again later with it.

```lua
local door = entity:get_id()
-- later, after anything at all:
local again = scene.find_by_id(door)
if again then
    again:set_position(vec3.new(0, 0, 2))
end
```

Answers `0` for an entity that does not resolve, and `0` is never a valid id, so it doubles as
"nothing".

[C++ equivalent](../../../03-cpp/api/entity/01-entity-queries.md)

## `get_name()`

The entity's tag, the string shown in the Hierarchy panel.

Names are not unique, and `scene.find` answers the first entity that matches, so a name finds a
well known entity rather than identifying one.

Answers `""` for an entity that does not resolve or carries no tag component.

[C++ equivalent](../../../03-cpp/api/entity/01-entity-queries.md)

## `set_name(name)`

Writes the entity's tag, creating the tag component if the entity somehow has none.

A no-op for an entity that does not resolve. Renaming an entity silently breaks every
`scene.find` that reached it, because a name matching nothing is indistinguishable from an entity
that has not spawned yet. That is why the id is the handle to keep.

[C++ equivalent](../../../03-cpp/api/entity/01-entity-queries.md)

## `is_persistent()`

Whether this entity survives a `scene.open_scene()` call, which destroys every entity without the
flag. This is what a game manager, a player controller or a HUD script sets before a level change.

Answers `false` for an entity that does not resolve, and for one that carries no persistent flag.

[C++ equivalent](../../../03-cpp/api/streaming/01-scene-streaming.md)

## `set_persistent(value)`

Sets or clears the flag. Called with no argument at all it **sets** it, so `set_persistent()` means
"keep this entity", and `set_persistent(false)` undoes that.

```lua
function on_create()
    entity:set_persistent()
end
```

A no-op for an entity that does not resolve. There is no effect on `scene.append_scene`, which
never destroys anything to begin with. An entity kept this way whose parent is about to be
destroyed is detached to the root rather than taken down with it, keeping its world placement.

[C++ equivalent](../../../03-cpp/api/streaming/01-scene-streaming.md)

## `native_fields()`

A table of the values a C++ module's behaviour members hold right now, keyed by behaviour name and
then by field name. Every value arrives as a **string**, because that is the shape the module's
memory is read back in: `"1.5"`, `"3"`, `"true"`, and `"0,1,2"` for a vector.

```lua
local fields = entity:native_fields()
if fields.Turret then
    log.info("turret heat " .. fields.Turret.Heat)
end
```

This reads the running instance rather than the authored component, which is what tells an applied
value from a stored one. A field the scene has a value for but the module never declared does not
appear at all, and a declared field nobody authored appears with the constructor's default.

An empty table for an entity that does not resolve, an entity with no native behaviours, and a
scene with no native module loaded.

[C++ equivalent](../../../03-cpp/api/module/04-declared-fields.md)

## `has_body()`

Whether the entity's body is in the running physics world.

The answer is read off the physics world rather than off a rigid body component, so an entity that
carries the component but whose body has not entered the simulation yet answers `false`. In
practice that is false before play starts, and true afterwards for a body of any motion type, a
static or kinematic one included.

Answers `false` for an entity that does not resolve.

```lua
if entity:has_body() then
    entity:add_impulse(vec3.new(0, 0, 5))
end
```

[C++ equivalent](../../../03-cpp/api/physics/01-physics.md)

## `has_material()`

Whether the entity carries a material component, which everything on the [material](04-material.md)
page needs.

Answers `false` for an entity that does not resolve.

## `has_ragdoll()`

Whether the entity has a **built** ragdoll in the running physics world.

Same shape as `has_body`: the answer comes from the physics world, so an entity whose ragdoll has
not been built yet answers `false`, and so does every entity before play starts. The build happens
lazily, the first time the scene sees the entity with a ragdoll component and a live physics
world.

Answers `false` for an entity that does not resolve.

## `has_mesh()`

Whether the entity carries a mesh component. The answer is read off the component, so it is the
same outside play mode.

Answers `false` for an entity that does not resolve.

## `has_camera()`

Whether the entity carries a camera component.

A script looking for the camera among an entity's children should not have to find it by name,
because renaming it in the editor would silently break the look.

Answers `false` for an entity that does not resolve.

[C++ equivalent](../../../03-cpp/api/entity/04-components.md)

## `has_light()`

Whether the entity carries any of the three light components: a point, a directional or a spot
light.

Answers `false` for an entity that does not resolve.

[C++ equivalent](../../../03-cpp/api/entity/04-components.md)
