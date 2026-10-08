---
title: "Test harness: assertions, readback and counters"
---

What a Lua test asserts with, what it can read back out of the frame, and what a build left
on disk. None of this is gameplay API: these names are registered by
`ScriptEngine::LoadTest`, so they exist while a test is loaded (in the editor under `--test`,
and in a standalone runtime the same way) and a script in an ordinary play session does not
have them at all.

See [Testing](../../../../07-projects-and-tools/02-testing.md) for the harness itself: how the suite is run, the
`-- scene:` header line and the callbacks a test defines. See
[Lua Scripting](../../index.md) for the chapter this reference belongs to.

## `expect(condition [, message])`

Reports a pass or a failure, and keeps going. It does not throw: a run that stopped at its
first failure would report one problem per run, and the point of running unattended is to
come back to the full list. A failure is written to the log as `[test] FAILED: <message>`,
and the number of failures is what decides the suite's exit code.

Returns the condition itself, so a test can put it in an expression. The message is
optional, but a failure with none is a line that names nothing.

```lua
expect(shadow_view_count() == 10,
    "sun (3 cascades) + spot (1) + point (6) = 10 shadow views, got " .. shadow_view_count())
```

## `expect_near(a, b [, tolerance] [, message])`

For floats. Reports whether `a` and `b` are within `tolerance` of each other, which defaults
to `0.001`. A failure names both numbers: `expected <b> +/- <tolerance>, got <a>`. Returns
whether it passed.

```lua
expect_near(door:get_position().z, 2.0, 0.01, "the door opened")
```

Reach for it whenever the value came out of a float pipeline, and plain `expect` for counts
and booleans.

## Pixel readback

What was actually drawn, which is the only evidence that survives the whole pipeline.
Everything else a test can see is a number the CPU computed and hoped the GPU agreed with:
shadows once shipped rendering correct depth maps and applying none of them, and no
CPU-side assertion would have caught it.

Readback waits for the device, so call these a few times in `on_finish`, never once per
frame.

## `pixel(u, v)`

One pixel of the finished frame, as a table `{ r, g, b }` with each channel in `0..255`, or
`nil` when there is nothing to read (no renderer yet, or the readback failed).

Coordinates are `0..1` across the render target rather than pixels, so a test does not have
to know the resolution it happens to be running at. A coordinate outside the range is
clamped rather than refused.

```lua
local centre = pixel(0.5, 0.5)
expect(centre ~= nil, "the frame could be read back")
if centre then
    expect(centre.r + centre.g + centre.b > 20, "the scene still rendered")
end
```

This reads the sRGB-encoded output, not display-linear values. A test comparing a readback
against an analytic figure has to decode it first, the way `Dev/Tests/assets/tests/denoise.lua` does
with its own `display_linear` helper.

## `average_pixel()`

The mean of the whole image, as `{ r, g, b }` in floats on the same `0..255` scale, or `nil`
when there is nothing to read. Coarse on purpose: it answers "did anything render at all"
and "did this change make the scene brighter", which are the two questions a screenshot
would otherwise be squinted at for.

## `save_screenshot(path)`

Writes the whole frame to a PNG, and returns `true`, or `false` when there is nothing to
read or the file will not open.

`pixel` answers a question you already know how to ask. This is for the ones you do not:
flicker, blotches, a filter blurring something it should not. Those are judged by looking.
A relative path is relative to the test's output folder, `Dev/Out/tests/<test name>/` (an absolute one is used
as it is), and the parent directory is created if it is missing. Alpha is forced opaque, because the readback carries whatever the
swapchain had there and a viewer that honours it would show a screenshot of nothing.

A screenshot is a diagnostic, so failing to take one returns `false` rather than throwing
and never fails the run.

```lua
save_screenshot("my_test.png")   -- Dev/Out/tests/<test name>/my_test.png
```

## `save_history_screenshot(path)`

The temporal history image, as the next frame's TAA and reflection resolve will see it.
Returns `true` or `false`, the same way `save_screenshot` does.

A plain screenshot cannot tell "the history is black" from "the history is fine and the
sampler is not reading it", and the reflection resolve went black for exactly that reason.
When a frame looks wrong and the weights or the reprojection are suspect, this is the image
to look at.

It resolves the path the same way, a relative one being relative to the test's output folder.

## `validation_errors()`

How many Vulkan validation errors have been reported in this run, counted by the
application rather than by the scene. A test asserting zero is the cheapest way to catch a
descriptor, barrier or image-layout mistake that the picture happens to hide.

## `validation_warnings()`

The same count for warnings. Worth asserting on any path that touches descriptor pools or
binding layouts: adding a binding without resizing the pool produced exactly one of these,
which the sky test pins.

```lua
expect(validation_errors() == 0, "no Vulkan validation errors")
```

## `script_failures()`

How many scripts failed to load, or were disabled by an error. A graph that compiles to
broken Lua shows up here and nowhere else a test can reach: otherwise it is a line in the
log, which a passing test would happily ignore. The count stays exact even when the details
below have stopped being collected.

## `script_failure_details()`

The detail behind that count, as a 1-indexed table of `{ path, entity, phase, message }`.
Native failures appear here too, with a phase of `"native"`. Only the first 64 are kept, so
a run that fails in a loop counts every failure and records the first 64.

The entity is captured by name at the moment of the failure, because entt recycles handles
and a failure read a minute later would otherwise name whatever entity now sits in that
slot.

## `script_count()`

How many script instances are loaded and running, one per entity whose script resolved. The
test's own file is not one of them: it is loaded by the same engine through a separate path.

## `clock_ms()`

A monotonic wall clock in milliseconds, as a number. Only a difference between two readings means
anything. It is how a test times something, since the standard `os` library is not available to scripts:

```lua
local start = clock_ms()
for _ = 1, 100 do entity:get_ui_rect() end
print(string.format("%.3f ms per call", (clock_ms() - start) / 100))
```

## `entity_count()`

How many entities the bound scene holds. Counted through the tag view, since every entity
the scene creates carries a `TagComponent`, so a scene that expects eight entities reads
eight whatever else they carry.

## `color_to_render(r, g, b)`

An authored colour as the renderer uses it, back as `{ r, g, b }`. A test that computes an
expected value analytically should start from this rather than from the authored numbers, so
the expectation follows the colour pipeline in either mode.

`Dev/Tests/assets/tests/gi_reference_furnace.lua` takes `rho` this way:

```lua
local rho = color_to_render(0.6, 0.6, 0.6).g
```

## `world_to_viewport(x, y, z)`

Where a world point lands in `pixel`'s `0..1` coordinates, as `{ u, v }`, or `nil` for a
point behind the camera (and `nil` with no renderer). It projects through the view-projection
the renderer last rendered with, so a test that wants to sample the point it placed has to
read this after the frame it is talking about.

## `host_platform()`

`"windows"` or `"linux"`. A test that has to name a platform-specific file, a native module
being the one that matters, has no other way to ask: the sandbox opens `base`, `math`,
`string`, `table`, `coroutine` and `utf8`, so there is no `os` and no `package` to read a
separator or an extension out of either.

## `temp_dir()`

A scratch directory outside the repository a test can write to.

## `test_output_path(relative)`, `test_scratch_path(relative)`, `test_name()`

Where a test writes what it produces; see [Testing](../../../../07-projects-and-tools/02-testing.md#where-a-test-writes).
`test_output_path("x.png")` is the absolute `Dev/Out/tests/<test name>/x.png`, for anything written by
absolute path. `test_scratch_path("models/x.Lmesh")` is an asset-relative path inside the test project's
`_scratch/<test name>/` folder, for an asset that has to be among the project's assets (an MCP tool's `path`,
a scene's `MeshPath`, a material instance). Both make the folder, empty it the first time the test uses
either, and answer `""` with a log line for an absolute path or one with `..`. The runner deletes `_scratch`
after the test; `Dev/Out` is kept until the test runs again. `save_screenshot("x.png")` writes to the output
folder too. `test_name()` is the test's file name without `.lua`.

## `require(name)`

Runs a project module once and hands every caller the same value, so scripts and tests can
share code and tables. Returns whatever the module returned, or its environment when it
returns nothing.

A bare name has its dots treated as directory separators and is looked for under the
project's `scripts/` first, then as an asset path: `require("chem.rules")` runs
`scripts/chem/rules.lua`. A name that already ends in `.lua` or `.Lscript` is looked up as
given and with a `scripts/` prefix.

A failed load is `nil` after a line in the log rather than an error through C, and a module
that requires itself is refused with a cycle message. Modules live as long as the Lua state,
which is one play session.

## `print(...)`

Writes a line through the engine log, arguments separated by tabs, non-strings passed
through `tostring`. It exists because the default `print` would go to a console a packaged
build does not have.

## Files on disk

The sandbox has no `io`, so these are the only file access a test has. Each takes a literal
OS path, built from `temp_dir()` or a build output directory. Nothing here raises: a refusal
returns `false` and writes a line to the log.

| Call | Answers |
|---|---|
| `fs_exists(path)` | Whether the path exists. |
| `fs_read(path)` | The whole file as a string, or `nil` when it cannot be opened. |
| `fs_list(directory)` | Every regular file under the directory, as a 1-indexed table of paths relative to it with forward slashes. An empty table for a directory that is not there. |
| `fs_set_readonly(path, flag)` / `fs_is_readonly(path)` | Set or read the read-only attribute of a file in the temp directory, for a test of a save onto a read-only target. |
| `fs_hold_open(path, ms)` | Windows: holds a file in the temp directory open without share-delete for `ms` milliseconds on a background thread, so a save over it meets a busy file. `false` where that is not possible. |
| `fs_remove_all(path)` | Whether the tree was deleted. Refused, with `false`, for anything outside the system temp directory and `Dev/Out/tests` (a test's own output), because a test that could delete a project by typo is not a test anyone wants. |
| `fs_rename(from, to)` | Whether the move happened. `from` must be inside the temp directory, `Dev/Out/tests` or the active project's assets, which is what lets a test rename a real imported asset to prove a reference still resolves after. |
| `fs_copy(from, to)` | The same discipline as `fs_rename`, for a test that needs to leave its checked-in fixture alone. An existing destination is overwritten. |

`Dev/Tests/assets/tests/asset_pak.lua` reads a built archive's contents back with `fs_list` and
`fs_read`, then cleans up with `fs_remove_all`.

## Engine content and asset paths

For `tests/engine_content*.lua`: what an asset reference resolves to, and the engine's own content
(`Engine/Content`, referenced as `engine:...`; see [Projects & Builds](../../../../07-projects-and-tools/01-projects-and-builds.md#engine-content)).

| Call | Answers |
|---|---|
| `asset_path(reference)` | The file a reference resolves to (`Project::GetAssetPath`): the project's, the engine's for an `engine:` reference or an old name the project has no file for, or the path it would have if there were one. |
| `to_asset_relative(path)` | What a saved scene would store for a path: asset-relative inside the project, `engine:...` under the engine content, else the input. |
| `legacy_alias(reference)` | The `engine:` reference an old name (`models/cube.obj`, `textures/checker.png` ...) stands for, or `""`. |
| `invalidate_asset_paths()` | Drops what path resolution remembers, as the editor does after it writes an asset. Call it after putting a file in the project behind the engine's back. |
| `asset_path_by_guid(guid)` | The path the asset-GUID index holds for a GUID, project and engine content alike, or `""`. |
| `engine_root()` | The folder holding `Engine/Content`, with forward slashes. |
| `find_engine_root(directory)` | The same search from a directory of the test's own making, and its parents; `""` when there is none. |
| `write_primitive_obj(shape, path)` / `write_png(path, width, height, rgba)` | For the content generator in `Dev/Tools/content`: a primitive as OBJ, and a PNG from raw RGBA8 bytes. Refused outside the engine tree and the temp directory. |
| `editor.command("asset_root" [, "project"\|"engine"])` and the other `asset_*` commands | Editor tests only. The Asset Browser's root and the operations its menus and drops call, each answering `ok` or `refused`: `asset_browser_dir`, `source_browser_dir`, `asset_copy` / `asset_cut` / `asset_paste <dir>`, `asset_move <src>\|<dir>`, `asset_rename <path>\|<name>`, `asset_delete <path>`, `asset_new_folder <dir>\|<name>`, `asset_duplicate <path>\|<dir>` (Duplicate to Project; answers the new path), `asset_reimport <path>`, `asset_favourite <dir>`, `asset_favourites`, `asset_recents`, `asset_select <path>`. `drop_asset <path>\|x\|y\|z` is a viewport or hierarchy drop, `drop_field <path>\|<ext,ext>` and `drop_texture <slot>\|<path>` are the asset-field and material-slot drops (they answer what was stored), `add_primitive <shape>` is Add > Primitive. `tests/asset_browser_engine_root.lua`, `drag_engine_asset.lua`, `add_primitive.lua`. |
| `run_game(exe, args, workDir [, timeoutSeconds])` | Runs another executable (a packaged build) to its end in `workDir` and returns its exit code, or `nil` when it was killed for running too long. `Dev/Tests/assets/tests/engine_content_packaging.lua` runs a packaged runtime with a test script of its own. |

## Mounting an archive

Direct access to the engine's virtual file system, for testing the `.Lpak` mechanism on its
own. A test can mount a real built archive and prove that a path present only in it still
resolves, without launching a second process running the packaged build.

| Call | Answers |
|---|---|
| `vfs_mount(pakPath, root)` | Whether the archive mounted. `root` is the path prefix the archive's entries are addressed under. |
| `vfs_resolve(path)` | What the VFS resolves the path to. A path that lives only in the archive comes back as the extracted temp file, not as the path that was asked for. |
| `vfs_unmount()` | Unmounts. Nothing to answer. |

```lua
expect(vfs_mount(output .. "/Game.Lpak", output), "mounted the real built archive")
local resolved = vfs_resolve(meshPath)
expect(resolved ~= meshPath, "the VFS found the path in the archive, not on disk")
vfs_unmount()
```

## Fonts and text layout

For `tests/text_layout.lua` and `tests/text_i18n.lua`: what `Font` bakes and lays out, exactly.

`font_probe_bake(primary [, options])` bakes a font of the test's own, CPU only: never uploaded
and never the HUD's. `primary` is asset-relative, absolute, or `engine:Fonts/...` for an engine
font. `options` takes `fallbacks` (a list of fonts), `language_fallbacks` (how many of those are
a text language's own, which draw the CJK full-width punctuation), `charset` (a string of extra
characters) or `charset_file`, `pixel_height`, `max_page_height` (small values force several
atlas pages) and `parallel` (false bakes on one core, for timing). It returns a probe, or `nil`
when the primary will not bake:

- `probe:layout(text [, size, wrap, spacing])` returns `width`, `height`, `line_height`,
  `glyphs`, `lines` (each line's width), `text` (each line's drawn characters, spaces left out),
  `faces` and `pages` (per placed glyph), `hash` (every placed glyph's exact float bits, line and
  the block size: equal hashes mean an identical layout) and `pixel_hash` (the atlas texels under
  every placed glyph: equal means the same fields were drawn, wherever they were packed).
- `probe:glyph(char or codepoint)` returns `face` (0 the primary, n the nth fallback), `page`,
  `advance`, `width`, `height`, `offset_x`, `offset_y`, or `nil` when nothing was baked for it.
- `probe:stats()` returns `pages`, `page_widths`, `page_heights`, `glyphs`, `fallback_glyphs`,
  `charset_requested`, `charset_baked`, `charset_missing`, `ms` and `atlas_bytes`.

`font_can_break(a, b)` answers whether a line may break between two characters with nothing
between them (`Font::CanBreakBetween`: the CJK rule and kinsoku).

The live HUD font: `ui_font_info()` (the faces, `pages`, glyph and charset counts, `language` and
`matched_language`, `bakes` and `uploads` since start-up, `descriptor_sets`), `ui_font_glyph(char)`
(`face`, the `file` it came from, `page`, `advance`), `ui_font_config()` and
`set_ui_font_config(table)` (the active project's `UIFont`, `UIFontFallbacks`,
`UIFontFallbacksByLanguage`, `UIFontCharset` and `UIFontCharsetByLanguage` as `font`,
`fallbacks`, `by_language`, `charset` and `charset_by_language`; nothing is saved, and a test
puts back what it read), `ui_font_manifest_roundtrip(table)` (the same table written by the
manifest's own writer and read back by its reader, `WriteUIFontSettings`/`ReadUIFontSettings`,
without saving a project: returns the YAML and the settings read back), and `gpu_allocations()`
(VMA's live allocation count and bytes, for asserting that a rebake leaked nothing).

`nine_slice_quads(x0, y0, x1, y1, l, t, r, b, tex_w, tex_h, pixels_per_texel, fill [, draw_center [, max_quads]])`
runs the HUD's 9-slice expansion (`Engine/Rendering/NineSlice.h`) with no texture and no frame: the
rect, the borders in texels, the texture size, and `fill` as `"stretch"`, `"tile"` or `"tile_fit"`. It
returns the quads in paint order as `{ x0, y0, x1, y1, u0, v0, u1, v1 }` (UVs top-down over the whole
texture), an empty list for a rect with no area, or `nil` when it needs more than `max_quads` (default
1024), which is when the HUD draws the sprite stretched. It is a plain global rather than an `editor.`
hook because the math is Engine code the runtime has too, so a runtime test can use it beside the pixel
readbacks (`tests/ui_nine_slice_math.lua`).

`set_ui_focus_navigation(on [, wrap])` turns the project's UI focus navigation on or off for the run (the
manifest on disk is untouched; turning it on adds the missing `ui_*` input actions, as loading a project with it on
does), `project_has_input_action(name)` says whether the project has an action of that name, and
`ui_focus_pick(from, direction, candidates [, wrap])` is the focus picking math (`Scene/UIFocusMath.h`) with no
scene: rects are `{ x0, y0, x1, y1 }` tables, `direction` is `"up"`, `"down"`, `"left"` or `"right"`, and the result
is the 1-based index of the candidate or `nil` (`tests/ui_focus_nav.lua`).

`input.inject_text(text)` and `input.inject_key(key [, mods [, mode]])` type for a test, through the same ImGui event queue a
keyboard fills, so a HUD text field, the key bindings and the editor's shortcuts see exactly what a player's hands would send:
`inject_text("h\xC3\xA9llo")` types those characters (UTF-8); `inject_key("Backspace")` taps a key, `inject_key("Left", "ctrl+shift")`
taps it with modifiers (`ctrl`, `shift`, `alt`), and a third argument of `1` presses and holds it, `2` releases it. Key names are ImGui's
(`"Backspace"`, `"A"`, `"Home"`, `"Enter"`, `"Space"`) plus `"Left"`, `"Right"`, `"Up"`, `"Down"`, `"Esc"`, `"Del"`, `"Return"`; the call
answers `false` for a name that is no key. What is injected is read when the next frame starts, which is just after a script's
`on_update`, and a tap's press and release land in separate frames, so two taps of the same key want a frame between them.

The text field's probes (`tests/ui_textfield.lua`): `ui_text_info(field)` answers `{ editing, caret, anchor, scroll, caret_visible, shown,
selected }` (`shown` is what is drawn, the mask of a password), `ui_text_caret_x(field, index)` where the caret sits before that
character in HUD pixels, `ui_text_caret_blink(false)` turns the blink off so the caret can be read from pixels, `ui_text_test_clipboard(on [, text])`
swaps the system clipboard for a string (so a test never writes over what a person copied) and `ui_text_clipboard()` reads either,
`ui_text_wants_input()` is whether the game's keys are being told a text field has the keyboard, `ui_text_test_action(name, key)` puts an
action bound to one key into the running project's table (in memory), and `ui_text_force_keyboard(on)` / `ui_text_keyboard()` /
`ui_text_dismiss_keyboard()` stand in for the Steam Deck's floating keyboard. `count_pixels(u0, v0, u1, v1, r0, g0, b0, r1, g1, b1)` counts the
pixels of the frame in a rectangle (0..1 across the target) whose channels all lie in the given inclusive ranges: one readback for a
region, for "is there any text here". `frame_size()` is the width and height of that same frame (the viewport panel's size in an editor run), `0, 0` when there is nothing to read. `project_build_options_read(yaml)` and `project_build_options_write({ demo, steam, steam_app_id, demo_app_id, require_ownership, website, input_manifest })` put the build target through the manifest's own reader and writer (`ReadBuildTargetSettings`, `WriteBuildTargetFlags`, `WriteSteamSettings`) without saving, and answer the table read or the YAML written. `ui_focus_manifest_roundtrip({ navigation, wrap, ring_width, ring_gap, ring_radius, ring_color })`
puts the project's focus settings through the manifest's own writer and reader and returns the YAML and what was read back
(`tests/ui_focus_nav.lua`).

## Counters

The readers below count what the renderer did to the last frame. They are the proof that a
subsystem ran at all, and they are needed exactly because their failure mode is invisible: a
sort that stopped sorting, a bind check that stopped matching, a cache that stopped sharing
all look identical in the picture and cost work nobody was counting.

Most of these describe the frame just recorded, and each answers a plain number.
`mesh_cache_count` is the exception: it counts a loader cache rather than a frame. Where a
reading needs a renderer that has not run yet, the answer is `-1` rather than zero, so a test
can tell "nothing happened" apart from "nothing has rendered".

The general counters (`draw_calls`, `mesh_cache_count`, `shared_mesh_count`, `culled_count`,
`binds_recorded`, `binds_skipped`) are listed in the harness chapter's own table as well.
Their assertions:

| Call | Answers |
|---|---|
| `draw_calls()` | Draw calls issued last frame. A scene of six entities sharing two meshes should read as two instanced draws plus the deferred lighting pass. |
| `binds_recorded()` | Mesh bind commands recorded last frame, where the GPU-scene descriptors are bound once and only mesh buffers change per batch. |
| `binds_skipped()` | Binds the redundant-bind check elided. Zero on a scene of repeated materials means that check is not doing anything. |
| `mesh_cache_count()` | Distinct models parsed. Five entities sharing a mesh should be one parse, and on a large OBJ the difference between one and five is measured in seconds. |
| `shared_mesh_count()` | Distinct models holding GPU buffers, so a count equal to the entity count means the sharing is not happening. |
| `culled_count()` | Meshes the frustum test skipped last frame. Zero on a scene with geometry behind the camera means the test is not running. |

The material-graph counters are for the shader-graph fast path and for batching, and are
read as a comparison rather than on their own: with instancing on, N entities sharing a graph
and a mesh should read as one draw call and N instances, and N draw calls with it forced off.

| Call | Answers |
|---|---|
| `material_graph_draw_calls()` | Draw calls the last opaque material-graph pass issued, after grouping draws that share a pipeline, mesh and index count into instanced draws. `-1` with no renderer. |
| `material_graph_instance_count()` | The instances those draw calls cover. `-1` with no renderer. |
| `material_graph_pipeline_count()` | Distinct material-graph pipelines compiled so far, opaque plus transparent. A `.Lmaterial` that only names the built-in default-PBR graph should not add one, while a genuinely custom graph adds its own. |

The ray-tracing structure readers live here rather than on the diagnostics page, since they
are the same kind of "did it happen" number. All of them answer `0` (or `false`) with no
renderer, so a test should gate on `ray_tracing_available()` first and treat the counts as
evidence only once the device has said it has ray query.

| Call | Answers |
|---|---|
| `tlas_ready()` | Whether the top-level acceleration structure exists and is usable. |
| `tlas_instance_count()` | Instances the TLAS holds. |
| `blas_count()` | Cached bottom-level structures. |
| `blas_compacted_count()` | How many of those have been compacted. |
| `denoised_signal_count()` | How many signals the denoiser filtered in the frame just recorded, with shadow and occlusion counted once even when occlusion is traced at half resolution as its own chain. Zero means it never ran at all, which is a different fault from a frame it ran over and changed nothing. Read it per frame: by the time a readback has waited for the device, the renderer has moved on. |
| `acceleration_structure_bytes()` | Total device memory the acceleration structures hold. |

## What is stable here, and what is not

The assertion pair, the pixel and screenshot reads, the validation, script and entity counts,
the colour and projection helpers, the environment and filesystem helpers, and `require` and
`print` are the harness's stable API. Use them in any test you write, and expect them to keep
answering the same way.

The rest are diagnostics that exist to prove one subsystem ran: the bind, draw and cull
counters, the mesh cache counts, the material-graph trio, `denoised_signal_count` and the
acceleration-structure readers. They are safe to read, and their exact values belong to a
particular fixture rather than to a contract, so assert them against a scene you know rather
than as a general rule. None of them should be reached for from a game script, and none of
them are reachable in a play session that is not a test.
