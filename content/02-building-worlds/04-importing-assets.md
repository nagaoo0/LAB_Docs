---
title: "Importing Assets"
---

LAB does not load a source `.obj`, `.gltf`, `.png` or `.hdr` directly into a scene. Every
mesh and texture an entity actually references is a **native asset** — a `.Lmesh` or
`.Ltex` file — produced by *importing* the source once. This chapter is the workflow;
[`docs/ASSET_PIPELINE.md`](../ASSET_PIPELINE.md) covers the container format and cooking
pipeline underneath it, engineer-facing.

## Why two browsers

- The **Asset Browser** (`F2`) lists only native assets — the things the engine actually
  loads. This is what you drag into the viewport or onto a material slot.
- The **Source Browser** (Windows menu) lists only raw files — everything under
  `<project>/source` and your asset directory that is *not* a native asset yet. Each row
  shows its import status: **Not imported**, **Imported**, or **Source changed** (the
  source file is newer than the last import — reimport to pick up the edit).

Nothing is in both. A file that is neither a recognised source type nor a native asset
(a `.psd`, a `.txt`) shows in neither — the Asset Browser is not a general file manager.

**Nothing can be dragged out of the Source Browser.** A source has to be imported first,
same as Unreal's content pipeline. This is deliberate: it is what guarantees a build never
has to ship a raw source file the loaders were not written to read at runtime.

## Importing a file

1. Open the **Source Browser** and find the file.
2. **Double-click it**, or right-click → **Import…**, to open the **Import Settings**
   modal. **Import All** on a folder imports everything in it with default settings, for
   when you do not need to tune anything.
3. The modal shows the source, its kind, and either its dimensions (a texture) or its
   primitive count (a mesh), plus a destination folder (defaulting to the same path mirrored
   under your asset directory) and a name.
4. Set the type's own settings (below) and press **Import**.

The result appears in the Asset Browser as a native asset, with an embedded thumbnail —
for a mesh, a small rendered three-quarter view, not a generic icon.

### Mesh import settings

| Setting | Meaning |
|---|---|
| Scale | Applied once, at import. Bounds are recomputed afterwards |
| Flip Winding | Corrects a mesh that renders back-face-culled inside out |
| Recompute Normals | Ignore the source's own normals and regenerate them |
| Primitive | glTF only — which primitive of a multi-primitive mesh this import is |
| Create material assets | glTF only: one `.Lmaterial` per glTF material, shared by every primitive that uses it |
| Import vertex colors | glTF only: keep the file's `COLOR_0` colours (they multiply the albedo); off cooks every vertex white |
| Light intensity | glTF only: multiplies the intensity of every light in the file when it becomes an object, see below |

A **skinned** mesh cooks to a skeletal `.Lmesh` (influences included) and writes its rig
beside it as a `.Lskel`, plus one `.Lanim` for every clip of that skin. The files are named
from the source: `<source>.Lskel` for the first skin, `<source>_skin<N>.Lskel` after that. A
clip that fails to cook is logged and skipped, and the mesh still deforms in its bind pose.
A skinned mesh whose skeleton cannot be cooked refuses to import rather than shipping with
its skin silently lost. A cooked skeletal mesh bound to a shared skeleton cannot be rescaled
by re-cooking; reimport its source instead. See
[Components](02-components.md#skeletal-mesh).

A glTF mesh's own PBR material comes along for free the first time you place it: if the
entity's Material component is still at its defaults (or absent entirely), the importer's
recovered colour, textures, emissive factor, emissive texture and texture transform are applied
automatically. **This never overwrites a material you have already touched**: change one field
away from default and the recovered material stops applying, permanently, for that entity. The
`.Lmaterial` an import writes is never rewritten by a reimport either, so a material edited in
Blender and exported again does not refresh a `.Lmaterial` that already exists; delete it to
take the new one. See [Materials](../03-rendering/01-materials.md#emissive-textures) for what each field does.

### Shape keys

A glTF primitive with morph targets (Blender shape keys, exported with the Shape Keys option, which the
add-on turns on) keeps them. The cooked `.Lmesh` is then format version 2: the same file with the target
names, their default weights and the per-vertex deltas (positions, and normals and tangents when the
file has them) after the geometry. A mesh with no shape keys is still written as version 1, byte for
byte as before, and both versions load. Imported meshes get a **Morph Targets** component, see
[Components](02-components.md#morph-targets). Sparse accessors, which Blender writes when few vertices
move, are read. *Recompute normals* drops the normal and tangent deltas, since the normals they were
relative to are replaced; *Scale* scales the position deltas with the vertices.

### Lights and cameras

A glTF imported as a **Scene Object** (each node keeps its own transform) also brings its lights
and cameras, as entities on the nodes that hold them, with a light or camera component:

* a **point** light becomes a Point Light, a **spot** light a Spot Light with the glTF cone angles;
* a **directional** light becomes a Directional Light in physical units, its glTF lux as the Lux;
* a **camera** becomes a Camera component with the glTF vertical field of view and clip planes.
  It is never the primary camera. Orthographic cameras are skipped with a warning.

glTF measures point and spot lights in candela, and this engine's point light has no such unit
(it is a flat intensity out to its radius), so the importer pins them at a reference distance of
3 m: a light of C candela imports with intensity C / 3² / 5000, the same 1 / 5000 the Sun's lux
uses. A 1000 W Blender point light is 54351 cd and arrives at 1.2. The radius is the light's
`range` when the file gives one, otherwise the distance at which the light would have fallen
to 100 lux (23 m for that lamp). If a scene comes in too bright or too dim, set **Light
intensity** in the import settings (or the editor's saved import default) rather than editing every light; it
scales the lux and the candela alike. Colours are converted from glTF's linear values to the
colours the light's colour field holds.

An **Asset Object** import has no node graph, so it brings no lights or cameras.

### Texture import settings

| Setting | Meaning |
|---|---|
| Color Space | sRGB or Linear. Guessed from the filename (a stem ending `_n`, `_normal`, `_orm`, `_aorm`, `_rough`, `_metal`, `_mask`, `_ao`, `_height` or `_disp` defaults to Linear; everything else to sRGB) and always overridable |
| Flip Vertically | |
| Max Size | Downscales on import rather than at every load |
| Generate Mips | |

An `.hdr` source always imports as a float texture regardless of this setting — there is no
sRGB/Linear choice for one, since it is never colour-encoded to begin with.

**Colour space matters because the same PNG can mean two different things.** A material's
fixed Albedo/Normal/Metallic-Roughness slots are typed by construction — the slot itself
decides sRGB or Linear at the GPU level, and a mismatch with what you imported is logged
once rather than silently trusted. A [material graph](../03-rendering/01-materials.md#material-graphs-node-based-materials)'s
untyped texture input is the one place your import setting is authoritative, since there is
no slot to fall back on.

## Reimporting

Editing a source file after import does not update the native asset by itself: the
Source Browser row turns **Source changed**, and you reimport deliberately (or turn on
[re-importing changed sources](06-blender.md#the-editor-follows-the-file), below):

- **Asset Browser**, right-click the native asset → **Reimport** (same settings) or
  **Import Settings…** (reopen the modal to change them first).
- **Source Browser**, the row's own **Import…** does the same from the source side.

Reimporting keeps the asset's identity (its GUID) — the point is updating what a path
already refers to, not creating a second asset next to it. The entities in the open scene
that draw the mesh are reloaded there and then, whether they point at the native asset or
at the source it was cooked from; anything else that names the path picks up the new import
the next time it loads. A mesh that changed outside the editor can also be picked up without
importing it at all: **Reload Mesh** in its [Mesh component](02-components.md#mesh) re-reads
the file from disk for every entity drawing it.

A model imported as an object (**Asset Object** or **Scene Object** in the dialog, or `import_asset`
over MCP) remembers that in its meshes. Reimporting one of them re-cooks all of the model's parts
and builds the object (`objects/<name>.Lobj`) again from the new file, so a part added in the
source appears and a removed one goes, and the copies of the object already placed in the open
scene are refreshed (each keeps its position, rotation and scale). An object whose file was
changed after the import built it, by hand or by saving it from the object editor, is not
overwritten: the meshes are re-cooked and the editor says it kept the object. **Rebuild Object
from source** in the object's Asset Browser menu replaces it anyway.

A model that comes from Blender is reimported the same way: the Blender add-on writes the `.glb`
into the source folder, the editor reimports it, and **Open in Blender** on the asset goes back to
the `.blend` it was exported from. See [Blender](06-blender.md).

## Where an import lands in a build

**Build Standalone** never ships a raw source file. Every native asset a scene actually
references gets copied; a source it can still find (because the asset was never imported,
or the setting **Package only referenced content** is off) gets cooked to the sibling name
the loaders look for once the source is gone. See
[Projects & Builds](../07-projects-and-tools/01-projects-and-builds.md) for the packaging settings, and
`BuildManifest.yaml` in a build's own output for exactly what shipped and why.

## Checking your work

`tests/assetimport.lua` and `tests/materialimport.lua` cover the cooked and live-uncooked
import paths; `tests/assetbrowser.lua` asserts the native/source split described above.

The same import, reimport and cook calls are reachable from a script: `import_asset`,
`reimport_asset`, `cook_mesh`, `cook_texture` and `asset_info` are how a test drives the
pipeline without a hand on the mouse. See
[Asset tooling](../06-scripting/02-lua/api/assets/01-asset-tooling.md).
