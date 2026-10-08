---
title: "Asset tooling"
---

Cooking, importing, inspecting and reimporting native assets, the material-instance asset, and the material thumbnails, all reached from Lua.

See [Testing](../../../../07-projects-and-tools/02-testing.md) and [Lua Scripting](../../index.md).

Every call on this page belongs to the **test API**: the set of functions a `.lua` file is given when it is loaded as a test, never the set an ordinary `ScriptComponent` is given. None of these names exists in a shipped build, so none of it is gameplay API. Most of it is shared with the editor: `cook_mesh`, `cook_texture` and the four importers are the same code the Import Settings modal and Build Standalone run, which is why a scene test can drive them too (see `tests/assetimport.lua`). The pipeline they talk to, what a source file is, what a container holds, what a build cooks, is described in [Importing Assets](../../../../02-building-worlds/04-importing-assets.md) and [docs/ASSET_PIPELINE.md](../../../../ASSET_PIPELINE.md).

Two path conventions run through the page:

- An **asset-relative** path (`models/crate.obj`) resolves against the open project's asset directory. Every `output` argument here is asset-relative, and so is the `asset`/`path` the inspector calls take.
- A **literal filesystem** path is taken as written. `asset_path` is the one call that bridges the two, and the `fs_*` family in [Testing](../../../../07-projects-and-tools/02-testing.md) is on the other side.

A call that cannot do what it says answers a neutral value rather than raising: `nil`, `false`, an empty string. Nothing on this page throws at a script.

## Reading an asset back

What a test asserts on after an import or a cook: the container's own metadata (`asset_info`), where an asset-relative reference actually lands on disk, the text of a project file, and the cooked name a packaged build will look for when the source has gone.

| Call | What it is |
|---|---|
| `asset_info(asset)` | Everything the Asset Browser's details popup knows, as a table, or `nil` for a file that is not there. |
| `asset_path(path)` | The real filesystem path an asset-relative reference resolves to, unchanged when no project is open. |
| `read_asset_text(relative)` | A whole project-relative text file, or `nil` when it will not open. Deliberately narrow: the sandbox has no `io`, and a test wants to look at what the serializer wrote rather than have arbitrary file access. |
| `cooked_mesh_sibling(meshPath)` | The cooked name a packaged build looks for beside a missing source mesh: `<stem>.Lmesh` for an OBJ, `<stem>_<N>.Lmesh` for glTF primitive N. A path that already names a `.Lmesh` comes back unchanged. |

`asset_info` answers the fields every type has first:

| Field | |
|---|---|
| `type` | The asset type's id string |
| `is_container` | Whether the file carries the container header |
| `guid` | The asset's GUID |
| `file_size` | Bytes |
| `modified` | The file's write time in filesystem ticks, as a double |
| `has_thumbnail` | Whether an embedded thumbnail is there |
| `source`, `source_state` | The recorded source path, and `"none"`, `"current"`, `"changed"` or `"missing"` |
| `settings` | The import settings the asset was cooked with |

and then whatever its payload has to say. A texture adds `width`, `height`, `color_space` (`"srgb"`, `"linear"` or `"hdr"`), `mips` and `compressed`. A mesh adds `vertex_count`, `index_count`, `bounds_min`, `bounds_max`, and `skeleton` on a skeletal `.Lmesh` (relative to the mesh's own directory). A skeleton adds `joint_count` and `animations`, a 1-indexed list relative to the `.Lskel`'s directory. An animation adds `clip`, `duration` and `channel_count`. A mesh cooked from a glTF primitive that named a material also brings the recovered material with it: `material_present`, `material_color` (`{r, g, b, a}`), `material_albedo`, `material_normal`, `material_orm`, `material_emissive` (`{r, g, b}`), `material_alpha_mode` (`"opaque"`, `"mask"` or `"blend"`) and `material_asset`, the `.Lmaterial` the import created for it, empty when asset creation was off or unavailable.

## Cooking

The lower-level cook, with no import settings, no container thumbnail and no timestamps. Both answer whether the file was written.

| Call | What it is |
|---|---|
| `cook_mesh(source, output)` | Cooks a source mesh to a `.Lmesh`. `source` is an OBJ path, or a glTF path with a `#N` fragment naming the primitive. |
| `cook_texture(source, output, isColor)` | Cooks a source image to a `.Ltex`. `isColor` picks the colour space an albedo map (`true`) versus a normal or AORM map (`false`) is loaded with. |

An **empty `output`** on `cook_texture` cooks to the exact sibling name the loader looks for on its own, `<stem>.srgb.Ltex` or `<stem>.linear.Ltex`. That is what makes cooking transparent: every future load of `source` picks the cooked file up with no scene or material change. A non-empty `output` cooks to that path explicitly, for a build step that wants control over where the file lands.

## Importing

The same importers the editor's Import Settings modal runs, exposed so a test can import with specific settings, read the asset back and reimport, all in one run. All four answer `ok, error`, where `error` is the reason on a failure and empty on success.

| Call | What it is |
|---|---|
| `import_asset(source, output [, settings])` | Imports a model or an image. The kind is the file's: a `#N` fragment is the mesh importer's business. |
| `reimport_asset(asset)` | Re-cooks a native asset from its recorded source and settings, keeping its GUID. |
| `import_skeleton(source, output [, settings])` | Writes a rig on its own from a glTF. |
| `import_animation(source, output [, settings])` | Writes one clip on its own from a glTF. |

`settings` is an optional table whose keys are the settings' own YAML names.

| Source kind | Keys |
|---|---|
| Mesh | `Scale`, `FlipWinding`, `RecomputeNormals`, `Primitive`, `CreateMaterialAssets`, and for the textures the model's material names: `TextureMaxSize`, `TextureGenerateMips`, `TextureCompress` |
| Texture | `ColorSpace` (`"sRGB"` or `"Linear"`), `FlipVertically`, `MaxSize`, `GenerateMips`, `Compress` |
| Skeleton | `Skin`, `Scale` |
| Animation | `Skin`, `Clip`, `Scale` |

A key that is absent leaves that setting at its default. `Clip` left empty takes the file's first clip. Importing a skinned mesh through `import_asset` writes the rig and every clip beside it already; `import_skeleton` and `import_animation` exist for re-exporting one without the other. A source that is neither a model nor an image answers `false` with `not an importable source: <source>`.

```lua
-- scene: scenes/asset_cook_test.Lscene

function on_finish()
    local ok = import_asset("models/crate.obj", "models/crate.Lmesh")
    expect(ok, "the crate imports")

    local info = asset_info("models/crate.Lmesh")
    expect(info ~= nil and info.vertex_count > 0, "the mesh has vertices")
    expect(info.source_state == "current", "the recorded source is up to date")

    expect(cook_texture("textures/crate.png", "", true), "the albedo cooks to its sibling")

    local compiled = shadergraph_compile("materials/tint.Lshader")
    expect(compiled.success, "the graph compiles")
end
```

## Skeletons and poses

Two parser-side readers. They trigger a real glTF parse through the same `ModelLoader::Load` path a `MeshComponent` uses and report what it found, so a test can verify the parser itself, hierarchy, parent indices and bind-pose positions included, independently of animation playback or rendering, neither of which these touch.

| Call | What it is |
|---|---|
| `model_load_skeleton(assetPath)` | A table: `valid`, `joint_count`, `clip_count`, and `joints`, 1-indexed, one row each with `name`, `parent` (1-indexed, `-1` for a root) and `bind_position`. |
| `model_sample_pose(assetPath, clipIndex, time)` | Samples one clip and reports every joint's resulting mesh-space position: `valid` and `positions`, 1-indexed. `clipIndex` is **0-based**, matching `clip_count`, and `-1` samples the bind pose. |

## Material instances

A material instance is a named, shareable asset bundling a parent `.Lshader` with a default override set. `entity:set_shader_graph("materials/x.Lmaterial")` already works with no new entity-level API, because a `MaterialComponent`'s shader graph path takes either a graph or an instance. These three calls are for authoring and inspecting the instance asset itself.

| Call | What it is |
|---|---|
| `create_material_instance(parentPath, outputPath [, settings])` | Writes a `.Lmaterial`. Answers `ok, error`. |
| `material_instance_info(path)` | What the asset contains, or `nil` when it will not load. |
| `material_settings_info(path [, component])` | The settings a draw of `path` routes with: the parent's own block, each set override on top, and the entity's flags ORed in when `component` names any. |

`create_material_instance` takes an optional table. `ScalarOverrides` maps a parameter name to a table with up to `r`, `g`, `b`, `a` keys (a Scalar parameter reads only `r`, a Vector3 reads `r`, `g`, `b`), and `TextureOverrides` maps a parameter name to a path. The same table may carry the four settings overrides this asset applies on top of its parent's own settings block: `Transparent` and `AlphaCutout` (bools), `TwoSided` (bool) and `BlendMode` (`"AlphaBlend"`, `"Additive"` or `"Multiply"`). A key that is absent leaves that setting inherited, so setting one never restates the other three. An unknown `BlendMode` is refused with `unknown BlendMode '<name>'`.

`material_instance_info` answers `parent`, `scalar_overrides` (name to `{r, g, b, a}`), `texture_overrides` (name to path) and `settings`, where a field present is a field **this asset overrides**, `false` included, and a field absent is inherited from the parent.

`material_settings_info` answers two tables plus the parent when `path` is an instance: `settings` is the resolved block every reader should use (`transparent`, `alpha_cutout`, `blend_mode`, `two_sided`), and `overrides` names only what the asset at `path` itself sets. The split is what lets a test prove inheritance and independence rather than only the merged answer. `component` is an optional table with `Transparent` and `AlphaCutout` keys, standing in for the entity's own `MaterialComponent` flags.

## Material thumbnails

The browser's tiles are rendered by a small CPU rasteriser, not by the renderer, so there is no framebuffer for `pixel()` to read. These three calls are the test seam onto it: the same `RenderMaterialThumbnail` the Asset Browser uses, at the fixed thumbnail resolution the browser actually renders.

| Call | What it is |
|---|---|
| `material_thumbnail_pixel(path, u, v)` | One pixel of a freshly rendered thumbnail, `{r, g, b, a}` in 0..255, or `nil` when there is nothing to read at that point. `u`/`v` are 0..1 across it. |
| `material_thumbnail_max_brightness(path)` | The brightest single colour channel anywhere in the thumbnail, 0 when nothing was drawn. |
| `dump_material_thumbnail_png(materialPath, outputPath, size)` | Writes the same render to a PNG for a human to look at. `outputPath` is written as given, absolute included. Answers true or false, never throws. |

`path` is a `.Lmaterial` or a bare `.Lshader`. Colour and texture are extracted the same name-based, bounded CPU-evaluation way, and `material_thumbnail_pixel` decodes no albedo texture, so a material with only a `BaseColor` override reads back a pure colour sphere. `material_thumbnail_max_brightness` is the position-independent proxy for "does this sphere carry a real specular highlight": where the highlight lands depends on the light directions, and a low-roughness metal clips a channel toward 255 while a high-roughness dielectric never gets close. See `tests/material_thumbnail.lua`. `dump_material_thumbnail_png` is the one that does decode the albedo, the same plain-source-image path the browser's own asset tiles use, so a textured material's CPU sampling is visible in the output.

## Shader graphs

| Call | What it is |
|---|---|
| `shadergraph_compile(relative [, boolOverrideName, boolOverrideValue])` | Compiles a `.Lshader` through the graph compiler, with no live renderer involved, and answers one result table. |

The compiler is pure C++, so a test can exercise it directly against a fixture graph rather than needing a scene with a material assigned. The two optional arguments drive one **Static Switch** without a second binding: absent, the compile runs with no overrides at all and the graph's own authored defaults decide every switch.

The table carries `success`, `load_error`, `error_count`, `warning_count`, `parameter_count`, `glsl` (the evaluated material source) and `parameter_names`, a 1-indexed list. A file that will not load answers `success = false, load_error = true` and nothing else. When there are errors, `first_error` and `first_error_node` name the first one, which is the point: a failing assertion usually wants to prove that a **specific** compile error fires, not just pass or fail.

## Baked material-graph SPIR-V

Build Standalone compiles every reachable graph into `<build>/shaders/graphs/<hash>.spv` with an `index.yaml` naming the variants, which is what makes a shipped build independent of the shader compiler. These three test-only calls drive that lookup directly, so a test can prove a packaged build does not compile GLSL at runtime: build one, point the lookup at its `Engine/Shaders/graphs` directory, turn the compiler off with `set_perf_toggle("shadergraph.compiler", false)` and watch the hit count climb as the graph is drawn. Without the bake the same draw compiles nothing and the hits stay at zero, which is the failing case this exists to be able to see. Nothing in shipping code calls any of the three.

| Call | What it is |
|---|---|
| `baked_shader_set_directory(directory)` | Points every subsequent lookup at `directory` (which is expected to hold its own `index.yaml`) instead of the engine path, and forces the index to reload. An empty string restores the default. |
| `baked_shader_hits()` | How many times the lookup has returned real bytes since the process started, or since a call to `baked_shader_set_directory` or `baked_shader_clear_cache` reset it. |
| `baked_shader_clear_cache()` | Forces the next lookup to re-read the index off disk, and resets the hit count. It drops the in-memory SPIR-V cache with it, because that cache would otherwise answer for a key this call is asking the baked index about. |
