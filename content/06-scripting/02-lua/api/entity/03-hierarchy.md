---
title: "Hierarchy"
---

An entity's parent and children, and how reparenting keeps world placement.

See [Lua Scripting](../../index.md).

Every call is an entity method, called on a handle like `entity:get_parent()`.

The parent link is always changed through the scene's own reparent routine, which rejects cycles
and rebases the child's local transform so the child keeps its world placement.

## `get_parent()`

The parent entity, or `nil` when it has none.

`nil` for an entity that does not resolve as well.

```lua
local root = entity
while true do
    local parent = root:get_parent()
    if not parent then
        break
    end
    root = parent
end
```

[C++ equivalent](../../../03-cpp/api/entity/03-hierarchy-and-spawning.md)

## `set_parent(parent)`

Makes `parent` the parent of this entity, and answers whether it happened: `true` on success,
`false` when the entity does not resolve, when the parent argument does not resolve, when the
parent is the entity itself, or when the parent is already somewhere below the entity in the tree,
which would build a cycle.

Called with no argument, or with `nil`, it detaches the entity to the scene root. A **stale**
parent handle is treated exactly like `nil`: it detaches rather than failing, because a handle
that no longer resolves counts as no parent at all.

```lua
local hat = scene.spawn("objects/hat.Lobj")
if hat then
    hat:set_parent(entity)
end
```

**World placement is preserved, on the way in and on the way out.** The child's local values
change so that its world position, rotation and scale stay where they were, the same thing the
editor does when you drag one entity onto another. The consequence worth knowing: after this call
the child's local values are not the ones authored in the object file, they are whatever produces
the same world placement under the new parent.

### The one case that cannot be exact

A parent that is both rotated and non-uniformly scaled cannot be represented exactly. Keeping the
child's world shape would need a shear term this transform does not have, so the closest
rotate+scale fit is used and the child's scale can shift slightly. World position is never
affected. Reparenting under a uniformly scaled parent, however large, is exact.

[C++ equivalent](../../../03-cpp/api/entity/03-hierarchy-and-spawning.md)

## `get_children()`

A new table on every call: a plain 1-based array of the entity's children, in the order they were
attached.

The list is copied out of the scene's storage before the table is built, so destroying or
reparenting an entity while you walk its siblings cannot pull the list out from under you. The
entities themselves can still go away mid-iteration, so check `valid()` if that can happen.

```lua
local children = entity:get_children()
for i = 1, #children do
    log.info(children[i]:get_name())
end
```

An empty table for an entity that does not resolve, or one with no children.

[C++ equivalent](../../../03-cpp/api/entity/03-hierarchy-and-spawning.md)

## `get_child_count()`

How many children the entity has, without building the table.

```lua
if entity:get_child_count() == 0 then
    log.info("nothing hanging off me")
end
```

Answers `0` for an entity that does not resolve, or one with no children.

[C++ equivalent](../../../03-cpp/api/entity/03-hierarchy-and-spawning.md)
