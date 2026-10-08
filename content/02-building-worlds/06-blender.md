---
title: "Blender"
---

LAB and Blender trade models as glTF (`.glb`). Blender is not a source format the engine reads: a `.blend` is
something you edit, and the `.glb` next to it is what LAB imports. A small Blender add-on, **LAB Bridge**, sends
work from Blender to a running editor, and the editor can open a source model in Blender and pick up the changes
when you save.

| Direction | How |
|---|---|
| Blender to LAB | **Send Selection** and **Send Scene** in the add-on, or **Export on Save** |
| LAB to Blender | **Open in Blender** in the editor |
| Batch work | Headless Blender jobs: `.blend` to `.glb`, LOD levels, convex hull colliders |

You need Blender 4.2 or newer. 5.2 LTS is the version LAB is tested against.

## Install the add-on

The add-on is the folder `LAB/tools/blender/lab_bridge` in the LAB checkout. Blender installs extensions from a
zip, so build one:

```
blender --command extension build --source-dir LAB/tools/blender/lab_bridge --output-dir <any folder>
```

Then in Blender open **Edit > Preferences > Get Extensions**, open the drop-down arrow at the top right, choose
**Install from Disk...** and pick `lab_bridge-0.1.0.zip`. The first time, Blender asks to allow the add-on network
and file access. It needs both: the network to talk to the editor on your own computer (127.0.0.1, nothing else),
and the disk to write the exported `.glb` into your project.

A **LAB** tab appears in the 3D viewport sidebar (press `N`). The same actions are in **File > Export** (Send
Scene) and in the object right-click menu (Send to LAB).

## Connect it to the editor

1. Start the editor with its MCP server on: **Editor Settings > MCP**, or the `--mcp` argument. The default port is
   7801. See [AI Agents (MCP)](../07-projects-and-tools/04-editor-ai-agents.md) for the server itself.
2. In Blender, press the refresh button in the LAB tab. It reads the project the editor has open and says so.

The add-on asks the editor where the project's source folder is, so there is nothing to configure while the editor
is running. When the editor is closed you can still export: set the project's `.lab` file in the add-on
preferences (or **Choose Project** in the LAB tab) and the add-on reads the source folder from it. Nothing is sent
until the editor is up; the next send, or the editor's own refresh of the Source Browser, picks the file up.

## Send Selection

Select what you want as one asset and choose **Send to LAB**. The add-on:

1. Exports the selection as `<project>/source/blender/<name>.glb`. The name is the active object's name, or the
   Name field in the operator panel.
2. Asks the editor to import it. This produces one `.Lmesh` per part and `objects/<name>.Lobj`.
3. Waits for the import (up to about 30 seconds) and, if **Place in Scene** is on, adds the object to the scene the
   editor has open, at the position, rotation and scale the active object had in Blender.

The panel in the sidebar shows how it went. Blender is not blocked while it waits.

The active object is the asset's origin. If you send several objects at once, each keeps its pose relative to the
active one, and the whole group is placed at the active object's transform. Your objects are moved to the origin
for the export and put back afterwards, so the scene is unchanged when it finishes.

Sending the same name again overwrites the `.glb` and asks the editor to reimport it. The instance already in the
scene updates; no second copy is placed.

**Save the `.blend` first.** The exported file records where the `.blend` is, so **Open in Blender** can open it
again later. An unsaved file still exports, with a warning, and without that link.

## Send Scene

**File > Export > LAB (Send Scene to Editor)**, or **Send Scene** in the sidebar, exports every visible mesh,
light, camera and armature where they are, as `<project>/source/blender/<blend name>.glb`, imports it, and places it
at the origin. Because LAB and Blender agree on the world (see below), the result sits exactly as it did in
Blender. The `.glb` becomes the scene's link: Send Scene again overwrites the same file.

Lights and cameras go through glTF too. Whether the editor turns them into light and camera entities depends on
the editor's glTF import; see [Importing Assets](04-importing-assets.md).

## Export on Save

Turn on **Export on Save** (sidebar or add-on preferences). Every time you save a `.blend` that is linked to a
glTF file, the add-on exports the scene to that file and asks the editor to reimport it. A file is linked once you
have used Send Scene on it, or when it was started by Open in Blender. The editor can also notice changed sources
by itself; see [The editor follows the file](#the-editor-follows-the-file).

The link is the scene custom property `lab_target_glb` (Scene properties > Custom Properties). It is stored
relative to the `.blend` when possible, and it is a plain property, so it survives without the add-on. Delete it to
unlink, or edit it to point elsewhere. It must stay inside the project's source folder.

## The editor follows the file

Turn on **Editor Settings > External Tools > Re-import changed source files** and the editor watches the source
file of everything it has imported. When one is rewritten (the add-on's export, a save with Export on Save, a copy
over it), the editor waits until the file has stopped changing, then reimports every asset made from it in place,
keeping their GUIDs, and rebuilds the object. Copies of the object already placed in the open scene are refreshed and
keep their position, rotation and scale. The editor does nothing with an empty file or a `.glb` that is not a glTF,
and nothing while a scene is playing or another import is running. The wait is 1.5 seconds by default; it is
`ExternalTools: AutoReimportDebounceSeconds` in `editor.yaml`.

Sources that were already out of date when the setting was switched on, or the project opened, are left alone: a fresh
checkout gives every file a new write time, and reimporting a whole project behind your back would be worse than the
state it started in. They count again as soon as they change. Reimport them yourself from the Source Browser.

The setting is off by default. Without it, a changed source shows **Source changed** in the Source Browser, and the add-on's Export on
Save still asks the editor to reimport.

**Objects are rebuilt from the file.** An object made by importing a model (Send Selection, Send Scene, the Import dialog's Asset Object and Scene Object modes) records its recipe in its meshes: the object's name, where its parts went and whether each node kept its
own transform. A reimport uses it, so a part added or removed in Blender shows up in the object. The object file
ends with a line holding a checksum of its contents. If the object was saved from the object editor or edited as
text afterwards the checksum no longer matches, and the editor leaves the object alone, re-cooks the meshes and says
so. **Rebuild Object from source** in the object's Asset Browser menu replaces it anyway. An object made by hand has
no recipe and is never rebuilt.

## Open in Blender

The editor has **Open in Blender** on a mesh in the Asset Browser, on a model in the Source Browser, on an entity
with a mesh in the Hierarchy, and next to the mesh path in the Inspector. Set the Blender program under **Editor
Settings > External Tools** (LAB looks for the usual install folders if it is empty).

* If the model's `.glb` records a `.blend` that still exists, that `.blend` opens.
* Otherwise Blender starts with the model imported into an empty scene and linked to the file, so a Send Scene (or
  Export on Save) from that session writes back over the same `.glb`. Save the `.blend` next to the model to keep
  working on it; the next export records it and the next Open in Blender goes straight to it.

## Settings

The add-on preferences (**Edit > Preferences > Add-ons > LAB Bridge**):

| Setting | Meaning |
|---|---|
| Project File | The `.lab` file. Only used while the editor is not running |
| MCP Port | The editor's MCP port, 7801 by default |
| Export Folder | Folder inside the project's source folder for exported files. Default `blender` |
| Place in Scene | Also add the object to the open scene after the first send |
| Export on Save | Export and reimport whenever a linked `.blend` is saved |

## What the export contains

The add-on uses Blender's own glTF exporter with these choices:

* Binary `.glb`, Y-up (always), custom properties as `extras`, lights, cameras and shape keys included.
* The active vertex colour layer is exported, whether or not a material uses it.
* Modifiers are applied. If a mesh has shape keys they are not, because applying them would remove the keys; the
  add-on warns when that happens. The shape keys arrive as morph targets with a Morph Targets component, see
  [Shape keys](#shape-keys).
* Lighting in physical glTF units.
* `asset.extras` carries `lab_source_blend` (the `.blend`, relative to the `.glb`, or absolute when it is on another drive) and `lab_blender_version`.

Custom properties whose names start with `lab_` on an object travel with it as `extras`. `lab_collider = hull` makes the
importer add a Convex Hull collider when the hull mesh exists (see [Collision hulls](#collision-hulls));
`box` and `mesh` and `lab_lod_ratios` (`0.5,0.2`) are read from the node and not acted on.

## Lights, cameras, colours and emission

Send Scene carries what the scene looks like, not only its meshes:

| In Blender | In LAB |
|---|---|
| Point light | Point Light. Watts become candela through Blender's exporter (W x 683 / 4 pi), then LAB's intensity at a 3 m reference |
| Spot light | Spot Light, with the cone angle and the blend as the inner angle |
| Sun | Directional Light in physical units. Blender's strength in W/m2 times 683 is the Lux |
| Camera | Camera component with the same vertical field of view and clip range; never the primary camera |
| Active colour attribute | Vertex colours, which multiply the albedo |
| Emission colour or texture, Emission Strength | Emissive Color, Emissive Texture and Emissive Strength |
| Mapping node on a texture | UV Scale, UV Offset and UV Rotation of the material |

A light or camera keeps the transform Blender gave it, and points the way it points in Blender. The
importer takes care of the one engine difference, a Directional Light shining along its own X axis rather than
its -Z. A Blender scene that arrives too bright or too dim is fixed with the **Light intensity** import setting,
see [Importing Assets](04-importing-assets.md#lights-and-cameras). Lights and cameras come with any export the add-on makes, Send Selection included, because both keep each node's
own transform; a selection that holds no light brings none.

## The coordinate rule

Blender is Z-up, glTF is Y-up, and LAB is Z-up. Blender's exporter converts `(x, y, z)` to `(x, z, -y)`, and the
object LAB builds from a glTF has a root turned 90 degrees about X, which converts back. After import a point has
the same world coordinates it had in Blender, so there is nothing to convert by hand: an object at (1, 2, 3) in
Blender sits at (1, 2, 3) in LAB. Keep the exporter's Y-up option on if you ever export by hand. The one thing the
add-on reorders is the rotation: Blender writes quaternions as (w, x, y, z), the editor takes `[x, y, z, w]`.

## Headless jobs

Three scripts in `LAB/tools/blender/headless` do batch work without a window. The editor runs them for you (below),
or you can run them yourself:

```
blender -b --python LAB/tools/blender/headless/make_lods.py -- --input crate.glb --ratios 0.5,0.2
```

Each prints a line `LAB_RESULT {json}` (Blender adds its own lines after it) and exits 0 when done, 1 on an error,
and 2 when it refuses.

| Script | Does | Output |
|---|---|---|
| `export_blend_to_glb.py` | Exports a `.blend` the way Send Scene does. `--out` chooses the file, `--collection` restricts it to one collection and the ones inside it | `<stem>.glb` beside the `.blend` |
| `make_lods.py` | Decimates the model to the given triangle shares, most detailed first (`--ratios`, default `0.5,0.2`) | `<stem>_lod1.glb`, `<stem>_lod2.glb` beside the input, or in `--out-dir` |
| `make_convex_hull.py` | One convex mesh around all the meshes, for a collider | `<stem>_hull.glb`; `--space local\|world\|auto` picks the frame |

`make_lods.py` and `make_convex_hull.py` accept `.glb`, `.gltf`, `.obj` and `.blend` input. They refuse (exit 2)
a mesh with shape keys or an armature, because decimating or wrapping it would lose them; `--force` goes ahead
anyway. Every LOD file has the same node and mesh order as the original, so the parts line up. LOD files carry no
materials (a LOD is geometry; the entity keeps its own), but keep one primitive per material slot so the numbering
still matches. A model that is not a `.blend` has its vertices welded before it is decimated: a glTF repeats a vertex
for every normal and UV, and decimating that as it comes would delete triangles and leave holes.

### Run them from the editor

The jobs run in the background, show **Blender running...** in the progress popup in the top right corner, and end
by importing what Blender wrote. A failure, and a refusal with the script's own message, comes up as a notification.
Each job also writes its output to `LAB/logs/blender_<job>_<id>.log`. The menu items need Blender to be found (see
Open in Blender); without it the editor says so and opens Editor Settings.

* **A `.blend`.** The Source Browser lists `.blend` files next to the models, but a `.blend` is never imported
  itself. Its menu has **Export to glTF with Blender**: it writes `<name>.glb` beside it and imports that as an
  object, the same way Send Scene does. The `.glb` records the `.blend` it came from, so Open in Blender from the
  object comes back to it, and exporting again refreshes the same assets in place.
* **A model, or a mesh imported from one.** The Source Browser menu (on the model) and the Asset Browser menu (on the
  cooked mesh) have **Generate LODs with Blender** and **Generate collision hull with Blender**.

The same jobs are MCP tools for agents and tests: `blender_export_blend {path, collection?, name?}`,
`make_lods {path, ratios?, force?}` and `make_convex_hull {path, force?}`. Each starts Blender and returns a
`job_id`; poll `get_import_status`. The job stays pending until Blender and the import after it are both finished, and
its `result` holds the script's own report (triangle counts, vertex counts) with what the editor did with it
(`lod_meshes`, `hull_mesh`, `rebuilt_object`). A refused input ends the job with `refused: true` and the reason in
`error`.

### LODs

`make_lods` writes `<stem>_lod1.glb` and `<stem>_lod2.glb` beside the model and cooks them as loose meshes next to
the ones already imported from it: if the model became `Rock_0.Lmesh`, `Rock_1.Lmesh`, the LODs are
`Rock_lod1_0.Lmesh`, `Rock_lod1_1.Lmesh`, `Rock_lod2_0.Lmesh` and so on. They use the scale the base meshes were
imported with. The part number stays after the LOD marker, which is how a mesh finds its LODs: the files are
looked up by name, in the folder of the mesh.

* An object built from the model is **rebuilt** afterwards, and each mesh entity in it gets its LOD 1 and LOD 2 paths
  from the files that exist. An object that was edited by hand is left alone and named in the result
  (`kept_objects`).
* **Assign sibling LODs**, next to **Reload Mesh** in the Mesh component, does the same for one entity, and the MCP tool
  `assign_lods {entity, recursive?}` does it for an entity or a whole placed object. The LOD distances are not
  touched; set them in the component.
* Building an object from a model, by any import, picks up LODs that are already there.

### Collision hulls

`make_convex_hull` writes `<stem>_hull.glb` and cooks it as `<name>_hull_0.Lmesh` next to the model's meshes. Use it
with a Collider whose shape is **Convex Hull** and whose **Collision Mesh** is that file (the **Use sibling hull**
button in the Inspector fills it in when the file is there, and **Generate with Blender** makes it). The shape is the
convex hull of the mesh's vertices, scaled by the entity's scale. Unlike a Mesh collider it is a solid, so it also
works on a dynamic body. Left empty, Collision Mesh means the entity's own mesh. At most 256 hull vertices are kept
(a dense mesh is deduplicated and thinned first), and a mesh that cannot make a hull (flat, a point, missing) is logged
and the collider uses its box.

A Blender object with the custom property `lab_collider` set to `hull` gets that collider by itself: when the model's
object is built and the hull mesh exists, the node gets a Collider with the Convex Hull shape and the hull path. It
is only the Collider; add the Rigid Body (static, kinematic or dynamic) that makes it simulate. The hull is built
in the mesh's own frame for a model with one mesh and in world space for several (`--space` overrides), so set the
property on the mesh object of a single-mesh model, or on the model's root when the hull is in world space.
Generating the hull rebuilds the object, so the order does not matter. `lab_lod_ratios` is still only read, never
acted on: LODs are made on request.

`open_in_blender.py` is what Open in Blender runs for a model without a `.blend`. It is meant for a Blender window
and stays open.

## Troubleshooting

| Symptom | Cause |
|---|---|
| "Editor not reachable" | The editor is closed, its MCP server is off, or the port differs. Check Editor Settings > MCP and the add-on's MCP Port |
| "no LAB project" | The editor is not running and no `.lab` is set in the add-on preferences |
| "this LAB editor has no '...' tool" | The editor is older than the add-on. Update the editor |
| The file exported but nothing appeared in the editor | The editor was closed during the send, or the project it has open is not the one the add-on exported into. The status line names the file; importing it from the Source Browser works |
| "timed out ... the editor may still be importing" | A large model. Look in the Asset Browser, it usually finishes |
| Warning: the `.blend` is not saved | Save it so Open in Blender can find it |
| Warning: a mesh has shape keys | Modifiers were not applied, so the exported mesh has none of its modifier effects |
| Export on Save does nothing | The scene has no `lab_target_glb`: use Send Scene once, or set the property |
| The model arrives lying on its side | The exporter's Y-up option was turned off in a manual export. Leave it on |

## Shape keys

A mesh with shape keys is exported as glTF morph targets (names in `mesh.extras.targetNames`, key values as the
default weights) and imported with a **Morph Targets** component that has one slider per key. Sending the object
again after editing a key refreshes it like any other change. Move the sliders, or call
`entity:set_morph_weight("Smile", 0.5)` from a script.

A non-zero weight draws the mesh on the GPU through the skinning pass, which only draws plain opaque materials and
is not in the ray-traced scene; see [Components](02-components.md#morph-targets) for the full list of limits. The
Basis key is the mesh itself, not a target. Animated shape keys are not imported (glTF animation channels that
target weights are skipped), and the headless LOD and hull jobs still refuse a mesh that has shape keys.

## What is not covered

The `.blend` itself never goes into a build, and LAB does not read it. Materials come across as glTF materials
(base colour, normal, metallic and roughness, emission and its texture, one texture transform), not as Blender node
trees. Orthographic cameras and animated shape keys are not imported (glTF has no area light, so Blender does not write one). Streaming every edit live is
not supported: sending is by choice or on save.
