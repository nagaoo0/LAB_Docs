---
title: "Components"
---

Everything an entity *is* comes from its components. Add them from the bottom of the
**Properties** panel; right-click a component's header to **Copy Component** or **Paste
Component** onto another entity.

Every entity always has a **Tag** (its name), a **Transform**, and a hidden **ID**. The rest
are optional.

---

## Tag

Every entity has one, and it holds nothing but the entity's name: shown in the **Hierarchy**
panel, edited at the top of the Properties panel, and what `scene.find` and the other by-name
lookups match on. Beyond the name there is nothing to configure, so two entities sharing one
is an ambiguity rather than a second field.

**ID** and **Relationship** are real components too, but not ones an author edits: see
[Scenes & Entities](01-scenes-and-entities.md) for naming, identity and hierarchy.

---

## Transform

| Field | Meaning |
|---|---|
| Position | Local position, relative to the parent |
| Rotation | Local rotation, shown in degrees (stored as radians) |
| Scale | Local scale |

Transforms are **local**. World position is this composed with every ancestor's. Z is up.

> **Physics writes here every frame while playing.** If you move a physics-backed entity
> from a script, the engine handles the bookkeeping for you — but this is why an entity with
> a dynamic body ignores a position you type into the Properties panel mid-play.

---

## Mesh

| Field | Meaning |
|---|---|
| Mesh Path | Project-relative path to an `.obj`, a glTF primitive, or a cooked `.Lmesh` |
| LOD 1 / LOD 2 Mesh | Optional project-relative, artist-authored replacement meshes |
| LOD 1 / LOD 2 Distance | Camera distance in metres at which that replacement becomes active |
| Assign sibling LODs | A button: fills LOD 1 and LOD 2 from the `_lod1_N.Lmesh` and `_lod2_N.Lmesh` files beside the mesh, see [Blender](06-blender.md#lods) |
| Render Custom Depth | Also draws this mesh into a separate depth target and writes its stencil value, for post-process materials to read. See [Custom depth and stencil](#custom-depth-and-stencil) (`RenderCustomDepth`) |
| Custom Stencil Value | 0 to 255, written for this mesh while Render Custom Depth is on (`CustomStencilValue`) |
| Reload Mesh | A button: re-reads this mesh from disk, for every entity drawing it — the cooked file or the source it was cooked from. Use it after changing a source outside the editor; an import does it by itself. Not available while play is running |

Assign it by dragging a model from the Asset Browser, or by typing the path. Entities using
the same model share one GPU upload — the Performance panel's *Shared meshes* counts
models, not entities.

Each authored LOD must be a complete mesh in its own right. At the first configured distance
the renderer swaps the entity to LOD 1; at the second it swaps to LOD 2. Missing or failed
LOD files simply leave the previous valid level active. The engine does not delete triangles
from LOD 0 at runtime — that approach creates holes because a random subset of triangles is
not a valid simplified surface.

### Custom depth and stencil

A mesh with **Render Custom Depth** on is drawn a second time into its own depth target, and its
**Custom Stencil Value** (0 to 255) is written to a separate 8-bit target. A
[Post Process Material](../03-rendering/01-materials.md#post-process-materials) reads them back through a Scene
Texture node set to the **Custom Depth** or **Custom Stencil** channel, which is how an outline,
an x-ray or a highlight is limited to chosen meshes. Two things are easy to get wrong:

- The custom depth buffer is separate from the main one, so a flagged mesh hidden behind an
  unflagged one still writes its value. The stencil is whatever the nearest flagged mesh wrote, not
  what is visible.
- It draws the first LOD only, culls nothing, ignores alpha cutout, and a pixel no flagged mesh
  covers reads 0 for both channels.

From a script: `entity:get_render_custom_depth()`, `entity:set_render_custom_depth(on)`,
`entity:get_custom_stencil()` and `entity:set_custom_stencil(value)`, see
[Lights and meshes](../06-scripting/02-lua/api/entity/15-lights-and-meshes.md#custom-depth-and-stencil).

---

## Skeletal Mesh

For a skinned model, in place of a plain Mesh component. Holds bind-pose geometry and
skin influences; pair it with an **Animator** to actually play a clip.

| Field | Meaning |
|---|---|
| Mesh Path | A skinned glTF primitive — needs `JOINTS_0`/`WEIGHTS_0` and a skin |
| Vertex Count / Index Count | Read-only, filled in once the asset loads |
| Render Custom Depth, Custom Stencil Value | The same two fields as on a Mesh, see [Custom depth and stencil](#custom-depth-and-stencil) |

**Reload Skeletal Mesh** re-reads the file. An OBJ has no skinning data and is refused.

---

## Morph Targets

A weight for each shape key (glTF morph target) of the entity's mesh. A model exported from
Blender with shape keys brings them, and importing it, or assigning such a mesh to a Mesh or
Skeletal Mesh, adds this component on its own. Add it by hand with **Add Component > Morph
Targets** if you removed it.

| Field | Meaning |
|---|---|
| one slider per target | The shape key's name from the mesh, 0 off and 1 fully applied. Ctrl+click types a value outside 0..1 |
| Reset Weights | Back to the weights the mesh was authored with |

Only the weights are saved with the scene; the names and the deltas always come from the mesh,
so renaming or adding a shape key in Blender and reimporting needs no scene edit. An empty
weight list means the weights the mesh was authored with.

A mesh whose weights are all zero draws as the plain static mesh and costs nothing. Once a
weight is non-zero the mesh is blended on the GPU each frame, in the same pass that skins a
Skeletal Mesh, and drawn from there. That has limits, listed here because they are easy to hit:

* Only **plain opaque materials** draw this way, including a `.Lmaterial` that uses the built-in
  PBR shader. A custom shader graph, transparency or alpha cutout keeps the base mesh, and the
  component says so.
* Like a skinned mesh, a morphing mesh casts shadows and has motion vectors, but is **not in the
  ray-traced scene**: no ray-traced shadows or reflections of it, and no bounce light from it
  in ray-traced GI.
* Picking, the selection outline, mesh colliders and the DDGI CPU rays use the **base mesh**.
  Authored LODs are ignored while a weight is non-zero.
* The culling box is the base mesh grown by the largest each shape key can move any vertex, so a
  morphed mesh is never culled while visible. Weights outside 0..1 can exceed it.
* A mesh carries at most 255 shape keys. **Animated weights are not imported**: a glTF
  animation channel that targets `weights` is skipped, so set weights from a script or the
  inspector.

From a script, `entity:set_morph_weight(name_or_index, value)` and
`entity:get_morph_weight(name_or_index)` (an index counts from 0), `entity:morph_target_count()`
and `entity:morph_target_name(index)`. `edit_entity` takes `MorphTargetComponent = { Weights = {...} }`.

---

## Animator

Authors what a Skeletal Mesh plays. **This is the only animation component** — blend
spaces, state machines, IK and transitions are all *nodes inside an animation graph asset*,
not components. See [Animation](../04-gameplay/02-animation.md) for the whole system; this is the field
reference.

It has two modes, and which one you pick decides which fields below matter.

| Field | Default | Meaning |
|---|---|---|
| Mode | Single Clip | **Single Clip** plays one named clip. **Animation Graph** evaluates a `.Lanimgraph`. |

**Single Clip mode** — everything a prop with one looping idle needs, and nothing more:

| Field | Default | Meaning |
|---|---|---|
| Clip Name | (empty) | The animation clip's name, from the source file |
| Speed | 1.0 | Playback rate multiplier; negative plays backward |
| Playing | on | |
| Loop | on | |

**Animation Graph mode**:

| Field | Default | Meaning |
|---|---|---|
| Graph | (empty) | A `.Lanimgraph` asset. **Browse** picks one, **New Graph…** makes one, **Open Graph Editor** opens it |
| Parameters | — | Read-only list of this entity's *live* parameter values. Editable while playing, which is the quickest way to find out why a transition is not firing |

Two entities pointed at the same graph **share the authored graph** and nothing else: each
keeps its own current state, its own crossfade and its own parameter values. Editing the
graph and saving it reaches every entity using it on the next frame, with no scene reload.

**Sockets** (both modes) — named attachment points on this skeleton, each an offset from a
joint. Referenced by name from a Socket Attachment component on another entity (to hang a
sword off a hand), or from Lua. These used to be their own component.

**Root Motion** (Animation Graph mode) — *which* joint carries root motion is set in the
graph asset; this decides what this entity does with it:

| Mode | Meaning |
|---|---|
| None | Ignore it. The travel stays in the pose, and the mesh slides away from its entity |
| Extract Only | Take the travel out of the pose and publish it for a script to read |
| Apply To Transform | Take it out and move this entity by it every frame |

> **Use Extract Only with a Character Controller, never Apply To Transform.** Apply writes
> the transform directly, which teleports a character controller rather than sweeping it,
> so it walks through walls. Read the delta with `entity:root_motion_delta()` and feed it to
> the controller's own move call instead.

The pose is evaluated on the CPU each frame and the mesh is skinned on the GPU. Skeletal
meshes are **raster only** — they do not participate in hardware ray tracing (no shadows or
reflections *of* one yet, though they still receive both), and they are not yet frustum
culled, so a scene with many of them off-screen still pays their vertex cost.

---

## Material

Controls how a surface responds to light. See [Materials & Textures](../03-rendering/01-materials.md) for
how to use these together.

| Field | Meaning |
|---|---|
| Color | Base colour tint, RGBA. Multiplied with the albedo texture |
| Albedo Texture | Base colour map |
| Normal Texture | Tangent-space normal map |
| Metallic-Roughness Texture | Packed AO / roughness / metallic map |
| Emissive Color | Light the surface gives off regardless of what is shining on it |
| Emissive Strength | Separate from the colour so it can exceed 1 without clamping to white |
| Transparent | Draws after deferred lighting with alpha blending, instead of in the opaque pass |
| Alpha Cutout | Marks foliage/fence-style cutout geometry for the ray-traced path (still draws opaque in the raster pass) |

An untextured material renders using **Color** alone, which is a perfectly good way to block
out a scene.

A material can instead point at a compiled **material graph** (a `.Lshader` file, shown
as **Shader Graph** in the Properties panel), built in the node-based shader editor, with
the fixed fields above ignored in favour of the graph's own pipeline. Once a graph is
assigned, its exposed Scalar/Vector3/Texture parameters appear inline in the Material
panel as per-instance overrides — that is what lets two entities share one compiled graph
and still look different without recompiling it. See [Materials & Textures](../03-rendering/01-materials.md)
and [Visual Scripting](../06-scripting/01-visual-scripting.md).

---

## Camera

| Field | Default | Meaning |
|---|---|---|
| FOV | 45 | Vertical field of view, degrees |
| Near Clip | 0.1 | |
| Far Clip | 1000 | |
| Exposure | 1.0 | HDR exposure multiplier used when this camera drives Play mode |
| Primary | true | Whether play mode renders through this camera |

Aspect ratio comes from the viewport, so it is not stored. A camera's **scale is ignored** —
scaling a camera should not warp what it sees.

Play mode renders through the primary camera. See [Play Mode](../01-getting-started/04-play-mode.md).

## Renderer Settings

Backed by `PostProcessComponent`, but the Add Component menu and the panel header both
call it **Renderer Settings**. Add it to a scene settings entity to control the final
image. The component exposes AgX contrast and saturation, plus Temporal AA, history
weight, and rejection threshold. AgX is always the tonemapper; these controls tune its
response rather than switching to another operator.

It is also where the rest of the renderer's per-scene settings live: GTAO and Bloom
(see [Lighting](../03-rendering/02-lighting.md)), ReSTIR direct illumination, and the whole *Ray tracing*
section — traced shadows, occlusion, reflections and global illumination, plus the quality
mode, ray budget and denoiser controls. Those have a chapter of their own:
[Hardware Ray Tracing](../03-rendering/03-ray-tracing.md).

One Renderer Settings component governs the whole scene. The Add Component menu greys the
item out, with a tooltip naming the entity that already has one, once any entity does.

Every field also has a project-wide default, set from **Project Settings → Rendering**. A
scene's own component only *means* the fields it has explicitly touched — each field's row
in the panel carries a small checkbox that turns that field's override on or off. Leaving a
field un-overridden means it tracks whatever the project default is, including future edits
to that default; overriding it pins the scene to its own value regardless of what the
project default later becomes. A component added fresh from the Add Component menu starts
with every field at the project's current default and nothing overridden.

---

## DDGI Volume

A dynamic diffuse global-illumination probe volume, following this entity's transform.
Probes refresh incrementally so a scene never stalls updating them all at once. Full
field-by-field usage guidance is in [Lighting & Shadows](../03-rendering/02-lighting.md#ddgi-probe-volume).

| Field | Default | Meaning |
|---|---|---|
| Enabled | off | |
| Volume Size | 12 × 12 × 8 | World-space extent; the entity's world scale expands it further |
| Probes X / Y / Z | 8 / 8 / 6 | Probe grid resolution, 1–32 per axis |
| Probes / Frame | 4 | CPU tracer budget only; the GPU tracer refreshes every probe every frame and ignores this |
| Rays / Probe | 24 | |
| Update Budget | 2.0 ms | CPU time allowed per frame; stops even with probes remaining |
| Max Ray Distance | 30 m | Keep it close to the volume's own coverage |
| Hysteresis | 0.95 | Ceiling on how much of a probe's history survives an update |
| Min Hysteresis | 0.60 | Floor the blend falls to when a probe's estimate jumps, so a light switching on converges fast |
| Normal Bias / View Bias | 0.15 / 0.10 | Push the sample point off the shaded surface / toward the camera, to stop leaking and grazing-angle seams |
| Backface Threshold | 0.25 | Fraction of a probe's rays hitting backfaces before it is classified inside geometry and stops contributing |
| Scrolling | off | Follows the camera, snapped to whole probe cells |
| GPU Tracing | on | Hardware ray queries instead of the CPU BVH; shades each hit with the surface's own material so coloured bounce carries through. Falls back to the CPU tracer where ray query is unavailable |
| Intensity | 1.0 | Scales the final indirect diffuse contribution |

Up to eight enabled volumes share one 65,536-probe pool; an excess volume is skipped with a
one-time warning rather than silently truncated. Ray-traced global illumination, where
enabled, *replaces* this rather than adding to it — see
[Hardware Ray Tracing](../03-rendering/03-ray-tracing.md).

---

## Fog

A simple distance fog applied after lighting.

| Field | Default | Meaning |
|---|---|---|
| Color | grey-blue | |
| Start | 15 m | Where the fade begins |
| End | 50 m | Fully fogged past this distance |

**The first Fog component in the scene wins**, the same rule as Sky — fog is a scene
setting, not something several entities blend together.

---

## Directional Light

The sun: parallel rays, direction only, no position and no falloff.

The light has no direction of its own. It shines along its entity's world +X axis, read from
the world transform every frame, so a sun parented under a rotated rig turns with it. The
Properties panel shows that axis as two angles and writes the entity's rotation when you
drag them.

| Field | Default | Meaning |
|---|---|---|
| Elevation | from the rotation | How far below the horizon the light points, in degrees: 0 is horizontal, 90 straight down |
| Azimuth | from the rotation | World heading the light travels along, in degrees from +X toward +Y |
| Color | white | |
| Intensity | 1.0 | |
| Cast Shadows | off | |
| Shadow Distance | 40 | Half-width of the shadowed area, in world units, centred on the camera |
| Shadow Cascades | 3 | How many nested boxes the distance is split into |
| Shadow Bias | 0.0015 | Depth slop, scaled by surface slope in the shader |
| Shadow Strength | 1.0 | 1 is fully occluded; less lets some light through |

**Only the first directional light with Cast Shadows on actually casts.** There is one sun
shadow, and the sun is the light whose shadows anyone notices.

Maximum 4 directional lights per scene.

---

## Point Light

Radiates in every direction from a position.

| Field | Default | Meaning |
|---|---|---|
| Color | white | |
| Intensity | 1.0 | |
| Radius | 1.0 | Falloff range |
| Cast Shadows | off | |
| Shadow Bias | 0.0035 | |
| Shadow Strength | 1.0 | |

**A shadowed point light costs six shadow views** — it shadows in every direction — out of a
scene-wide budget of 16. That is why it is off by default.

Maximum 16 point lights per scene.

---

## Spot Light

A cone from a position, aimed by the entity's rotation.

| Field | Default | Meaning |
|---|---|---|
| Color | white | |
| Intensity | 1.0 | |
| Inner Cone Angle | 15 degrees | Full brightness inside this |
| Outer Cone Angle | 30 degrees | Falls to zero at this |
| Cast Shadows | off | |
| Shadow Range | 25 | How far the shadow frustum reaches |
| Shadow Bias | 0.0025 | |
| Shadow Strength | 1.0 | |

The cheapest kind of shadow caster: one view, because a spot light already *is* a frustum.
Raising **Shadow Range** costs depth precision; it does not make the light reach further —
the light's own falloff comes from the cone angles.

Maximum 8 spot lights per scene.

---

## Ambient Light

| Field | Default |
|---|---|
| Color | white |
| Intensity | 0.1 |

A flat fill so nothing is pure black. A **Sky** or **HDRI** with *Contribute Ambient* on
supersedes this with something far better.

---

## Sky

A procedural gradient sky, drawn wherever there is no geometry. Needs no asset.

| Field | Default | Meaning |
|---|---|---|
| Zenith Color | blue | Straight up |
| Horizon Color | pale | At the horizon |
| Ground Color | brown | Straight down |
| Intensity | 1.0 | |
| Horizon Falloff | 2.5 | How sharply zenith gives way to horizon. Higher keeps the horizon band tight, which is what a clear day looks like |
| Contribute Ambient | on | Feed the ambient term from the sky rather than from an Ambient Light |
| Ambient Intensity | 0.35 | |

**The first sky found wins.** Two skies is not a thing.

---

## HDRI Environment

An equirectangular HDR image used as the background.

| Field | Meaning |
|---|---|
| Texture Path | Project-relative. A `.hdr` loads as float; an ordinary LDR image works and is just less bright at the top end |
| Intensity | |
| Rotation | Yaw, in degrees. The one adjustment an HDRI always needs — the sun in the image is never where the scene wants it |
| Contribute Ambient | |
| Ambient Intensity | |

**HDRI takes precedence over Sky** when both are present. Ambient from either is an average
of the image rather than true image-based lighting — much closer than a flat grey, and it
costs nothing.

---

## Particle Emitter

A stream or burst of camera-facing billboarded quads — smoke, sparks, a muzzle flash.
Particles only simulate while playing, exactly like physics, so nothing moves until you
press Play.

| Field | Default | Meaning |
|---|---|---|
| Enabled | on | A paused emitter keeps whatever particles already exist but spawns no more |
| GPU Simulation | off | Moves spawning, ageing and movement onto the GPU. See below |
| Texture | empty | Project-relative image, tinted by Start/End Color. Empty draws a plain white quad |
| Burst Count | 0 | 0 is continuous, spawning at Emission Rate for as long as the emitter is enabled. Nonzero fires exactly that many particles once, when play starts, and nothing more after |
| Emission Rate | 10/s | Particles per second; ignored once Burst Count is nonzero |
| Max Particles | 256 | Live particle cap |
| Lifetime Min / Max | 0.5 s / 1.5 s | Each particle's lifetime is randomised between these |
| Initial Velocity | (0, 0, 1) | |
| Velocity Spread | (0.5, 0.5, 0.5) | Random offset added to Initial Velocity on each axis independently, for a cheap cone-ish spread with no real direction distribution to author |
| Gravity | (0, 0, -9.81) | Added to velocity every tick |
| Drag | 0 | Slows velocity over time; 0 disables it |
| Start Color / End Color | white / white, transparent | The particle's colour and opacity fade linearly between these over its life — no curve editor |
| Start Size / End Size | 0.1 / 0.1 | Half-size in world units, faded the same way |
| Shader Graph | empty | A compiled material graph driving the particle instead of Texture and the colour fade. See below |

Assign the texture by dragging an image from the Asset Browser, or by typing a
project-relative path.

A Particle Emitter can instead point at a compiled **material graph** (a `.Lshader` or
`.Lmaterial` file, shown as **Shader Graph** in the Properties panel), built the same way as
a Material's own graph. Once assigned, it replaces Texture entirely — only the graph's
Albedo, Emissive and Opacity are read, since a camera-facing quad has no lit, normal-mapped
surface for its other outputs to describe. The Start Color/End Color fade still applies on
top, tinting and fading the graph's output over the particle's life, because that is the
particle's own animation rather than material authoring.

**GPU Simulation** moves spawning, ageing and position/velocity integration onto a compute
dispatch that runs once per frame over a buffer sized for Max Particles, instead of the CPU
updating a per-particle list every tick — useful for an emitter with a large Max Particles
that would otherwise cost CPU time. The result looks and behaves the same either way, and a
GPU emitter can use Shader Graph exactly like a CPU one. Toggling this mid-play drops
whichever side's particles were already in flight, so choose one before pressing Play rather
than switching while an effect is running.

---

## Script

| Field | Meaning |
|---|---|
| Script Path | A `.lua` file or a `.Lgraph` visual graph |

Scripts run only while playing. See [Lua Scripting](../06-scripting/02-lua/index.md) and
[Visual Scripting](../06-scripting/01-visual-scripting.md).

---

## Native Script

| Field | Meaning |
|---|---|
| Class | The name of a behaviour the project's native module registered |

Adds C++ gameplay code to an entity, from the module the project's **Native Module** setting
names. The class has to match a name the module registered, or the behaviour never runs. Once a
module is loaded the field is a list of the names it offers rather than a text box, because a
typo here is a behaviour that silently does nothing.

An entity can run several classes at once, and each entry gets its own instance, so naming the
same class twice runs two of them.

The line under the block reports which of three states you are in: no module loaded, a class the
module does not register, or a module running N behaviours with M disabled. That last one is the
difference between a scene that works and one that looks right and is inert.

A behaviour's own fields appear under its class here once it declares any, one row each. Only a
row you actually change writes, so opening the panel is not an edit.

Behaviours run only while playing, exactly like scripts. See
[C++ Scripting](../06-scripting/03-cpp/index.md).

---

## Rigid Body

Makes an entity participate in the physics simulation. **It needs a Collider too** — a body
with no collider is ignored.

| Field | Default | Meaning |
|---|---|---|
| Type | Static | `Static` never moves, `Dynamic` is simulated, `Kinematic` is moved by you and pushes things |
| Mass | 1.0 | |
| Use Gravity | on | |
| Restitution | 0.0 | Bounciness. 0 is dead, 1 bounces back to its drop height |
| Friction | 0.2 | 0 is ice; above 1 is legal and grips harder |
| Linear Damping | 0.05 | Velocity bled off per second — this is how you model drag |
| Angular Damping | 0.05 | |
| Gravity Scale | 1.0 | Per-body. 0 floats, 2 is heavy, negative falls upward |
| Lock Position X/Y/Z | off | |
| Lock Rotation X/Y/Z | off | |

> **Surface properties combine between the two bodies in contact** (as a geometric mean), so
> setting one side is rarely enough. A bouncy ball on a dead floor is only half as bouncy as
> the ball alone.

> **Locking every translation axis is not the same as making a body static.** It still
> collides and still pushes what it touches; it just cannot be moved itself.

---

## Collider

The shape used for collision. **It needs a Rigid Body too.**

| Field | Meaning |
|---|---|
| Shape | `Box`, `Sphere`, `Capsule`, `Mesh` or `Convex Hull` |
| Collision Mesh | Convex Hull only: the mesh whose vertices are wrapped. Empty means the entity's own mesh. A hull made in Blender is `<name>_hull_0.Lmesh`, see [Blender](06-blender.md#collision-hulls) |
| Size | Box dimensions, and the box a Mesh or Convex Hull falls back to when no shape can be built |
| Radius | Sphere / capsule radius |
| Height | Capsule height |
| Is Trigger | Detects what enters it but stops nothing |

Capsules stand up along Z, matching LAB's Z-up convention.

A trigger reports **overlap** events instead of collision events, and only notices *moving*
bodies — a static wall sitting inside a trigger never fires anything, because nothing ever
happens there.

---

## Character Controller

For a player or NPC steered directly, rather than pushed by the solver — Jolt's
`CharacterVirtual`. Walks up steps, holds its footing on slopes, and never tips over, none
of which a dynamic Rigid Body/Collider does. **Replaces** the Rigid Body/Collider pair
rather than joining it; the Add Component menu only offers it when the entity has no Rigid
Body, and if both somehow end up on one entity the character is ignored (logged as a
warning) since both would write the entity's transform.

| Field | Default | Meaning |
|---|---|---|
| Radius | 0.3 m | |
| Height | 1.8 m | Total, including the capsule caps |
| Max Slope | 45° | Steeper counts as a wall — the character slides rather than walking up it |
| Step Height | 0.35 m | Highest step climbed without jumping |
| Mass | 70 kg | What it weighs when it pushes a dynamic body — nothing pushes it back |
| Push Force | 100 | Force available for shoving dynamic bodies aside; 0 still collides but moves nothing |
| Layer | 0 | Same collision-layer table as Rigid Body |

**Grounded**, **Ground Normal** and **Velocity** are shown read-only — live state written by
the physics step each frame while playing, and what a script reads through
`entity:is_grounded()` / `entity:get_ground_normal()` / `entity:get_character_velocity()`.
Move it with `entity:move(velocity)` and `entity:jump(speed)`. See [Physics](../04-gameplay/01-physics.md).

---

## Constraint

A physical link between this entity's body and another one, or the world — a door, lid,
lever, swing, chain or rope, differing only by **Type**. Needs a Rigid Body on this entity.

| Field | Default | Meaning |
|---|---|---|
| Type | Point | `Fixed` welds; `Point` is a ball joint; `Hinge` turns about one axis; `Slider` slides along one axis; `Distance` holds a range of distances (rope/spring) |
| Connected To | World | The other body, picked by name in the panel. World attaches to level geometry rather than another entity |
| Anchor | (0,0,0) | Joint point, in this entity's local space |
| Connected Anchor | (0,0,0) | The same point, in the other end's local space |
| Axis | +Z | What a Hinge turns about or a Slider travels along, in this entity's local space (Hinge/Slider only) |
| Use Limits | off | Hinge/Slider/Distance only — an unlimited Hinge is a wheel |
| Min / Max Limit | -45 / 45 | Degrees for Hinge, metres for Slider/Distance |

The other end is stored as a stable entity **id**, not a handle, so it survives a reload.
Applied when Play starts; changing it mid-play has no effect on the running simulation. See
[Physics](../04-gameplay/01-physics.md).

---

## Prefab Instance

Not added by hand. Marks the root of an entity tree that came from an `.Lobj` file and
records which one, so saving the object can bring the copies already placed in your scenes
up to date. See [Objects](03-objects.md).

---

## Persistent

No fields — a marker. Add it to a game manager, player controller, or anything else that
must survive `scene.open_scene()` at runtime: everything else in the scene is destroyed and
replaced by the new level's entities, a Persistent entity is not (its hierarchy is detached
to the scene root first if its parent would otherwise be destroyed with it).

---

## Socket Attachment

Follows a named socket on another entity every frame, which is how a sword ends up in a hand.
The socket itself is authored on the target's **Animator**, by name; see
[Animation](../04-gameplay/02-animation.md) for that side of it.

| Field | Default | Meaning |
|---|---|---|
| Target | (none) | The socket-owning entity, chosen from the entities that carry an Animator. Stored as a stable id, so it survives a reload |
| Socket Name | empty | The socket's name on that skeleton, for example `hand_r` |
| Offset Position | (0, 0, 0) | This entity's own offset from the socket |
| Offset Rotation | (0, 0, 0) | The same, in degrees |
| Offset Scale | (1, 1, 1) | The same, as a scale |

The offset is there because a sword's own origin is rarely the hand joint's origin.

**This component overwrites this entity's Transform every frame** with the socket's world
transform, so a script that moves the entity itself is overwritten on the next frame. With no
target (the default), nothing is resolved and the transform is left completely alone.

---

## Ragdoll

Which joints of a skeletal entity get their own Jolt body during Play, linked by joints that
mirror the skeleton's own hierarchy. A ragdoll is physics rather than animation: while it is
dynamic, the solver owns the named joints outright. See [Animation](../04-gameplay/02-animation.md) for how
it sits beside an Animator, and [Ragdoll](../06-scripting/02-lua/api/entity/09-ragdoll.md) for the Lua
side.

| Field | Default | Meaning |
|---|---|---|
| Bones | empty | The joint names that each get a body. Sparse is fine (spine and arms, skipping fingers) and the order does not matter |
| Bone Radius | 0.1 | One radius for every bone |
| Mass | 5.0 | One mass for every bone |
| Start Dynamic | off | What a Play session starts as. Authoring input, read once when the ragdoll builds |
| Per-Bone Overrides | empty | Per joint: a mass (`-1` means "use Mass") and swing/twist limits in degrees |
| Physical Animation | empty | Per joint: a weight from 0 to 1 that pulls it toward its live animated pose while the ragdoll is dynamic |

**Nothing is built until Play.** The first time the scene steps a skeletal entity carrying
this component with a live physics world, the bodies are created from the pose the entity is
in at that moment rather than the bind pose, so a character killed mid-swing does not pop to
rest first. After that, `set_ragdoll_dynamic` is the live switch, and Start Dynamic is only
where a fresh Play begins.

A joint name that is not on the current skeleton is skipped at the build with a warning in the
log, and an entity whose names all miss is simply left without a ragdoll.

---

## Reflection Capture

A baked positional probe that gives nearby shiny surfaces a reflection of the room instead of
just the sky. It sits at the entity's own position, and a pixel outside its influence radius
keeps using the global sky or HDRI.

| Field | Default | Meaning |
|---|---|---|
| Resolution | 128 px | Cube face size. Small on purpose: this feeds a prefiltered specular chain, not a mirror-sharp reflection |
| Influence Radius | 10 m | How far from the capture its baked result reaches |
| Intensity | 1.0 | Scales what the capture contributes |

The bake happens **once, automatically, the first time a loaded scene sees the component**,
and after that only when you press **Capture**. There is no per-frame recapture, unlike a DDGI
volume, so move geometry or lights and press Capture again yourself.

The bake only draws entities with an ordinary Material component. An entity shaded by a
material graph is not in the draw list, so a room built from graphs reflects as though those
surfaces were not there. `tests/reflection_capture.lua` is the fixture.

---

## Scene Capture

A camera that renders the scene into a small image of its own, live: the feed of a security
camera on a monitor, a mirror, a portal preview. The image is shown on a HUD sprite (the
Sprite's **Capture** field, `set_ui_capture` from a script), and in this component's own
inspector as a small preview.

The entity's transform places the camera and it looks down its local -Z, like a camera does. A
Camera component on the same entity, if there is one, supplies the lens (FOV, near, far,
exposure) and the fields below are ignored; give it a Primary of false or it also becomes the
view the game is drawn from.

| Field | Default | Meaning |
|---|---|---|
| Width / Height | 320 x 240 | The image size in pixels. 16 to 2048 |
| Rate | 12 Hz | How often it renders. 0 renders only on `capture_now` |
| Active | on | Off freezes the picture on its last frame and costs nothing |
| Field Of View | 60 | Vertical, in degrees |
| Near / Far | 0.1 / 200 | Clip planes |
| Target | none | A [render target asset](../03-rendering/01-materials.md#render-targets-live-feeds-on-materials-and-the-hud) (`.Lrt`) to draw into instead of this capture's own private image. Width and Height are ignored: the asset gives the size, format and clear colour (`Target` in a scene file) |

**It is cheap on purpose.** At most one capture renders per frame across the whole scene, the one
that has waited longest, so ten cameras at 12 Hz still add one small render a frame at most.
`capture_now` (the inspector's **Capture Now** button) jumps the queue, and works on an inactive
capture too, once.

**What it draws.** Opaque static meshes and skinned meshes (a character walking past the camera
shows up), lit by the scene's lights, sky and DDGI. Static meshes outside the camera's view are
skipped on the CPU.

**What it leaves out.**

- Transparent meshes.
- Material-graph meshes are drawn flat, with the default material, because a capture does not run
  shader graphs.
- Screen-space GI (ReSTIR GI, radiance cascades, surfels, the GI reference), ReSTIR direct
  lighting, ray-traced effects, bloom, motion blur, depth of field and TAA are all off for it.
- The sun's shadow cascades are fitted to the main camera, so the sun's shadows can be missing
  or coarse in a capture that looks somewhere the main camera does not. Point and spot light
  shadows work.

A capture's image is made one frame after the component appears, and again after a resize.

**Drawing into a render target.** With a **Target** set, the picture goes into a shared render target
asset instead of a private image, and anything that names the same `.Lrt` shows it: HUD sprites
(the Sprite's Texture) and 3D materials (an albedo or emissive texture, or a material graph texture
node). Several captures may draw into different targets. Two captures on the *same* target take turns
(still one render a frame), so the target shows whichever rendered last; give each camera its own asset
when you want both pictures. The image exists while at least one capture targets the asset, and is freed
with the last one. `entity:set_capture_target(path)` changes it from a script, and the inspector's
preview follows it.

`tests/scene_capture.lua` and `tests/render_target.lua` are the fixtures. See [HUD](../05-ui/01-hud.md#live-camera-feeds-and-the-cctv-look).

---

## Sprite

A single coloured or textured rectangle for the HUD: a health bar, a panel, an icon.

| Field | Default | Meaning |
|---|---|---|
| Color | white | A tint multiplied into the texture, or the colour outright with none set |
| Texture | empty | An asset-relative image, or a render target asset (`.Lrt`) for a live picture. Empty draws the colour alone |
| Corner Radius | 0 px | Rounds the corners. Half the sprite's size is a circle |
| Border Width | 0 px | A border drawn inside the edge |
| Border Color | black | Shown once Border Width is above zero |
| Softness | 0 px | Feathers the edge over this many pixels. A soft drop shadow is a dark sprite behind the panel with Softness set |
| Slice Left / Top / Right / Bottom | 0 texels | 9-slice border widths in texels of the image; any above 0 turns 9-slice on (`SliceBorder` in a scene file). Corner Radius, Border Width and Softness are ignored while it is on |
| Slice Scale | 1 | The on-screen size of one border texel, before the HUD scale (`SliceScale`) |
| Slice Fill | Stretch | How the edges and centre fill: Stretch, Tile or Tile Fit (`SliceFill`, saved as 0, 1 or 2) |
| Draw Center | on | Off leaves the middle out, a hollow frame (`SliceDrawCenter`) |
| Capture | none | A [Scene Capture](#scene-capture)'s live picture in place of the texture (`CaptureEntity`) |
| CCTV Look | off | Scanlines, grain and a vignette over the picture (`CctvEffect`), with Noise (`CctvNoise`), Scanlines (`CctvScanlines`), a tint (`CctvTint`) and Pixelated (`CctvPixelated`, nearest filtering) |

**A bare Sprite renders nothing.** It only draws as a HUD element when the same entity also
carries a **UI Transform**, which is what places it on screen. Add the UI Transform to the
entity first, then the Sprite or the Text.

The rectangle is drawn as a signed-distance rounded shape, so a panel, a pill or a circle
needs no texture and stays crisp at any size. A texture with the Slice fields set is drawn as a
9-slice instead (corners at their natural size, edges and centre filling the rest); see
[9-slice sprites](../05-ui/01-hud.md#9-slice-sprites).

---

## UI Transform

Screen-space placement for a HUD element, and the component that turns a Sprite or a Text on
the same entity into part of the HUD. Without it there is nowhere on screen for either to go.

| Field | Default | Meaning |
|---|---|---|
| Anchor Min | (0, 0) | The point pinned in the viewport, as a fraction: 0 is the left/top edge, 1 the right/bottom. The anchor preset picker sets this for you. Saved as `Anchor` |
| Anchor Max | (0, 0) | The far corner of the anchor rectangle. Different from Anchor Min on an axis and the element stretches with its parent there |
| Pivot | (0, 0) | The point of the element that sits at the anchor, as a fraction of its own Size. Ignored on a stretched axis |
| Offset | (0, 0) px | Where the pivot sits relative to the anchor; on a stretched axis, the left/top inset |
| Offset Max | (0, 0) px | On a stretched axis, the right/bottom edge relative to Anchor Max (shown as an inset in the inspector) |
| Size | 100 × 100 px | The element's own size; on a stretched axis only a cache of what it resolves to |
| Layer | 0 | Paint order among HUD elements: higher draws on top |
| Visible | on | A hidden element is not drawn and receives no interaction |
| Relative | off | Anchor against the parent entity's UI rect instead of the screen. A relative element also inherits its parent's visibility and draws at the parent's layer plus its own |
| Clip Children | off | Cuts relative descendants to this element's rect, which is how a scrolling list is made |
| Interactive | on | Off lets the pointer pass through to whatever is underneath, which is what a label on a button wants |

**+y is down.** Anchor (0, 0) is the top-left corner and (0.5, 1) is bottom centre, where a
subtitle belongs, so a positive Offset y moves the element further down. Positions and sizes
are UI units, scaled to the real viewport by the project's reference height, so a layout
authored for a 1080p reference height looks the same on a Steam Deck. Anchors, presets and
stretching are covered in [HUD](../05-ui/01-hud.md#anchors-points-and-stretching).

Hit testing is updated once per frame from the pointer position, after that frame's scripts
have had their turn. Only visible, interactive elements take part, and where two overlap the
topmost wins: a higher Layer first, then the deeper element. Scripts read the result from
`ui_hovered`/`ui_clicked` on the entity and from the `ui` table, see
[HUD interaction](../06-scripting/02-lua/api/ui/01-hud-interaction.md) and
[HUD](../06-scripting/02-lua/api/entity/10-hud.md).

---

## Text

A HUD text label. The glyphs come from a signed-distance font atlas that is scaled to the font
size at draw time.

| Field | Default | Meaning |
|---|---|---|
| Text | "Text" | UTF-8, and a newline breaks a line |
| Color | white | |
| Font Size | 24 px | |
| Alignment | Left | Left, Center or Right, inside the element's own Size |
| Vertical | Middle | Middle, Top or Bottom |
| Wrap | off | Breaks lines at the element's width |
| Line Spacing | 1.0 | A multiplier on the line height |
| Outline | 0 px | Outline width, in pixels at the font size |
| Outline Color | black | Shown once Outline is above zero |
| Shadow Offset | (0, 0) px | Zero draws no shadow |
| Shadow Color | black, 50% alpha | |

Like a Sprite, **a Text needs a UI Transform on the same entity** to be placed and drawn, and
renders nothing without one. Its text, font size, wrapping and measured size are all reachable
from a script, see [HUD](../06-scripting/02-lua/api/entity/10-hud.md).

---

## UI Line

A straight line with round ends, drawn between two HUD elements or two points: a link on a network map, a wire, a divider. Needs a UI Transform on the same entity for its layer, visibility and clip, and its Interactive switched off so the empty rect does not swallow clicks.

| Field | Default | Meaning |
|---|---|---|
| From Entity / To Entity | none | An element each end follows, at the centre of its rect |
| From / To | (0, 0) / (100, 0) | The point for an end with no entity, in UI units from the top-left of the line entity's own rect |
| Thickness | 2 | UI units |
| Softness | 1 | Feather on the edge; more is a glow |
| Color | white | |

Fields in full, and a script example, in [HUD](../05-ui/01-hud.md#lines). From a script, `set_ui_line` and
`set_ui_line_color`, see [HUD](../06-scripting/02-lua/api/entity/10-hud.md).

---

## UI Button

Opt-in button behaviour for a HUD element: hover, press and click events for scripts
(`ui.events()`), and the element's own Sprite and Text tinted to match (Normal, Hover, Pressed
and Disabled tints with a Fade Time), plus optional click and hover sounds. Needs a UI Transform
and a Sprite or Text on the same entity. An element without one is left exactly as it is. Every
field, and how it fits the frame, is in [UI Widgets](../05-ui/03-ui-widgets.md).

---

## UI Toggle

A two-state HUD widget (checkbox, switch, radio button): a click flips Is On and scripts get a
`value_changed` event. A non-empty Group makes toggles with that Group in the scene a radio group, and
Graphic names an element (a check mark) drawn only while it is on. Same tints, fade and sounds as the
UI Button. See [UI Widgets](../05-ui/03-ui-widgets.md#toggle).

---

## UI Slider

A draggable HUD value between Min and Max, with an optional Step, a Direction, and a Fill and a Handle
element it drives. A press jumps to the pointer and a drag follows it, also outside the slider. See
[UI Widgets](../05-ui/03-ui-widgets.md#slider).

---

## UI Scroll List and UI Scrollbar

A scrolling window: the element's rect shows part of taller (or wider) content, a child that is normally a
container with Fit Content. The wheel, a drag on the content and a scrollbar move it, and a drag does not
click the button it started on. UI Scrollbar goes on the bar's track, with its thumb as the Handle, and is named in
the list's V Scrollbar or H Scrollbar. The offset is run-time state, not saved. See
[UI Widgets](../05-ui/03-ui-widgets.md#scroll-lists).

---

## UI Focus and UI Focus Scope

Used only when the project turns on UI focus navigation. **UI Focus**, optional on a Button, Toggle or Slider, names
explicit Up / Down / Left / Right neighbours, marks the Default Focus widget, or keeps a widget out of navigation
(Focusable off). **UI Focus Scope** on a panel confines navigation to what is inside it while it is visible (a pause
menu, a dialog), and is where `cancel` goes. See [UI Widgets](../05-ui/03-ui-widgets.md#focus-and-navigation).

---

## UI Text Field

A single-line text input: a Relative child with a Text shows what is typed (Text Entity), another a hint while it is empty (Placeholder
Entity). A click, or an accept on a focused one, starts editing; Enter submits, Escape cancels. Max Length, a Filter (Integer, Decimal,
Alphanumeric), a password mask, a caret, a selection and the clipboard; while it is being edited the game's keys are silent.
`get_value()` is the text. See [UI Widgets](../05-ui/03-ui-widgets.md#text-field).

---

## UI Terminal

A scrolling console: output lines above a prompt line you type on, with scrollback, history, Tab
completion, wrapping and clickable link spans. Needs a UI Transform on the same entity; give it a
Sprite for the background. Prompt, Initial Text, Max Lines, History Size, font size and colours,
Padding, Rich, Caret Blink, Read Only and Disabled are saved with the scene. The lines, the typed
text, the history and the scroll are run-time state and are not saved. Every field, the keys, links
and events are in [UI Widgets](../05-ui/03-ui-widgets.md#terminal); the `term_*` calls are in
[HUD](../06-scripting/02-lua/api/entity/10-hud.md).

---

## UI Layout and UI Layout Element

A layout container (row, column or grid) that places its Relative child HUD elements, and the optional
component on a child that tunes it (minimum size, expand, stretch ratio, order, ignore layout). The
children's rects are worked out at layout time and never written. See
[UI Widgets](../05-ui/03-ui-widgets.md#layout-containers).

---

## Network Identity

Marks an entity as one that takes part in networking, so its state and RPCs reach the other peers.
Add it from Add Component. Only entities with it are replicated, and a prefab spawned over the
network needs it on its root entity. See [Networking](../04-gameplay/04-networking.md).

---

## Blockout Model, Blockout Brush and Blockout Operation

The components behind the blockout tools: a Blockout Model owns the baked mesh of a level piece, a
Blockout Brush holds one convex solid, additive or subtractive, and a Blockout Operation is the
group that combines the brushes under it. They are normally made by the Blockout tools rather than
by hand. See [Blockout Tools](05-blockout-tools.md).

---

## Audio Source

A clip this entity can play, spatialised at the entity's own position unless you turn that
off. One component is one sound: a second sound at the same time from the same entity is what
`audio.play_2d` is for, since that one is not tied to an entity at all.

| Field | Default | Meaning |
|---|---|---|
| Clip | empty | A `.wav`, `.ogg`, `.mp3` or `.flac`, relative to the asset directory. There is no cooked form, so this names a raw file: drop one here from the Source Browser or type the path |
| Volume | 1.0 | |
| Pitch | 1.0 | Playback rate multiplier |
| Loop | off | |
| Play On Start | off | Fires the moment Play starts, or the moment the entity is spawned mid-play, so an ambience or an engine hum needs no script just to get going |
| Ambient | off | An outside-facing ambience (rain, wind, distant traffic) rather than a sound with a real position: with a listener's acoustics running, the acoustics system drives this source's volume and pan every tick instead of leaving it at its authored volume and centred pan |
| Bus | SFX | Music, SFX or UI. The bus volumes are what an options screen writes to, from Lua |
| Spatial | on | Off makes this a flat, non-attenuated sound that follows the entity in name only |
| Min Distance | 1.0 | Where attenuation begins. Inactive while Spatial is off |
| Max Distance | 50.0 | Where it ends. Inactive while Spatial is off |

Sound exists only while playing: the engine behind it is created when Play starts and
destroyed when it stops (one per running scene, not a global), so nothing continues across a
scene switch. There is no in-editor preview either, so press Play to hear it.

**A machine with no output device makes every audio call a silent no-op, not an error**, and
`tests/audio.lua` has to pass on one. **A destroyed entity's sound is stopped rather than left
to finish**, because the entity handle it is tracked under gets recycled; a sound that has to
outlive its emitter is played non-spatially with `audio.play_2d`. See
[Sound](../06-scripting/02-lua/api/entity/12-sound.md) and [Audio](../06-scripting/02-lua/api/audio/01-audio.md).

---

## Audio Listener

Where sound is heard from. **The first one found in the scene wins**, and a scene with none
placed hears through the primary camera instead, the same fallback a shipped game gets without
adding anything.

| Field | Default | Meaning |
|---|---|---|
| Acoustics | on | Raytraced audio: rays bounce around this listener every tick to drive muffle, echo/reverb and directional ambience. Off leaves plain distance and pan spatialisation running with no raycasting cost |
| Ray Count | 16 | Rays cast per tick |
| Max Bounces | 3 | How many times a ray may bounce before it stops |
| Max Ray Distance | 343 m | How far a ray may travel. 343 is one second of travel at the speed of sound, the reference implementation's own default |
| Ray Scatter | Sphere | Sphere suits a scene with real verticality (multi-storey buildings, pits). Horizontal is cheaper and enough for a flat-ish scene, since half a sphere's rays spend themselves on floor and ceiling bounces that tell the horizontal room shape nothing new |
| Muffle Enabled | on | Muffling driven by the rays that are blocked |
| Muffle Interpolation | 5.0 | How quickly the estimate follows a change |
| Echo Enabled | on | Reverb from the estimated room |
| Room Size Multiplier | 2.0 | Multiplies the estimated room size before it becomes reverb decay or send level, to account for sound travelling to a wall and back rather than one way |
| Echo Interpolation | 5.0 | |
| Ambient Enabled | on | The directional ambience estimate that Ambient sources read |
| Ambient Interpolation | 5.0 | |

These tunables live on the listener rather than on a scene settings entity because the rays
are cast around *this* entity, so its configuration travels with it. Acoustics costs roughly
Ray Count × (1 + the number of enabled spatial sources) raycasts per tick, which is worth
knowing before raising Ray Count on a busy scene. With no listener placed at all, acoustics
simply does not run: there is nothing to read the tunables from.

> **Only one listener is ever used, and extras are removed for you.** Beyond the first, extras
> are stripped when the scene loads and again just before Play starts, with a warning naming
> the count. The entity survives, only the component goes: hand-editing a listener onto a
> second entity, or instantiating a prefab that carries its own, is how one appears.
> `tests/duplicate_audio_listener.lua` and `Dev/Tests/assets/scenes/duplicate_audio_listener_test.Lscene`
> are the fixture.

---

## NavMesh Volume

Marks a region of the scene that a walkable Detour navmesh is baked over: the mesh the agents'
paths are searched on.

| Field | Default | Meaning |
|---|---|---|
| Extent | 20 × 20 × 10 | Half-extents around the entity's position, in metres, so the region is the entity's world position plus and minus this. The entity's scale does not change it |
| Cell Size | 0.3 m | Rasterisation cell width. Smaller is a finer navmesh and a heavier one |
| Cell Height | 0.2 m | Cell height, and the vertical resolution of the heightfield |
| Agent Radius | 0.4 m | The clearance the bake leaves around geometry |
| Agent Height | 1.8 m | The clearance a surface needs above it to count as walkable |
| Agent Max Climb | 0.4 m | The tallest step bridged without a jump |
| Agent Max Slope | 45 degrees | The steepest surface that counts as walkable |
| Baked | (never) | Read-only: the `.Lnavmesh` this volume last baked to, empty until the first bake |

**Bake Navmesh** walks the scene's static colliders, rasterises them into the volume and
writes a fresh file. It is synchronous, so a large volume takes a moment. That button is the
only thing that ever bakes: a reflection capture bakes itself automatically, but walking every
collider in a volume is a deliberate action, so nothing polls for it on load.

One volume is one navmesh, and the first volume with a baked asset is the one that gets
loaded. Multiple volumes and tiled streaming are not what this component does yet.

> **A navmesh has to exist before Play starts.** A volume's **Baked** path is loaded in
> `OnRuntimeStart`, ahead of any script, so a scene that only bakes from a script's
> `on_update` is too late for an AI controller: the entry state evaluates before that frame's
> scripts, its `MoveTo` finds no navmesh, the task fails, and the tree moves on without the
> entity ever moving. `tests/behaviortree.lua` was written against exactly that failure. Bake
> at edit time, or from `on_create` at the very latest.

Without a volume, or with a volume that has never been baked, the scene simply has no navmesh:
`entity:move_to()` answers `false` and nothing moves.

Recast and Detour assume Y-up and LAB is Z-up. The conversion between them is a rotation
rather than an axis swap, which is what keeps a floor from being read as a ceiling and baking
an empty navmesh.

---

## Nav Link

A jump, drop or ladder for the navmesh: an off-mesh connection from this entity's position to
**End**, for two pieces of ground the bake cannot join itself.

| Field | Default | Meaning |
|---|---|---|
| End | (3, 0, 0) | Where the link lands, in this entity's local space |
| Bidirectional | on | Off for a one-way link: paths then only use it from start to end |
| Radius | 0.5 m | How far from each end the bake looks for walkable ground to attach it to |
| Arc Height | 1 m | How high a crossing rises above the line between the ends; 0 is straight, for a ladder |

It is baked into the volume that contains both its ends, so re-bake after moving or changing
one. An agent whose path uses it walks to the start, follows the arc to the end at its own
speed, and walks on. See [Nav Links](../04-gameplay/03-navigation-and-ai.md#nav-links).

---

## Nav Agent

A mover that follows the scene's baked navmesh. It only ever *wants* a destination: something
else sets it, either a script (`entity:move_to(target)`) or a behaviour tree's own task, and
the scene's update does the walking.

| Field | Default | Meaning |
|---|---|---|
| Radius | 0.4 m | Must not exceed the baked navmesh's own Agent Radius |
| Height | 1.8 m | |
| Speed | 3.5 m/s | |
| Acceleration | 8 m/s² | |
| Stopping Distance | 0.15 m | Arrived once within this of the final corner |
| State | Idle | Read-only: Idle, Moving, Arrived or Failed. The destination and the path are runtime state and are not saved |

Radius and Height are authoring notes rather than steering inputs: a path search uses fixed
extents, so what actually decides clearance is the **Agent Radius** the navmesh itself was
baked with. Keep the two at or under it and they describe a character that fits.

Paired with a **Character Controller**, the agent is steered through it: the scene sets the
controller's move velocity each frame and the physics step that follows consumes it, so the
agent is pushed through collision rather than dragged through it. With no character controller
the entity's Transform is advanced directly instead.

> **The steering is horizontal.** Only x and y are driven, so rising ground does not carry the
> agent up a slope on its own: following the terrain's height is left to whatever gave the
> agent its destination.

The agent is steered at the very top of the frame, before the physics step, which is why a
call from a script's `on_update` steers from the next frame on (the nav update has already run
for the frame that script is in). An entity with no Nav Agent is not left out either: calling
`entity:move_to()` on it gives it one with these defaults. See
[AI](../06-scripting/02-lua/api/entity/14-ai.md).

---

## AI Controller

Drives this entity through a behaviour tree: states holding tasks, and the transitions that
leave them. It is a different system from the animation graph, despite both having been called
a state tree at one point.
[Behaviour trees (ai)](../06-scripting/02-lua/api/ai/01-behaviour-trees.md) covers a task script's side
of it.

| Field | Default | Meaning |
|---|---|---|
| Asset | empty | A `.Lbehavior` this entity's own tree can be loaded from or saved to, so two entities pointed at the same file start from the same authored states and transitions |
| Tree | empty | The tree itself, carried inline in the component and in the scene file. **Open State Tree Editor** edits it, **New Behavior Tree…** creates a blank asset to point this field at |
| Runtime | — | Read-only live state: which state is current, and whether its tasks have completed. Reset when an asset is loaded |

**Load and save are both explicit.** **Load From Asset** overwrites this entity's own tree and
resets its current state, so local edits since the last load or save are lost, and nothing
reacts to the file changing on disk by itself.

> **The entry state starts evaluating on the very first frame, before that frame's scripts
> run.** Everything the tree needs on that first tick has to exist before Play starts: a
> navmesh baked in a script's `on_update` arrives too late for a `MoveTo` in the entry state,
> and the failure is quiet.

> **A task that failed looks like a task that finished.** The runtime tracks whether a state's
> tasks are complete, not whether they succeeded, so a transition gated on that fires straight
> past a state whose task failed on its first tick.

---

## Spline

A smooth curve through control points in the entity's local space: a race line, a rail, a
camera track. On its own it is a path a script can query. A **Spline Mesh** turns one into
geometry.

| Field | Default | Meaning |
|---|---|---|
| Points | empty | The control points. Each one has a **Position**, a **Roll** in degrees and a **Width** |
| Closed | off | Joins the last point back to the first |
| Length | — | Read-only: the curve's length in metres and its point count |

The curve is a centripetal Catmull-Rom through every point, so it passes through each control
point exactly, needs no tangent handles to author, and cannot form a cusp or a self-loop
between two points. Roll and Width are interpolated with the same cubic, so banking and
narrowing ease in and out rather than kinking at a point.

Width is not used by the spline itself: it is a scale the spline's users read, a track's width
or a tube's radius. Roll banks the frame, positive raising the right side, and the frame is
built against world up. A curve that runs straight up or down is carried through rather than
supported, so **vertical loops are not a thing**.

Click a point in the panel to select it and move it with the gizmo; `+` inserts a point after
it and `x` deletes it. Adding the component from the Add Component menu starts you with three
points, so there is something to grab. Scripts query the live curve in the editor and in a
running build, see [Splines](../06-scripting/02-lua/api/entity/13-spline.md).

---

## Spline Mesh

Geometry extruded along a spline: a road, a wall, a tube, a rail. It follows a Spline on this
entity or on another one it names.

| Field | Default | Meaning |
|---|---|---|
| Spline Entity | 0 | The entity whose Spline is followed. 0 uses this entity's own. The panel prints the name it resolved, or "no Spline found" |
| Profile | (-1, 0) to (1, 0) | The cross-section, a list of points in the spline frame's (right, up) plane, in metres |
| Closed Profile | off | Joins the last profile point back to the first, which is what makes a tube |
| Scale With Width | on | Multiplies the profile's x by the spline point's Width |
| Scale Height Too | off | Multiplies the profile's y by Width as well, which is a tube's radius. Leave it off for walls |
| Segment Length | 2 m | How much curve one extruded segment covers |
| V Per Metre | 0.1 | How fast the V texture coordinate runs along the curve |
| Start Distance | 0 m | Where along the spline the extrusion begins |
| End Distance | -1 | Where it ends. -1 runs to the end of the spline |
| Offset | (0, 0) | Moves the profile within its own plane |
| Flip Normals | off | |
| Baked Mesh | — | Read-only: the generated file this entity's Mesh component points at |

> **The mesh is baked, never generated at runtime.** The editor extrudes it to
> `generated/splines/<entity id>_<hash>.obj` and points this entity's Mesh component at that
> file, which is what lets ray tracing, mesh colliders and packaging treat it as an ordinary
> mesh. Nothing regenerates during play: a packaged build just loads the baked file.

The file is named by a hash of everything that shapes it, so an unchanged spline keeps the
file it already has, and an undo that restores an old shape re-bakes the file its hash names.

**A profile's surface faces left of each edge**, in the (right, up) plane: a road listed left
to right faces up, and a closed profile listed counter-clockwise faces inward (a tunnel),
clockwise outward (a tube).

**Spline Entity is a stable id, so a copied or instanced pair keeps following the original
spline.** If the copy should follow a curve of its own, it needs a different spline to follow.

---

Some areas documented above have chapters of their own: the NavMesh Volume, Nav Agent
and AI Controller are covered in [Navigation & AI](../04-gameplay/03-navigation-and-ai.md); the UI
Transform, Sprite, Text and UI Line in [HUD & on-screen UI](../05-ui/01-hud.md); the UI Button, Toggle,
Slider, Terminal and layout containers in [UI Widgets](../05-ui/03-ui-widgets.md); Network Identity in
[Networking](../04-gameplay/04-networking.md); and the Blockout components in
[Blockout Tools](05-blockout-tools.md).

## Adding a component

**Properties → Add Component** offers, in this order: Mesh Component, Native Script, Skeletal Mesh Component, Animator Component, AI Controller, Socket Attachment Component, Ragdoll Component, Morph Targets, Material Component, Camera Component, Renderer Settings, DDGI Volume, Reflection Capture, Scene Capture, NavMesh Volume, Nav Link, Script, Rigid Body, Collider, Constraint, Character Controller, Nav Agent, UI Transform, Sprite, Text, UI Line, UI Button, UI Toggle, UI Slider, UI Scroll List, UI Scrollbar, UI Focus, UI Focus Scope, UI Text Field, UI Terminal, UI Layout, UI Layout Element, Ambient Light, Sky, HDRI Environment, Fog, Directional Light, Point Light, Spot Light, Persistent, Network Identity, Audio Source, Audio Listener, Particle Emitter, Spline, Spline Mesh, Blockout Model, Blockout Brush, Blockout Operation. Most items are hidden once the entity already has that component, since
adding one replaces it with a fresh default; a few (Constraint, Character Controller) are also
gated on what else the entity has, noted above. These labels are the menu's own, and the test
and agent API accepts exactly them, so it is `Mesh Component` and not `Mesh`.

There is deliberately no component for blend spaces, state machines or IK, since those are
nodes inside an animation graph asset rather than components. See [Animation](../04-gameplay/02-animation.md).
