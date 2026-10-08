---
title: "scene"
---

Finding an entity by name or by id, naming the scene, instantiating an object, destroying an
entity, and baking a navmesh. See [Lua Scripting](../../index.md).

`scene` is one table, bound to the scene that is playing. A script never creates it and never
replaces it. Streaming is on [its own page](02-streaming.md), picking on
[the picking page](03-picking.md), and the shared `globals` table on
[the globals page](04-globals.md).

## `scene.find(name)`

```lua
local door = scene.find("Level Door")
if door then
    door:translate(vec3.new(0, 0, 2))
end
```

The first entity whose name matches, or nil when nothing matches. The name is the entity's tag as
the Hierarchy panel shows it, and names are not unique: this answers the first match it reaches,
so it is for finding a well-known entity, not for identifying one.

A find walks every entity's tag and compares the string, and what it matches on is editable in the
Properties panel. Renaming an entity silently breaks every `scene.find` that reached it, with no
error, because a name that matches nothing is indistinguishable from an entity that has not
spawned yet. Prefer the id wherever the entity is one you hold on to.

The module ABI has the same call as `FindEntityByName`, documented with the rest of the queries in
[Entity queries](../../../03-cpp/api/entity/01-entity-queries.md).

## `scene.find_by_id(id)`

```lua
local saved_id = entity:get_id()

-- Later, possibly after a save, a load or a level switch.
local target = scene.find_by_id(saved_id)
if target then
    target:set_position(vec3.new(0, 0, 5))
end
```

The entity that id names, or nil when the id no longer names a live entity.

An id is the stable handle: minted once when the entity is created, and the value that survives a
save, a load, play mode and undo, none of which a name does. Ids are drawn below 2^53 so Lua holds
one exactly, and zero is never a valid entity, so a lookup that fails and a lookup of an id that
was never set both answer nil. Store the id, look it up again when the entity is needed, and check
the answer.

The module ABI has no lookup to make, because an id is already the handle there; see the same
[Entity queries](../../../03-cpp/api/entity/01-entity-queries.md) page.

## `scene.get_name()`

The name of the scene that is playing: the name written in the scene file's own header, not a
path. A scene with no name answers the empty string, and a switch through `scene.open_scene`
replaces it along with the file.

## `scene.spawn(path)` / `scene.spawn(path, position)`

```lua
local crate = scene.spawn("objects/crate.Lobj")

local placed = scene.spawn("objects/crate.Lobj", vec3.new(3, 4, 5))
```

Instantiates an object (a `.Lobj`, the same format a scene file holds) and answers the new root
entity, or nil when the path does not resolve or the file does not load: a missing file, one that
does not parse, and one that is not an object at all all answer nil rather than raising.

The path is project-relative, the same string the scene file holds, so a spawn survives the
project moving. It resolves against the project's asset folder and then against the packaged
archive, which is what lets the same call work in a build.

With a position, the instantiated root is placed there and its children keep their local offsets,
so the subtree keeps the shape it was authored with. A body-backed root is pushed to the physics
world too, so the first step does not put it back where it was authored.

**The entity is whole by the time the call returns**: its rigid body is in the simulation, its
mesh upload is queued, and any script it carries gets its own `on_create`, exactly as it would if
the object had been in the scene all along. Spawning from `on_update` is ordinary; an instance
started this way begins updating on the next frame.

C++ modules call the same thing as `SpawnEntity`/`SpawnEntityAt`; see
[Hierarchy and spawning](../../../03-cpp/api/entity/03-hierarchy-and-spawning.md).

## `scene.create_entity(name [, parent])`

```lua
local panel = scene.create_entity("Panel")
panel:add_component("UI Transform")

local label = scene.create_entity("Label", panel)
label:add_component("UI Transform")
label:add_component("Text")
label:set_text("Score")
```

Makes an empty entity (a name, a transform and an id) and answers it, or nil when `parent` is
given but is not a live entity. With a parent the new entity is linked under it and keeps its
local transform as it is. Everything else comes from `entity:add_component`, see
[Components](../entity/16-components.md).

The entity is announced like a spawned one, so a body added later is picked up by physics.

**It exists on this machine only.** Nothing replicates an entity made here, so on a networked
game every peer has to make its own, or spawn a prefab with `net.spawn` instead.

## `scene.destroy(entity)`

Destroys the entity and everything under it, and everything each of them owns: GPU mesh buffers
are released, bodies leave the simulation, sounds are cut off, and scripts get their `on_destroy`.
A handle that no longer resolves is a no-op rather than an error.

```lua
local victim = scene.find("Crate")
scene.destroy(victim)

-- false: a reference outlives the entity it named
log.info(tostring(victim:valid()))
```

A destroyed reference is safe to keep and useless to use: `valid()` reports false and calls
through it answer their neutral values rather than raising.

It is safe to destroy the entity a script is running on, from inside that script's own callback.
The engine takes the instance out of its table before running `on_destroy`, and the update loop
walks a snapshot of its instances, which is why a projectile that removes itself on impact is
ordinary rather than a landmine. Children are destroyed with the parent.

Note that this call does not consult the `Persistent` flag: `scene.destroy` takes the subtree it
is given, persistent descendants included. Only `scene.open_scene` spares persistent entities, and
that is a different call on [the streaming page](02-streaming.md).

The module ABI's destroy is `DestroyEntity`; see
[Entity queries](../../../03-cpp/api/entity/01-entity-queries.md).

## `scene.bake_navmesh(volume)`

Bakes the navmesh for a `Navigation Mesh Volume` and loads the result, answering whether it
worked. It makes the same two calls the editor's **Bake Navmesh** button does, which is what
lets a test bake without an editor session, and a game rebuild a navmesh at runtime for
procedural content.

The bake reads the scene's own geometry, writes the result to the volume's `Baked Asset Path`,
and reloads the navmesh, so a path query straight after it has something to work with. The
file it writes is what a build ships and what the scene loads when play starts.

```lua
local volume = scene.find("Nav Mesh")
if not scene.bake_navmesh(volume) then
    log.warn("the navmesh did not bake, see LAB.log for the reason")
end
```

`false` for a handle that does not resolve and for a bake that failed, with the reason in
`LAB.log`. It is synchronous: it returns once the mesh is built and loaded.

See [Navigation & AI](../../../../04-gameplay/03-navigation-and-ai.md) for the workflow, and
[AI](../entity/14-ai.md) for the calls that use the result.
