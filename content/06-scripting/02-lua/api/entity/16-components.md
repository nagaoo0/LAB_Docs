---
title: "Components by name"
---

`add_component`, `has_component` and `remove_component` work on any entity, one component at a
time, by the name the editor's Add Component menu shows: `"UI Transform"`, `"Sprite"`, `"Text"`,
`"Rigid Body"`, `"Collider"`, `"Point Light"` and so on. The names are matched exactly, case
included. The same table drives the menu, so a component the menu offers can be added here.

## `entity:add_component(name)`

```lua
local e = scene.create_entity("Lamp")
e:add_component("Point Light")
```

Adds the component with its defaults and answers true. An unknown name, or a dead entity,
answers false. Adding one the entity already has resets it to the defaults.

Adding a `"Rigid Body"` or `"Collider"` to an entity that now has both gives it a body in the
running simulation.

Component data is set through the usual entity methods afterwards (`set_text`, `set_position`
and the rest), or through the editor before the game runs.

## `entity:has_component(name)`

True when the entity has it. False for an unknown name.

## `entity:remove_component(name)`

True when it was there and is now gone. False for an unknown name or a component the entity
did not have.

Tag, Transform and ID are not in the table, so they can't be removed. To take a physics body
away, destroy the entity rather than removing its components.
