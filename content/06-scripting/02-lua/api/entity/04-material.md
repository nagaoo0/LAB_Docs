---
title: "Material"
---

The colour, alpha, emissive and shader graph settings an entity's material carries.

See [Lua Scripting](../../index.md).

Every call is an entity method, called on a handle like `entity:get_color()`.

Colour is a `vec3` plus a separate alpha rather than a `vec4`, because `vec3` is the only vector
type this API has and inventing a second one for this would be worse than one extra call. The
component keeps a single `vec4` underneath, so `set_color` leaves the alpha alone and `set_alpha`
leaves the colour alone.

Every call on this page is a no-op, or answers its neutral value, when the entity does not resolve
or carries no material component. `has_material()` tells you whether there is one, and the
component itself is added in the Properties panel.

## `get_color()`

The material's colour, as a `vec3` with the alpha dropped.

```lua
local rgb = entity:get_color()
entity:set_color(vec3.new(rgb.x, 1.0 - rgb.y, rgb.z))
```

Answers `(1, 1, 1)` for an entity that does not resolve or has no material.

## `set_color(rgb)`

Writes the colour and leaves the alpha exactly as it was.

A no-op for an entity that does not resolve or has no material.

## `get_alpha()`

The material's alpha, the fourth component of the same colour.

```lua
entity:set_alpha(entity:get_alpha() - dt)
```

Answers `1.0` for an entity that does not resolve or has no material.

## `set_alpha(a)`

Writes the alpha and leaves the colour alone. This is the fade knob for a dissolve or a spawn
effect.

A no-op for an entity that does not resolve or has no material.

## `get_emissive()`

The material's emissive colour: the light it gives off of its own.

Answers `(0, 0, 0)` for an entity that does not resolve or has no material.

## `set_emissive(rgb)`

Writes the emissive colour.

A no-op for an entity that does not resolve or has no material.

## `get_emissive_strength()`

How strong the emissive colour is.

Answers `0.0` for an entity that does not resolve or has no material.

## `set_emissive_strength(strength)`

Writes the emissive strength. A strength of zero is an ordinary, unlit surface, which is how a
glowing marker is switched off again.

A no-op for an entity that does not resolve or has no material.

## `get_shader_graph()`

The compiled `.Lshader` path this entity's material names.

This is the lower level of the two references a material can carry: it is read directly when the
entity has no material asset, which is the escape hatch for naming a compiled shader with no asset
wrapper around it. `get_material()` below is the authored one, and it wins when both are set.

Answers `""` for an entity that does not resolve or has no material.

## `set_shader_graph(path)`

Points the material at a `.Lshader` and leaves any material asset path alone.

Material resolution prefers the material asset while there is one, so setting this on an entity
that also carries a `.Lmaterial` path will not change what it renders until that path is cleared.
See [Material graphs](../../../../03-rendering/01-materials.md#material-graphs-node-based-materials).

A no-op for an entity that does not resolve or has no material.

## `set_shader_graph_scalar(name, value)`

Sets this entity's own override for one of its shader graph's named Scalar Parameters.

The override is stored on this entity's material component, by name, so two entities pointed at
the same graph can carry different values for the same parameter without touching the graph asset
itself. The value is packed into the first component of an internal vector with the rest zero, so
a Scalar Parameter reads `value` alone.

A no-op for an entity that does not resolve or has no material. A name the graph never declared is
stored anyway rather than refused, so a typo is quiet.

## `set_shader_graph_vector3(name, value)`

Sets this entity's own override for a named Vector3 Parameter. Same rules as the scalar form.

A no-op for an entity that does not resolve or has no material.

## `set_shader_graph_texture(name, path)`

Sets this entity's own override for a named Texture Parameter. `path` is asset-relative, the same
shape `set_material` takes.

A no-op for an entity that does not resolve or has no material.

## `get_material()`

The asset-relative path to the `.Lmaterial` asset this entity's material instantiates.

This is the authored material reference, and it takes priority over `get_shader_graph` when both
are set: a `.Lmaterial` names a parent shader plus a named override set, and resolution prefers
it.

Answers `""` for an entity that does not resolve, has no material component, or has no material
path set.

## `set_material(path)`

Points the material at a `.Lmaterial` asset.

It does not clear the shader graph path, which stays there as the fallback for whenever the
material path is emptied again; `get_material()` is simply the one resolution reads first.

A no-op for an entity that does not resolve or has no material.
