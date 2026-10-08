---
title: "Lights and meshes"
---

What rendering components an entity carries, and the light values a script animates. See [Lua Scripting](../../index.md) for how a script runs.

These are presence tests and two pairs of live values. The other `has_*` checks (`has_material`, `has_body`, `has_character`, `has_spline`) are documented with the areas they belong to; this page is meshes, cameras and the three light types.

[C++ equivalent](../../../03-cpp/api/entity/04-components.md)

## `has_mesh()`

Whether the entity carries a `MeshComponent`, which is what makes it draw. A component test, not a loaded-asset test: an entity whose mesh path failed to load still answers `true`, and the reason is in the log rather than here.

## `has_camera()`

Whether the entity carries a `CameraComponent`. It completes the `has_*` family: a script looking for the camera among an entity's children should not have to find it by name, since renaming it in the editor would silently break the look.

A camera's field of view is scriptable too: `entity:get_fov()` and `entity:set_fov(degrees)` are its vertical FOV in degrees, which is the pair a speed effect widens.

## `has_light()`

Whether the entity carries any of the three light components: point, directional or spot.

The three are separate components rather than one with a kind field, because that is what they are in the engine: a script animating a spot's cone has no business carrying a radius it will never use. A light's other properties (colour, radius, cone angles, shadow flags) are authored per type in the Properties panel.

## `get_light_intensity()` / `set_light_intensity(value)`

```lua
local current = entity:get_light_intensity()
entity:set_light_intensity(current * 0.5)
```

The intensity of whichever light the entity carries: point, directional or spot. `get_light_intensity` answers `0` for an entity with no light component, and `set_light_intensity` is a no-op for one.

The units are the component's own scene-linear scale, with `1` as the authored default, so this is a multiplier rather than a physical figure. Two things follow from that:

- **A directional light with Physical Units on ignores its `Intensity`** and reads its `Lux` value instead. Writing the intensity on that light changes a number the renderer does not use, so a script driving one has to know which mode it is in.
- A light with shadows disabled still lights: this is brightness and nothing else.

`set_light_intensity` writes whichever of the three components exist on the entity, so an entity carrying two of them moves both.

## `get_light_indirect_intensity()` / `set_light_indirect_intensity(value)`

```lua
entity:set_light_indirect_intensity(0)
```

Scales this light's own contribution to **bounce lighting** (the surfel GI the engine traces), independently of `Intensity`, which still governs what the camera sees it light directly. `1`, the default, means "contributes to indirect exactly as much as `Intensity` already says", so a scene that never touches this is unchanged.

`0` removes the light from bounce lighting entirely while it keeps lighting everything it directly illuminates. That is what a light that should read as strong and local (a window's harsh beam, a flashlight) wants, without also flooding the room it is in with GI a viewer never asked for.

`get_light_indirect_intensity` answers `0` for an entity with no light component, which is the same value an authored zero gives. An entity with nothing to read and a light contributing nothing to indirect are indistinguishable here, so pair it with `has_light()` where the difference matters.

## Morph targets (shape keys)

A mesh imported with shape keys, from Blender or any glTF with morph targets, carries a `MorphTargetComponent` with one weight per
key. See [Components](../../../../02-building-worlds/02-components.md#morph-targets) for what a non-zero weight costs and what it cannot do. A target
is named by its string or by its index, which counts from 0 like `mesh_vertex`.

### `morph_target_count()`

How many shape keys the entity's mesh has. `0` for an entity with no `MorphTargetComponent`.

### `morph_target_name(index)`

The name of target `index`, or `""` for an index the mesh does not have.

### `set_morph_weight(target, value)`

Writes one weight, usually between 0 and 1 (`0` off, `1` fully applied). Answers `true` when it landed and `false` for an entity
with no component or a target the mesh does not have. The other weights keep the value they were showing.

### `get_morph_weight(target)`

The weight in effect: the one set, or the mesh's authored default while none has been. `nil` for an entity with no component or a
target the mesh does not have.

```lua
entity:set_morph_weight("Smile", 0.5)
entity:set_morph_weight(1, 1.0)   -- the second key
```

The tests can read the shape keys of a mesh file without an entity: `mesh_morph_targets(path)` answers
`{ count, flags, names, default_weights }` and `mesh_morph_delta(path, target, vertex)` the per-vertex delta (`x, y, z`, plus
`nx, ny, nz` and `tx, ty, tz` when the mesh has them), both with 0-based indices.

## Custom depth and stencil

```lua
entity:set_render_custom_depth(true)
entity:set_custom_stencil(200)
if entity:get_render_custom_depth() then log.info(entity:get_custom_stencil()) end
```

`set_render_custom_depth(on)` flags the entity's mesh to be drawn into the custom depth target, and
`set_custom_stencil(value)` sets the value it writes there. Both work on a Mesh or, failing that, a
Skeletal Mesh component. The value is clamped to 0..255. The getters answer `false` and `0` and the
setters do nothing on an entity with neither component. A Post Process Material reads the result
through a Scene Texture node, see [Components](../../../../02-building-worlds/02-components.md#custom-depth-and-stencil).

## Example: a lamp that dims

```lua
local base = 1
local t = 0

function on_create()
    base = entity:get_light_intensity()
    if base == 0 then
        base = 1
    end
end

function on_update(dt)
    t = t + dt
    local dim = 0.5 + 0.5 * math.sin(t * 2)
    entity:set_light_intensity(base * dim)
    entity:set_light_indirect_intensity(dim * 0.5)
end
```

Reading `base` in `on_create` and driving from it keeps the animation relative to what was authored, which is the same shape to use for any light effect that is a modulation rather than an absolute value.
