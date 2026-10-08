---
title: "Testing"
---

LAB has a test harness you can use for your own content, not just for the engine. A test
is a Lua file full of assertions, run against a real scene by the real runtime.

There is no unit-test layer, because there could not usefully be one: everything the engine
does needs a Vulkan device, a loaded scene and a few stepped frames. **Adding a test costs a
`.lua` file and no rebuild.**

## Running tests

Everything under `Dev/Tests/assets/tests/` (the tests of the engine's own test project, `Dev/Tests/TestHarness.lab`),
then the tests of each fixture project under `Dev/Tests/fixtures`, with a summary:

```bash
powershell -ExecutionPolicy Bypass -File scripts/Run-Tests.ps1
```

The exit code is the number of failing tests, so it works as a pre-commit gate. Options:
`-Config Debug|Release|Dist`, `-Frames <n>`, `-Filter <pattern>`, `-SkipFixtures`.

The test project holds the tests and every scene, model, texture and script they use, so nothing a test
needs comes from the sample project. A test writes only to `Dev/Out` (see "Where a test writes" below), and
after a full run the working tree is as it was: `Run-Tests.ps1` says `UNCLEAN` and lists the difference when
it is not.

A test file whose name starts with `_` is scratch — a throwaway written while chasing
something — and is skipped even by a `-Filter` that would otherwise match it.

An editor test (see below) gets more frames automatically: the suite raises the cutoff to
at least 240 for any test whose header sets `-- editor`, because a cold asset load through
the editor's own event path can otherwise lose the race with a shorter one. A scene test
loads synchronously and keeps whatever `-Frames` says.

One test on its own, with the window shown so you can watch it:

```bash
powershell -ExecutionPolicy Bypass -File scripts/Run-Test.ps1 tests/shadows.lua -Show
```

Exit codes: **0** all passed, **1** something failed, **2** the test would not load — so a
misspelt filename fails rather than passing quietly. (**3** means there was nothing to
run — build the target named by `-Config` first.)

To run one against another project (a fixture, `Dev/Tests/SurfelBenchmark.lab`, your own game),
pass `-Project`; its tests are looked up in that project's `assets/tests`:

```bash
powershell -ExecutionPolicy Bypass -File scripts/Run-Test.ps1 tests/user_saves.lua -Project Dev/Tests/fixtures/UserSaves/UserSaves.lab
```

A test whose header says `-- project: none` runs the editor with no project at all.

## Writing a test

The first line names the scene it needs:

```lua
-- scene: scenes/shadow_lights_test.Lscene
```

Kept in the test rather than in a registry, so adding one is a single file and nothing to
remember. A test with no such line runs against the project's start scene.

Then the callbacks — `on_create`, `on_update(dt)` and `on_finish` — and inside them, the
whole scripting API plus assertions:

```lua
-- scene: scenes/my_level.Lscene

local frames = 0

function on_update(dt)
    frames = frames + 1
end

function on_finish()
    expect(frames > 0, "the scene actually stepped")

    local door = scene.find("Door")
    expect(door ~= nil, "the door exists")
    expect_near(door:get_position().z, 2.0, 0.01, "the door opened")

    expect(validation_errors() == 0, "no Vulkan validation errors")
    expect(script_failures() == 0, "every script ran")
end
```

| Assertion | |
|---|---|
| `expect(condition [, message])` | |
| `expect_near(a, b [, tolerance] [, message])` | For floats; `tolerance` defaults to `0.001` |

## What a test can see

Everything a normal script can — `scene`, `physics`, `input`, `entity`, `vec3`, `quat` — plus
readers for state a script cannot otherwise reach. The whole global surface a test has,
readers included, is in [Test harness](../06-scripting/02-lua/api/testing/01-test-harness.md), and the
editor side of it in [Editor API](../06-scripting/02-lua/api/editor/01-editor-api.md):

| Reader | Returns |
|---|---|
| `entity_count()` | |
| `script_count()` / `script_failures()` | |
| `validation_errors()` / `validation_warnings()` | Vulkan validation counts |
| `shadow_view_count()` | Shadow views actually rendered |
| `shadow_cascade_count()` / `shadow_split(view)` | |
| `shadow_casters_drawn()` / `shadow_casters_culled()` | |
| `sky_mode()` | |
| `culled_count()` | Meshes the frustum test skipped |
| `mesh_cache_count()` / `shared_mesh_count()` | Distinct models parsed, and distinct models holding GPU buffers — either equal to the entity count means sharing is not happening |
| `binds_recorded()` / `binds_skipped()` / `draw_calls()` | |
| `frame_timings()` | The same per-pass GPU/CPU timings the Performance panel reports (`gpu_offscreen_ms`, `wait_imgui_fence_ms`, ...) |
| `gpu_culling_stats()` | GPU-driven culling results for the last frame whose readback fence has retired, or `nil` before one has |
| `set_gpu_occlusion(enabled)` / `set_indirect_count(enabled)` | Toggle GPU culling features mid-test |
| `recreate_render_targets()` | Resizes the render targets down and back, to exercise the image-layout/history lifecycle without changing the camera |
| `read_asset_text(path)` | Read a project asset as text |
| `shadergraph_compile(path [, boolOverrideName, boolOverrideValue])` | Compiles a `.Lshader` fixture directly through `ShaderGraphCompiler`, no live renderer or material needed. Returns a table: `success`, `error_count`, `warning_count`, `first_error`, `first_error_node`, `glsl`, `parameter_names`, ... |
| `model_load_skeleton(path)` / `model_sample_pose(path, clipIndex, time)` | Parses a glTF skeleton, or samples a pose from it, independent of rendering or playback |

### Typing and the HUD

`input.inject_text("text")` and `input.inject_key("Backspace" [, "ctrl+shift" [, mode]])` type through ImGui's own event queue, the one the
keyboard fills, so a test of a HUD text field (or of any input code) runs the real path: the field, the key bindings, and the gating that
keeps the game's keys quiet while one has the keyboard. They are read when the next frame starts, so a script's injection takes effect
in the same frame's HUD update, and a tap's press and release are separate frames (leave a frame between two taps of one key). The probes
(`ui_text_info`, `ui_text_caret_x`, a test clipboard, a stand-in for the Steam Deck keyboard) and `count_pixels` are in
[Test harness](../06-scripting/02-lua/api/testing/01-test-harness.md); `tests/ui_textfield.lua` is the worked example.

### Ray tracing

The suite has to pass on a device with no hardware ray query, so **these are a branch, not a
requirement**. Ask `ray_tracing_available()` first and return early when it is false, exactly
as `tests/restir_gi.lua` does.

| Reader | Returns |
|---|---|
| `ray_tracing_available()` | Whether this device has ray query at all |
| `ray_tracing_pipeline_supported()` / `ray_tracing_pipeline_active()` | RT pipelines available, and used this frame |
| `set_ray_tracing(bool)` | Force it off to compare against the raster path |
| `tlas_ready()` / `tlas_instance_count()` | The top-level acceleration structure |
| `blas_count()` / `blas_compacted_count()` | Cached geometry structures |
| `ray_traced_effects_recorded()` | Whether any traced effect ran this frame |
| `denoised_signal_count()` | How many of the three signals the denoiser filtered |
| `checkerboard_active()` | Whether reflections traced at half rate this frame |
| `ray_budget_scale()` | Where the dynamic budget settled, 0.25 to 1 |
| `effective_ray_quality()` | The tier actually used, after the performance mode had its say |
| `gpu_timing_supported()` | Whether timestamps drive the dynamic budget on this device |
| `renderer_tier()` / `set_renderer_tier(n)` | The capability tier, 0 to 3 |
| `running_below_authored()` | Whether the tier is honouring less than the scene asked for |
| `max_shadow_views()` / `max_ddgi_probes_per_frame()` | Per-tier budgets |
| `reflection_scale_percent()` | Reflection trace resolution against the render resolution |
| `device_local_mb()` | Largest device-local heap |

`set_renderer_tier` is how a test asserts a *ceiling* without owning four GPUs: force each
tier in turn and check what each one honours. `tests/capabilitytier.lua` does exactly that.

### Asset pipeline

The same importers the editor's Import Settings modal runs, exposed so a test can cook a
fixture with specific settings, read the result back, and reimport it — all in one run
rather than trusting the two halves separately. `tests/assetimport.lua` does exactly that.

| Function | |
|---|---|
| `import_asset(source, output [, settings])` | Runs the real importer. `settings` keys match the YAML settings' own names: `Scale`, `FlipWinding`, `RecomputeNormals`, `Primitive` for a mesh; `ColorSpace` (`"sRGB"`/`"Linear"`), `FlipVertically`, `MaxSize`, `GenerateMips` for a texture. Returns `ok, error` |
| `reimport_asset(path)` | Re-cooks a native asset from its recorded source and settings, keeping its GUID. Returns `ok, error` |
| `asset_info(path)` | Reads a `.Lmesh`/`.Ltex` container's metadata — type, GUID, source state, dimensions, colour space, vertex/index counts, bounds, and (for a mesh cooked from a glTF primitive) its recovered material — without decoding the payload |
| `cook_mesh(source, output)` / `cook_texture(source, output, isColor)` | The lower-level cook, with no import settings and no container thumbnail or timestamps |
| `cooked_mesh_sibling(meshPath)` | The `.Lmesh` filename a packaged build would look for beside a missing source mesh |
| `fs_exists(path)` / `fs_read(path)` | Check what an import did or did not write to disk, project-relative |
| `fs_list(directory)` / `fs_remove_all(path)` | List a directory, or clean up around a test's own fixtures |
| `temp_dir()` | A scratch directory a test can write to |

### Where a test writes

A test never writes into the tree. The runner gives it two places, and both are emptied the first
time the test asks for them, so a run starts from nothing and cannot read what another test left:

| Function | |
|---|---|
| `test_output_path("x.png")` | The absolute path `Dev/Out/tests/<test name>/x.png`, with its folder made. For anything written by absolute path: screenshots, saved scenes, imports and cooks the test only reads back, scratch copies |
| `test_scratch_path("models/x.Lmesh")` | The asset-relative path `_scratch/<test name>/models/x.Lmesh` inside the project's asset directory, with its folder made. For an asset that has to be among the project's assets: a path given to an MCP tool, a scene's `MeshPath` or `ScriptPath`, a material instance, a shader graph |
| `test_name()` | The test's file name without `.lua` |

Both refuse an absolute path or one with `..` (they return an empty string and log a warning).
`save_screenshot` takes a relative path as relative to the output folder; `compare_screenshot` takes an
asset path or an absolute one. `_scratch` is git-ignored and the runner deletes it after every test.

What the engine itself writes into a project (generated meshes, a Terrain plugin's heights, baked
navmeshes, save slots, an MCP import's object file) cannot be sent elsewhere. Those files are git-ignored
too, and after every test the runner removes the ignored files of the test project and says so
(`left in the test project, removed: ...`), so a test leaves the project as checked in. Only projects
under `Dev/Tests` are cleaned.

### Reading the framebuffer

```lua
local p = pixel(0.5, 0.5)          -- { r, g, b } in 0..255
local avg = average_pixel()
save_screenshot("my_test.png")      -- lands in Dev/Out/tests/<test>/
```

Coordinates are 0..1, so a test does not depend on the resolution it happens to run at.

**Use these for anything visual.** Every other assertion available is a number the CPU
computed and hoped the GPU agreed with — shadows once shipped rendering correct depth maps
and applying none of them, and only a pixel read would have caught it.

`save_screenshot(path)` writes the whole frame to a PNG (a relative path goes to the test's output folder), with alpha
forced opaque so a viewer honouring it doesn't show a screenshot of nothing; it returns
`false` rather than throwing if there's nothing to read. `save_history_screenshot(path)`
does the same for the temporal history image the next frame's TAA and reflection resolve
will actually sample — a plain screenshot can't tell "the history is black" from "the
history is fine and the sampler isn't reading it."

Readback waits for the device, so call it a few times in `on_finish`, never per frame.

## Renderer and subsystem diagnostics

Past the general readers above and the ray-tracing block below, a few subsystems expose
further diagnostic readers not listed there — mostly for that subsystem's own regression
tests, but available to any test. They are all listed, with the renderer's own quality
controls beside them, in
[Renderer diagnostics](../06-scripting/02-lua/api/testing/02-renderer-diagnostics.md):

| Group | Readers |
|---|---|
| ReSTIR DI | `set_restir(bool)` / `restir_enabled()` |
| Ray tracing | `acceleration_structure_bytes()` — total device memory the acceleration structures hold |
| DDGI | `set_ddgi_visualization(bool)` / `ddgi_visualization_enabled()`, `ddgi_gpu_tracing()`, `ddgi_active_probe_count()`, `ddgi_traced_rays()`, `ddgi_average_irradiance()` |
| FSR | `fsr_available()`, `fsr_library_loaded()`, `fsr_version()` |
| Retro resolution | `set_retro_resolution(enabled, targetHeight)` / `render_resolution()` — the internal render size, the display size, and whether upscaling (retro or TAAU) is active this frame |

### Surfel GI diagnostics

The surfel backend has its own, larger diagnostic family, since its regression tests need
to inspect cache state directly rather than only the rendered image:

`surfel_gi_available()`, `surfel_stats()`, `surfel_history_stats()`,
`surfel_cell_stats(x, y, z)`, `surfel_cell_key(x, y, z)`, `surfel_depth_bins(slot)`,
`surfel_nearest(x, y, z, maxDistance)`, `surfel_guide_bins(slot)`,
`surfel_guide_direction(bin, normal)`, `surfel_attachment(entityId [, requestedSlot])`,
`capture_surfel_frame(name)`, `set_surfel_pressure_override(pressure)`,
`set_gi_backend(backend)`, `set_surfel_rays(rays)`, `set_surfel_updates(updates)`,
`set_surfel_max_bounce_depth(depth)`, `set_surfel_debug(debug)`,
`set_surfel_ray_binning(bool)`, `set_surfel_light_tree(bool)`. See `docs/SURFEL_GI.md` for
what each one is actually reading.

## Editor tests

Add `-- editor` to the header and the test can drive editor panels through an `editor` table.
The editor is where crashes happen, and none of it is reachable from a scene test — opening
an object replaces the scene the test would be running in.

```lua
-- scene: scenes/picking_test.Lscene
-- editor
```

| Function | |
|---|---|
| `editor.open_object(path)` / `save_object()` / `close_object()` / `is_editing_object()` | |
| `editor.open_scene(path)` / `save_scene_as(path)` | |
| `editor.open_graph(path)` | |
| `editor.open_anim_graph(path)` | Opens a `.Lanimgraph` in the animation graph editor; returns whether the panel is now open |
| `editor.anim_graph_queue_add_state(x, y)` / `anim_graph_queue_add_pose_node(state, type, x, y)` | Queue an add that the panel applies **inside its own canvas**, at the point a context menu adds. Read the result with `anim_graph_last_added_state()` / `anim_graph_last_added_pose_node()` a few frames later |
| `editor.anim_graph_add_state(x, y)` / `anim_graph_add_pose_node(...)` / `anim_graph_add_transition(from, to)` | Add immediately, from outside the panel's render. Returns the index/id |
| `editor.anim_graph_open_state(index)` | Enter a state's pose graph, as double-clicking it does — a pose graph only renders while it is the open level |
| `editor.anim_graph_state_position(index)` / `anim_graph_pose_node_position(state, nodeId)` | `vec3(x, y, 1)` when it exists, `vec3(0,0,0)` when it does not |
| `editor.anim_graph_condition_has_result(transitionIndex)` / `anim_graph_state_count()` | |
| `editor.open_material_instance(path)` | Opens the Material Instance Editor for a `.Lmaterial` asset; returns whether the panel is now open |
| `editor.material_instance_parameter_count()` | The parent graph's overridable Scalar/Vector3/Texture Parameters plus Static Switches; `-1` if the hook isn't available |
| `editor.set_material_instance_override(name, r, g, b, a)` | Sets a Scalar/Vector3/Static Switch override by parameter name (`r`/`g`/`b` used per type, `a` unused) — not for Texture Parameters. Returns `false` if no instance is open |
| `editor.save_material_instance()` | Saves the open instance; returns whether it is no longer dirty |
| `editor.select(name)` / `add_to_selection(name)` / `clear_selection()` / `select_all()` | |
| `editor.selected_name()` / `selected_id()` / `selection_count()` | |
| `editor.pick(u, v [, additive])` | Viewport picking at a normalized point; returns the name of the entity hit, or `""` for a miss |
| `editor.select_in_rect(u0, v0, u1, v1 [, additive])` | Marquee selection over a rectangle in viewport fractions; returns how many entities ended up selected |
| `editor.undo()` / `redo()` | |
| `editor.delete_selected()` / `duplicate_selected()` / `gizmo_duplicate()` | `gizmo_duplicate` is alt-drag without the drag: duplicates the selection in place, leaving the copies selected |
| `editor.set_camera(x, y, z, pitch, yaw)` | Pitch and yaw in radians |
| `editor.frame_selection()` | Returns whether anything had bounds to frame |
| `editor.set_outline(width, r, g, b)` | Outline width in pixels — widen it past the shipped 2.5px so a sampled point reliably lands on the ring |
| `editor.reload_shaders()` / `shader_reload_ok()` | |
| `editor.set_mesh_path(name, path)` / `get_mesh_path(name)` / `is_mesh_loading(name)` / `mesh_info(name)` | A named entity's Mesh component: assign a path, read it back, or read `{ has_mesh, path, buffer_key, index_count, bounds_min, bounds_max }` |
| `editor.reload_mesh(name)` | What the Mesh component's **Reload Mesh** button runs: re-reads the file from disk for every entity drawing it, and returns whether the entity was found. Refused while playing, like `set_mesh_path` |
| `editor.play()` / `stop()` / `is_playing()` | Enters/exits Play from the editor — the editor-scene-to-copy transition no runtime test can reach |
| `editor.selected_position()` / `set_selected_position(x, y, z)` | The primary selection's *local* position. Use these rather than `scene.find` inside the object editor, where `scene` still binds to whatever was open when the test loaded |
| `editor.set_hidden(name, hidden)` / `set_locked(name, locked)` | Sets hierarchy visibility/lock state directly |
| `editor.viewport_mouse(u, v, down)` | Queues a mouse button at a viewport fraction across frames — how a test drives a gizmo drag or a marquee |
| `editor.hierarchy_mouse(name, lock, down)` | Clicks a hierarchy row's visibility button (`lock = false`) or lock button (`lock = true`); returns whether the button was drawn |
| `editor.key(key, down, ctrl, shift)` | Queues a key through ImGui, so shortcut routing (Ctrl+Z, Ctrl+Y, ...) is actually exercised, not simulated. A letter, or `"ESCAPE"`, `"LEFT"`, `"RIGHT"`, `"UP"`, `"DOWN"`, `"DELETE"` |
| `editor.screenshot()` | What F12 does — the same readback, converted and either handed to Steam or written as a PNG |
| `editor.build_standalone(directory [, options])` / `last_build_result()` | Runs Build Standalone to completion; `options` is `{ strip_unreferenced, all_scenes, scene }` (a start-scene override, asset-relative), plus `demo`, `steam`, `steam_app_id`, `demo_app_id` and `require_ownership` to change the build target for that one build — omitted, the project's own Build settings apply |
| `editor.list_assets(folder)` / `list_sources(folder)` | What the Asset Browser / Source Browser would list for a folder (asset-relative), names only |
| `editor.show_source_browser(show)` / `browse_assets(folder)` / `browse_sources(folder)` | Drives the browsers for real — tiles, container thumbnails — so a panel-only ImGui assert actually has a chance to fire |
| `editor.open_import_dialog(source)` / `import_dialog_open()` / `close_import_dialog()` | The Import Settings modal — see [Importing Assets](../02-building-worlds/04-importing-assets.md) |

**A crash fails the run rather than hanging it.** With a test loaded, assertion and crash
dialogs are suppressed, so the process aborts with a non-zero exit instead of waiting for
someone to click OK.

## Simulating gestures

`editor.viewport_mouse(u, v, down)`, `editor.hierarchy_mouse(name, lock, down)` and
`editor.key(key, down, ctrl, shift)` drive a mouse button or key through the same ImGui
input path a real gesture would, at a low enough level to exercise press-versus-drag,
modifier keys and shortcut routing — `tests/altdrag.lua` is built on exactly this. Sending
a down and an up several frames apart, at the same position, is a click; moving between
them past the pick-drag threshold is a drag.

**A window that has the OS focus re-reads the real cursor every frame**, and that position is queued
after an injected one, so over a run the pointer `viewport_mouse` placed is overwritten by wherever the
real cursor is -- and it is not always reproducible, because it depends on whether the window has
the focus. A test whose assertions rest on the pointer staying put should use a panel's own virtual
pointer where it has one: `editor.ui_designer_mouse(x, y, down)` puts the UI Designer's pointer on a
preview pixel through the panel's per-frame mouse code, so a drag spread over many frames (and its one
undo step, and Esc) is tested without ImGui's pointer. `tests/ui_designer_edit.lua` does both that and
the same gestures in a single call (`editor.ui_designer_drag`). The 9-slice pane's margin drag
(`editor.ui_designer_drag_slice`) and the widget palette's locks (`editor.ui_designer_locked`) have the
same kind of hook: `tests/ui_designer_slice.lua` and `tests/ui_designer_widgets.lua`; chapter
[26](../05-ui/02-ui-designer.md) describes the panel they exercise.

**What still cannot be produced this way** is anything ImGui or ImGuizmo infer from
continuous pointer motion within a single frame, or anything about how the editor UI
*looks*. For the latter, run the editor with validation on and exercise the path by hand.

## Write the failing case too

A test that has never been seen to fail is decoration. Break the thing it covers and watch it
go red before you trust it.
