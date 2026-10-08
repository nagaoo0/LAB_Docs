---
title: "The editor API"
---

The `editor` table: the calls a Lua test uses to drive the editor's panels and its open document.

See [Testing](../../../../07-projects-and-tools/02-testing.md) and [Lua Scripting](../../index.md).

## What the table is

An editor test is a test whose header carries `-- editor`. Only that host is given the table: the editor wires one function per call into `EditorHooks`, and `ScriptEngine` creates `editor` only when a host supplied it. The runtime has an empty hook set and therefore no `editor` table at all, so a gameplay script or a scene test that reaches for `editor` gets a nil index and says so rather than silently doing nothing. Nothing on this page is gameplay API, and none of it ships.

The table is created from the editor's own layer, which means every call operates on **whatever the editor currently has open**, not on the scene the test started with. That is the point of most of it: the editor UI is where the crashes were, and the transitions between documents are exactly the ones a scene test cannot reach. Opening an object replaces the scene, and the hierarchy row that took the editor down only draws for an instance root, which is why the undo and gesture hooks exist too. The editor's tests under `Dev/Tests/assets/tests/` are the working examples (`altdrag.lua`, `skeleton_overlay.lua`, `assetbrowser.lua`).

Three conventions hold across the whole table:

- A call whose hook the host did not supply answers a neutral value rather than crashing on an empty function: `false`, `0`, `""`, an empty table or `vec3(0, 0, 0)`. Where a call did run, its answer describes what happened. Calls whose work is an action answer nothing at all.
- A call that takes an entity names it the way the Hierarchy panel shows it, by tag. Names are not unique, and a lookup answers the first match, so use them to reach a well-known entity, not to identify one.
- **Play mode refuses edits.** `open_object` logs `Stop play mode before editing an object` and does nothing; `set_hidden`, `set_locked`, `set_mesh_path` and `set_material_color` return without touching anything; `gizmo_duplicate` answers `false`. `play()` and `stop()` are the door in and out, and everything else that only reads keeps working while the game runs.

The suite gives an editor test at least 240 frames, because a cold asset load through the editor's own event path can otherwise lose the race with a shorter run. See [Testing](../../../../07-projects-and-tools/02-testing.md).

## The `scene` trap

**`scene` binds to the scene that existed when the test loaded. Opening an object replaces that scene with the object's own, so inside the object editor `scene.find` searches a scene that is no longer open.** It can return a same-named entity from the old scene or nothing at all, and it will never return what you are looking at. `editor.open_scene(path)` has the same effect.

Everything under `editor` goes through the editor's layer, so it follows what is actually open. `editor.selected_position()` and `editor.set_selected_position(x, y, z)` exist for exactly this reason, and any other read or edit of the open document has to go through `editor.*` as well. The same trap is why an editor test asserts on the outcome through `editor.selected_*` and the panels rather than through `scene`.

```lua
-- scene: scenes/editor_edit_test.Lscene
-- editor

function on_finish()
    expect(editor.select("Cube"), "the cube is in the hierarchy")
    expect_near(editor.selected_position().z, 0.0, 0.001, "the cube sits on the ground")

    editor.set_selected_position(0.0, 0.0, 2.0)
    expect_near(editor.selected_position().z, 2.0, 0.001, "the move landed")

    expect(editor.open_object("objects/crate.Lobj"), "the crate opens in the object editor")
    editor.set_selected_position(1.0, 0.0, 0.0)
    expect(editor.save_object(), "the moved crate saves")
    editor.close_object()
    expect(editor.is_editing_object() == false, "closing restores the scene")
end
```

## Objects and scenes

Opening a document replaces the editor's scene, which is the trap above and the reason these calls are the first group on the page. An object is edited one at a time: opening one while another is open closes the first, because nested object editing would need a stack of stashed scenes and an object that contains another object is a reference the format does not have.

| Call | What it is |
|---|---|
| `open_object(path)` | Opens a `.Lobj`, asset-relative, in the object editor. Answers whether that object is now open. Does nothing while playing, with a log line. When another object with unsaved edits is open it raises the "Save changes before continuing?" prompt and answers `false` until the prompt is answered (see `transition_prompt_choose`). |
| `save_object()` | Writes the open object back to its file and refreshes the copies already placed in the scene, so saving is visible without re-instantiating anything. `false` when no object is open; otherwise `true`, so a test that cares whether the write landed reads the file or the log. A save whose root entity has been deleted logs an error and writes nothing, and still answers `true`. |
| `close_object(discard)` | Leaves the object editor and restores the scene it was opened from. Answers `true` once the object is closed. An object with unsaved edits raises the "Save changes before continuing?" prompt and answers `false`, unless `discard` is `true`, which drops the edits. |
| `transition_prompt_open()` | Whether that prompt is up. It also appears when `open_scene`, a new scene or a project change would replace a scene or an open object with unsaved edits. |
| `transition_prompt_choose(choice)` | Answers the prompt with `"save"`, `"discard"` or `"cancel"`. Answers `ok, error`; `ok` is `false` when no prompt is up, the choice is unknown or the save failed (the prompt then stays up). |
| `is_editing_object()` | Whether the object editor is the open document. |
| `open_scene(path)` | Opens a `.Lscene`, asset-relative. `false` when the file is not there. The work happens at the top of the next frame, so a test checks what is open a frame or more after asking. |
| `save_scene_as(path)` | Serializes the open scene to a project-relative path. The serializer does not report, so the answer is whether the file exists afterwards. |
| `open_graph(path)` | Opens a `.Lgraph` in the node graph editor. Answers whether the panel is open. |

## Selection and picking

The selection is the editor's own, the one the Hierarchy panel draws and the gizmo acts on. These calls build it by name and by viewport gesture, and report what is in it. `selected_position` and `set_selected_position` are the pair to reach for inside the object editor, where `scene` is the wrong search.

| Call | What it is |
|---|---|
| `select(name)` | Makes the named entity the selection, replacing whatever was selected. `false` when no entity carries that name. |
| `add_to_selection(name)` | Adds it to the selection instead of replacing it, the way ctrl-click does. Same answer. |
| `clear_selection()` | Empties the selection. Answers nothing. |
| `select_all()` | Selects everything. Answers nothing. |
| `selection_count()` | How many entities are selected. |
| `selected_name()` | The primary selection's name, or `""`. |
| `selected_id()` | The primary selection's entity id, the UUID a scene file stores, or `0`. |
| `selected_position()` / `set_selected_position(x, y, z)` | The primary selection's **local** position, and a way to set it. The setter is a no-op on a locked entity, and answers nothing. |
| `frame_selection()` | Frames the selection in the viewport. Answers whether anything had bounds to frame; whether the view actually ended up on it is a question for the pixels. |
| `pick(u, v [, additive])` | Viewport picking at a point in 0..1 viewport fractions: the ray, the intersection, and what it does to the selection. Answers the name of the entity hit, or `""` for a miss. `additive` adds to the selection rather than replacing it. |
| `select_in_rect(u0, v0, u1, v1 [, additive])` | Rubber-band selection over a rectangle in viewport fractions. Answers how many entities ended up selected. |

## Undo, duplicate and delete

The editor's own history and the two destructive gestures, which a test drives directly rather than through ImGui.

| Call | What it is |
|---|---|
| `undo()` / `redo()` | One step of the editor's history. Each answers whether there was a step to take. |
| `delete_selected()` | Deletes the selection. `false` when nothing is selected. |
| `duplicate_selected()` | Duplicates the selection. `false` when nothing is selected. |
| `gizmo_duplicate()` | Alt-dragging the gizmo without the drag: duplicates the selection in place and leaves the copies selected. `false` while playing or with nothing selected. The rest of that gesture is ImGuizmo's internal drag state, which no test can reach. |

## Hierarchy state

The per-row eye and lock buttons, set directly. Both are keyed on an entity name and are no-ops for an unknown name.

| Call | What it is |
|---|---|
| `set_hidden(name, hidden)` | The row's visibility state, what its eye button toggles. No-op in play mode. Answers nothing. |
| `set_locked(name, locked)` | The row's lock. A locked entity, and any entity under a locked ancestor, is skipped by the editor's own edits, `set_selected_position` included. |

## Properties and components

The inspector's own writes, for tests that need a component or a field to exist without a prefab built around it. These go through the same component table and the same capture and commit cycle the Properties panel uses, so what they exercise is the real path, not a back door into the registry.

| Call | What it is |
|---|---|
| `add_component(entityName, componentName)` | Adds a component by the name the Add Component menu shows, read from the same table the menu's items use, so the two cannot disagree about a name. Undoable. `false` for an unknown component or an entity that is not there. |
| `set_mesh_path(name, path)` | Points a named entity's `MeshComponent` at a path, adding the component and loading it the way the inspector's mesh-path field does. Answers nothing. |
| `get_mesh_path(name)` | The path it actually resolved to, which can differ from what was set: the asset-GUID fallback rewrites the stored path when the original stops resolving. `""` when there is no such component. |
| `is_mesh_loading(name)` | Whether that entity's mesh load is still in flight. |
| `set_material_color(name, r, g, b, a)` | Sets the entity's `MaterialComponent` colour through a real Capture, mutate, Commit cycle, so prefab-override tracking is exercised exactly as a properties-panel widget edit would. Emplaces a material component on an entity that has none. Answers nothing. |
| `get_material_color(name)` | `{r, g, b, a}` from the component, zeroes when the entity has none. |

## Native behaviour modules

The play-time `native_*` readers go through `ScriptEngine`'s bound scene, which only exists once play has started, so an editor test cannot use them. These follow whatever scene is actually open, like every other editor command, and they ask the editor's own loaded module rather than the runtime's. See [Building and loading a module](../../../03-cpp/api/module/03-building-and-loading.md).

| Call | What it is |
|---|---|
| `load_native_module(name)` | Loads the project's native module before play. The path resolves the way the project setting does: the open project's own directory first, then beside the executable. `false` when it could not be loaded. In the editor nothing is instantiated, so the registry only becomes readable; in play it also re-runs every `NativeScriptComponent`'s `on_create` against the freshly loaded module. |
| `is_native_module_loaded()` | Whether a module is loaded. |
| `native_behaviour_count()` | How many behaviour instances exist. |
| `native_classes()` | The names the loaded module registered, 1-indexed. This is what makes an unknown class answerable rather than just reported. |
| `native_field_defs(behaviour)` | One row per declared field: `behaviour`, `name` and `type`. Definitions, so this answers in the editor, where nothing is instantiated. |
| `set_native_field(entityName, behaviour, field, value)` | Writes one declared field, the way the inspector's own row does. `false` when the entity or its native script is not there, or the value did not land. No reader comes with it: what a test should check is that the value survives a save, which the scene text answers. |
| `add_native_class(entityName, className)` | Appends a class name to an entity's native script, as the inspector's Add row does, and removes nothing. `false` when the name was not added, which covers empty and already on the entity. |

## Source control

The Git panel, read and driven. The status snapshot comes from the editor's own poller, so `git_status` answers what the panel is showing.

| Call | What it is |
|---|---|
| `git_status()` | A table: `is_repo`, `git_available`, `branch`, `changes` (unstaged), `staged`, `lfs_patterns`, `busy`. |
| `git_refresh()` | Asks for a status refresh. Answers nothing. |
| `git_init(lfs)` | Initialises a repository in the project, with LFS as well when `lfs` is set. `false` when no project is open. |
| `git_stage_all()` | Stages everything. Fire and forget: the answer only says whether a repository was there to ask. |
| `git_commit(message)` | Commits. `false` for an empty message or when there is no repository, and otherwise the same fire-and-forget answer. |

## Animation graph editor

The animation graph editor's canvas is authored entirely with the mouse, which no test can drive. What those gestures produce is plain data, though, and that is where the bugs were: a node added but parked at `FLT_MAX` by the position readback, a condition graph with no Result node to wire a rule into. These calls produce the same data, either immediately or through a queued add that the panel applies inside its own canvas, at the point a context menu adds. See [Animation](../../../../04-gameplay/02-animation.md).

| Call | What it is |
|---|---|
| `open_anim_graph(path)` | Opens a `.Lanimgraph`. Answers whether the panel is now open. |
| `anim_graph_add_state(x, y)` | Adds a state immediately, from outside the panel's render. Answers its index, `-1` on failure. |
| `anim_graph_add_pose_node(stateIndex, nodeType, x, y)` | Adds a pose node to a state's pose graph. Answers its id, `-1` on failure. |
| `anim_graph_add_transition(from, to)` | Adds a transition between two states. Answers its index, `-1` on failure. |
| `anim_graph_state_position(index)` / `anim_graph_pose_node_position(stateIndex, nodeId)` | Where a node sits on the canvas: `vec3(x, y, 1)` when it exists, `vec3(0, 0, 0)` when it does not. |
| `anim_graph_condition_has_result(index)` | Whether a transition's condition graph has a Result node to wire a rule into. |
| `anim_graph_state_count()` | How many states the open graph has, `-1` when there is nothing open. |
| `anim_graph_queue_add_state(x, y)` / `anim_graph_queue_add_pose_node(stateIndex, nodeType, x, y)` | Queue an add that the panel applies inside its own canvas. Read the result with the `last_added` calls a few frames later; these answer nothing. |
| `anim_graph_last_added_state()` / `anim_graph_last_added_pose_node()` | The index or id the queued add produced once the panel applied it, `-1` before that. |
| `anim_graph_open_state(index)` | Enters a state's pose graph, as double-clicking it does. A pose graph only renders while it is the open level, so this has to come before asserting on pose nodes. Answers nothing. |

## Material instance editor

The Material Instance Editor (MaterialInstancePanel), driven from a test for the same reason as the graph editor: the widgets themselves are not scriptable, and the override setter is the seam. See [Materials & Textures](../../../../03-rendering/01-materials.md) for the asset itself.

| Call | What it is |
|---|---|
| `open_material_instance(path)` | Opens a `.Lmaterial`. Answers whether the panel is now open. |
| `material_instance_parameter_count()` | The parent graph's overridable Scalar, Vector3 and Texture Parameters plus Static Switches, after the parent compile. `-1` when the hook is not available. |
| `set_material_instance_override(name, r, g, b, a)` | Sets a Scalar, Vector3 or Static Switch override by parameter name (`r`, `g` and `b` used per type, `a` unused), which mirrors ticking a parameter's checkbox on and dragging its value, in one call. Not for Texture Parameters. `false` when no instance is open. |
| `save_material_instance()` | Saves the open instance. Answers whether it is no longer dirty. |

## AI state tree panel

The gameplay AI system's own editor. It opens by entity name rather than by path, because an AI behaviour tree is inline component data: there is no file for the panel to open.

| Call | What it is |
|---|---|
| `open_ai_state_tree(entityName)` | Opens the panel for an entity that carries an `AIControllerComponent`. `false` when the entity or the component is not there. |
| `ai_state_tree_panel_open()` | Whether that panel is open. |

## Keyboard owners

The document panels (Node Graph, Material Graph, Material Instance, Animation Graph, AI State Tree) handle their own undo and save keys, and the editor's scene shortcuts stand down while one has focus.

| Call | What it is |
|---|---|
| `keyboard_owners()` | A list of `{name, window}`, one per such panel. `window` is what `editor.command("focus_window", window)` takes. |
| `keyboard_owner()` | The name of the panel that has the keys now, or `""`. |
| `open_material_graph(path)` | Opens a `.Lshader` in the Material Graph. Answers whether the panel is now open. |
| `history_revision()` | The scene history's revision. An undo or a new edit moves it; a key the scene did not act on leaves it. |

## Viewport

The view the test asserts on: where the camera is, what the frame was rendered at, what is drawn over it, and the ways to get a picture out. A test that asserts on what is where on screen sets the camera first, so the view is known.

| Call | What it is |
|---|---|
| `set_camera(x, y, z, pitch, yaw)` | Places the editor camera. Position plus pitch and yaw in **radians**, which is what the camera stores. Answers nothing. |
| `set_render_size(width, height)` | Pins the render resolution regardless of the viewport panel's size, for a screenshot at an exact size. `0x0` hands it back to the panel. |
| `set_exposure(exposure)` | The viewport's exposure: the editor camera's in edit mode, the primary scene camera's in Play. Clamped to 0.01 through 32. Answers nothing. |
| `set_outline(width, r, g, b)` | Selection outline width in pixels and colour. A test needs a fat, unmistakable ring, because the shipped 2.5 pixels is three thousandths of the frame's height, which a sampled point can pass straight through without ever landing on. Deliberately not saved, so a test's own outline does not end up in the user's preferences. |
| `set_overlay(name, enabled)` | A viewport overlay by name: `"grid"`, `"colliders"`, `"skeletons"` or `"ddgi"`. `false` for a name the viewport has no overlay for. |
| `set_viewport_fullscreen(enabled)` | Hides or restores every docked panel except the viewport itself, the same state F10, Fullscreen Viewport, drives. It is how a performance probe isolates how much of a frame's CPU cost is panel chrome. Answers nothing. |
| `reset_layout()` | Windows > Reset Layout: rebuilds the default dock layout and shows the four core panels again. Takes effect on the next frame. Answers nothing. |
| `dock_info(title)` | Where a window sits in the dock layout, as a table: `found`, `docked`, `visible` (false for a tab that is not the selected one), `node` (the dock node id, 0 when floating), `x`, `y`, `w`, `h`, and `display_w`, `display_h` for the editor window. `found` is false for a window that has not been created yet. |
| `render_split_ms()` | `OnUIRender`'s own split from the last frame, as a table: `render_engine_ms`, `render_ui_ms`, `shadow_pass_ms`, `prepare_runtime_ms`, `begin_frame_ms`, `render_prepared_ms`, `lighting_pass_ms`, `end_frame_ms`. `-1` for anything the host did not supply. |
| `debug_line_count()` | How many debug lines the last frame's overlays drew, the one number that tells a skeleton overlay that drew from one that found no skeleton, since a few thin lines barely move a pixel average. |
| `screenshot()` | What F12 does: reads the viewport back, converts it to RGB and either hands it to Steam or writes a PNG. `false` when there was nothing rendered to read. |
| `start_video_export(width, height, fps, duration [, backend [, split_seconds]])` | Export Video, driven from a script rather than the modal. `split_seconds` starts a new file every that many seconds inside the one export. Always fixed-step, so a test gets a deterministic frame count rather than one that depends on how long the machine took per frame. `backend` is `"auto"`, `"ffmpeg"`, `"ffmpeg_hq"`, `"png"` (or `"pngsequence"`) or `"avi"`, case-insensitive; empty means auto. Answers whether the export started. |
| `video_export_active()` | Whether an export is running. |
| `video_stats()` | `captured`, `dropped`, `elapsed` seconds and `outputs`, the list of files or directories opened, for the running or last export. |
| `last_video_output()` | The file or directory the backend resolved to once it opened. |

## UI preview

The UI Designer's render target: the HUD alone, drawn at a design resolution of its own into an offscreen image the editor viewport does not share, through the same pipeline, shaders and draw list as the game's HUD. These calls drive it without the panel; while the UI Designer panel is open and on screen it is the panel that decides the size, the backdrop and the overrides, and these calls wait their turn (see the next section). A test sets the size, waits for it to be ready, and reads pixels back. The UI scale follows the preview's height exactly as it follows the viewport's, so a project with a `UIReferenceHeight` scales its layout in the preview too.

The preview is rendered by the frame driver every frame while it is requested. A resize is built at the top of the next frame and drawn in the one after, so after `ui_preview_render` with a new size, poll `ui_preview_ready()` (or compare `ui_preview_size()`) rather than reading on the next frame. It is a session state of the editor, never saved.

| Call | What it is |
|---|---|
| `ui_preview_render(width, height [, r, g, b, a])` | Starts the preview, or resizes it. The clear colour is 0..1 per channel and defaults to a dark grey; it is always drawn opaque, whatever `a` says. Answers nothing. |
| `ui_preview_ready()` | Whether the preview exists at the size last asked for and a frame has been recorded into it. `false` again after a resize until the new image is drawn, and after `ui_preview_stop()`. |
| `ui_preview_size()` | The size of the image that exists, `width, height`; `0, 0` before the first. Lags a resize by a frame. |
| `ui_preview_pixel(x, y)` | `r, g, b, a`, 0 to 255 like `pixel()`, of one preview pixel (top-left origin). All zeroes when it is not ready or the point is outside. Waits for the device, so call it a handful of times, never per frame. |
| `ui_preview_force(name, state)` | Overrides one element's visibility in the preview only: `true` draws it although its own `Visible` is off, `false` hides it and everything Relative under it, `nil` returns it to what is authored. Only the element's own flag is replaced: it still inherits from its parent, so forcing a child on under a hidden parent needs the parent forced on too. The scene is never written, so `entity:is_ui_visible()` still answers what the scene file says. `false` for a name nothing carries. |
| `ui_pick(x, y [, includeNonInteractive])` | The names of the elements under a point in preview pixels, topmost first: higher layer, then deeper in the Relative chain, then later in registry order (the rule the game's pointer uses). Uses the preview's own size and overrides, so it lists what is drawn. Elements with `Interactive` off are left out unless `includeNonInteractive` is true. An empty table when nothing is there or the preview is not running. |
| `ui_preview_stop()` | Stops rendering the preview and forgets its size and overrides. The image is kept until shutdown or the next size change. |

The hierarchy eye applies in the preview as it does in the viewport: an element hidden there is not drawn or picked here, and wins over `ui_preview_force(name, true)`. Play mode ignores it, in both. `tests/ui_preview.lua` is the working example.

## UI Designer panel

The dockable **UI Designer** panel (Windows menu): the preview above on a pannable, zoomable canvas, a resolution and backdrop toolbar, and a list of the HUD's screens (every UI root: an entity with a `UITransformComponent` that is not Relative, or whose parent has none). A screen's eye is a preview-only override (as authored, hidden, shown), its radio isolates it (every other screen hidden), and "Follow selection" isolates the screen of whatever is selected. Clicking the canvas selects the topmost element under the cursor, the non-interactive ones too; clicking empty canvas clears the selection.

The canvas also edits. Everything is done in canvas pixels and solved back to the element's own fields with `UILayoutMath::SolveForRect`, which keeps the anchors and the pivot and rewrites only `Offset`, `OffsetMax` and `Size` (a stretched axis takes `Offset` / `OffsetMax`, a point axis `Offset` / `Size`):

- **Click** selects the topmost element; **Ctrl or Shift-click** toggles one in the selection; **Alt-click** steps down through everything under the cursor and wraps. A click over the body of an already selected element keeps the selection, so an element under another can still be dragged; a click on a child of the selected element selects the child.
- **Marquee**: dragging from empty canvas selects what is fully inside (**Alt**: anything it touches; Ctrl/Shift adds). A root screen only counts when it is wholly inside.
- **Move**: drag the body of a selected element; every editable selected element moves by the same pixel delta, each solved against its own parent. An element whose Relative chain reaches a selected ancestor is left to that ancestor. **Shift** keeps to the axis the pointer has gone further along.
- **Resize**: drag one of the eight handles on the primary selection. **Shift** keeps the aspect ratio, **Alt** resizes about the centre. An axis stops at one UI unit and never flips.
- **Arrow keys** nudge the selection one UI unit, ten with Shift.
- **Delete** with the panel focused deletes the selection, as in the viewport (and Ctrl+D duplicates it, as everywhere). **Esc** during a drag puts every field back and records nothing. Ctrl+Z pressed during a drag ends it first, so it is the drag that is undone.
- Each gesture is one undo step (`Move UI`, `Resize UI`, `Nudge UI`), however many frames it took. A gesture that ends where it began records nothing.
- Nothing is edited while the game is playing or on an element locked in the hierarchy: the drag is refused with a note at the cursor, and selection still works. **Ctrl+A** with the panel focused selects the UI elements beside the selected one (every screen when nothing is selected) rather than the whole scene.

**Snapping.** The toolbar's *Smart guides*, *Grid* (with its size in UI units) and *Pixel snap* are saved with the panel's other preferences. While a move or a resize is dragged:

- **Smart guides** snap to the edges and centres of the element's siblings (the elements sharing its UI parent that the preview draws, leaving out everything being edited and what hangs under it), of its parent's rect, and of the canvas. A move looks at the moving rect's left, centre and right (top, middle, bottom), a resize only at the edge the handle drags, and the smallest correction within 6 *screen* pixels (so the reach in canvas pixels is 6 over the zoom) wins. The selection moves as one rect. Magenta lines run from the moving rect to what it matched.
- **Equal spacing**: the gap to the neighbour on a side (the nearest sibling that shares some of its extent across) snaps to a gap that already exists between two siblings, and a move between two neighbours makes the two gaps equal; a line beats spacing on a tie. Gaps to the neighbours are measured in UI units whenever smart guides are on, in magenta when they are equal to something.
- **Grid** snaps the top-left corner (a resize: the edge it drags) to multiples of the grid size counted from the canvas's corner; it is drawn over the canvas once its lines are 4 screen pixels apart. **Pixel snap** rounds the same points to whole UI units counted from the parent's corner. Smart guides go first within reach; else the grid; else whole units.
- Holding **Ctrl** through the drag turns all of it off. Shift-resizing (the aspect is the drag's) snaps nothing, and a Shift-locked axis of a move is never snapped. Arrow-key nudges ignore guides and the grid but respect Pixel snap.

**Anchors and pivot.** The primary selection's four anchor petals (right triangles at the corners of its anchor rect, inside the parent rect) and its pivot circle are handles. Dragging a petal moves that corner of the anchor rect -- top-left is `AnchorMin.x` and `.y`, top-right `AnchorMax.x` and `AnchorMin.y`, and so on -- while the element stays exactly where it is drawn (`UILayoutMath::SetAnchorsKeepingRect`: only the anchors and the offsets change). Anchors clamp to 0..1 and keep `min <= max` per axis: an edge dragged across the other drags it along, as the inspector does. The four petals of a point anchor are one place: dragging it moves the point. With smart guides on the anchors snap to 0, 0.25, 0.5, 0.75 and 1 of the parent and to the element's own edges and centre (within 6 screen pixels); Ctrl turns that off. Dragging the pivot circle changes `Pivot` and re-solves `Offset` so the rect stays; it snaps to 0, 0.5 and 1 of the element, and Alt drags it freely. Only a point axis has a pivot (a stretched axis ignores it, and the circle is hidden when both are stretched). The pointer prefers resize handles, then the pivot, then the petals, then the body; petals and the pivot show a hand cursor. A readout follows the pointer ("Anchor 0.25, 0.00 - 1.00, 0.00", "Pivot 0.50, 0.00"). Each drag is one undo step ("Move anchors", "Move pivot"), Esc puts it back, and like every edit it is refused in Play and on locked elements.

Holding **V** over the canvas (Godot's pivot key) takes the pivot from anywhere over the primary's rect: a press there puts the pivot at the pointer and, dragged, it follows it (same snapping to 0, 0.5, 1; Alt is free). That is how a pivot sitting under a resize handle -- the default (0, 0) is the NW handle -- is reached. The pivot circle gets a white ring when it coincides with a handle.

**Create palette.** A collapsible pane on the canvas's right (its state is saved with the panel's preferences) lists what can be made: the primitives Panel (a rounded-12 sprite, 200 x 120), Label (text "Label", 32 px, 240 x 48), Image (a sprite with an empty texture slot, 128 x 128) and Empty (a UI Transform that fills its parent, Full Rect), and under *Prefabs* every `.Lobj` anywhere under the project's `assets/objects/` whose file contains a `UITransformComponent` (rescanned with the button). Drag an item onto the canvas, or double-click it: it is created as a Relative child of the element under the drop point (a double-click: the selected element, else the isolated screen, else top level), with its top-left at that point (`SolveForRect`; an axis that stretches keeps its anchors), under a unique name ("Panel", "Panel 2", ...), selected, as one undo step that removes the whole tree and redoes it with its parent. Prefabs are instantiated through `Prefab::Instantiate`, so they stay linked to their `.Lobj`. Nothing is created in Play or inside a locked element. Under *Widgets* the palette also makes a **Button** (sprite, a *Label* child, a UI Button), **Toggle** (a box, a *Checkmark* child wired as its Graphic, a *Label*), **Slider** (a track, *Fill* and *Handle* children wired to it, value 0.5), **HBox**, **VBox** and **Grid** (a UI Layout over a background sprite) and a **9-Slice Panel** (a sprite with no texture, which opens the 9-Slice pane on an empty slot); each widget's children are named after it ("Slider 2 Fill") and the references between them are entity ids, which undo and redo keep. A child made inside a container takes the next `UILayoutElementComponent` Order. New kinds are one more `PaletteItem` in `UIDesignerPanel::GetPaletteItems` (see the marked extension point). The whole panel is described in [UI Designer](../../../../05-ui/02-ui-designer.md).

**9-Slice tab.** The right-hand column has **Create | 9-Slice** tabs while the primary selection is a sprite with a texture (or sliced, or a just-made 9-Slice Panel); the panel picks 9-Slice by itself for a sliced sprite until a tab has been clicked. It holds the texture at a whole-number zoom with four draggable margin lines and the L / T / R / B, Scale, Fill and Centre fields. A line moves in whole texels and stops at the opposite one; a drag or a field edit is one undo step (`Edit 9-Slice`), `Esc` puts it back, and it is refused in Play and on locked elements. The selected sliced sprite also shows its cell lines, dashed, on the canvas.

**Placed and driven elements.** A child a layout container places, and a slider's Fill and Handle and a toggle's Graphic, are drawn with a padlock badge, have no resize handles, anchor petals or pivot, and are not moved by a nudge or a preset (a press on one still selects it); the tooltip says *Placed by <container>* or *Driven by <widget>*. A drag on a container's child **reorders** it, or -- 24 screen pixels outside the container -- re-parents it (*Reorder*, *Reparent*, one undo step each); a selected container has padding bars, spacing grips and, for a Grid, a columns chip, and a Fit Content axis has no resize handle. The chapter [UI Designer](../../../../05-ui/02-ui-designer.md#layout-containers) describes them.

The UI Designer opens, the first time it has no saved layout entry, as a tab of the dock node the 3D viewport is in; a saved layout (docked elsewhere, or floating) is never overridden.

The toolbar's anchor button (the preset picker the inspector has) applies a preset and mode to every editable selected element as one undo step ("Anchor preset"; parents first, so a Snap-mode child is placed in the parent where it went). Right-clicking an element on the canvas (selecting it first if it was not) opens a menu with **Anchors** (the same picker), **Select parent**, **Frame selection**, **Isolate this screen** and **Delete**.

The selected element is drawn with its parent's rect (dashed blue), the rect of every ancestor that clips its children (dashed red), anchor markers at its `AnchorMin` / `AnchorMax` in the parent, and its pivot (a circle, point axes only).

There is one source of the preview request. While the panel is open and on screen (not a background dock tab, not collapsed) it is the panel's; otherwise it is whatever the `ui_preview_*` calls asked for. Closing the panel stops the preview, so `ui_preview_ready()` is `false` a frame later unless `ui_preview_render` is in force. The overrides live in the panel, by UUID, and are never written to the scene. In Play they are off unless "Apply overrides in Play" is on.

These calls are session state: they never write the person's saved panel preferences (resolution, backdrop, follow), so a test can set each one it depends on without inheriting the last choice made by hand.

| Call | What it is |
|---|---|
| `ui_designer_open(open)` | Shows or closes the panel. It draws from the next frame; poll `ui_designer_state().visible`. |
| `ui_designer_set_resolution(width, height)` | The canvas's resolution (the toolbar's Custom). The preview is rebuilt at that size within a few frames; poll `ui_preview_ready()` and `ui_preview_size()`. |
| `ui_designer_set_background(r, g, b)` or `ui_designer_set_background("checker")` | A solid backdrop (0..1 per channel, the preview's clear colour) or the checkerboard. The checker is drawn by the preview's own pass under the HUD, so it is in the pixels `ui_preview_pixel` reads: squares of the clear colour 0.22 and 0.34 grey, 16 canvas pixels across (doubling when a large canvas would need thousands). |
| `ui_designer_set_follow(follow)` | Turns "Follow selection" on or off. It acts when the selection changes, and only for a UI element. |
| `ui_designer_isolate(name)` | Isolates the screen the named UI element belongs to; `nil` shows every screen again. `false` for a name that is not a UI element. |
| `ui_designer_set_visible(name, state)` | A preview-only override on one element: `true` shown (even if its own `Visible` is off), `false` hidden, `nil` as authored. `false` for a name that is not a UI element. Isolation wins over these for screens. |
| `ui_designer_click(x, y [, toggle])` | A click at a preview pixel, by the code a mouse click takes: selects the topmost element there (the name, which `editor.selected_name()` then answers) or clears the selection over empty canvas (`nil`). Over the body of a selected element it keeps the selection. `toggle` is a Ctrl-click: the element goes in or out of the selection. `nil` while the panel is closed. It does not grab handles: they are a screen-size affair. |
| `ui_designer_alt_click(x, y)` | An Alt-click: selects the next element down the stack under the point, starting below the primary selection when it is in that stack and at the topmost otherwise, and wrapping. The name, `nil` over empty canvas (which clears the selection). |
| `ui_designer_select(name, ... [, additive])` | Sets the selection to the named entities (a trailing `true` keeps the current one); the last becomes the primary. Answers how many names named an entity. `ui_designer_select()` clears. |
| `ui_designer_selection()` | The selected names, primary last. |
| `ui_designer_marquee(x0, y0, x1, y1 [, alt [, additive]])` | The rubber band over a rectangle of preview pixels, by the code the mouse takes: fully inside, or with `alt` anything it touches (a root screen only when wholly inside). Answers the selected names afterwards. Elements the preview does not draw are never in it. |
| `ui_designer_drag(name, handle, dx, dy [, shift [, alt [, cancel [, nosnap]]]])` | One whole gesture on the named element, by the code the mouse takes (press, first movement, last, release): `handle` is `"body"` (moves the selection; the element joins it if it was not in it) or `"n"`, `"s"`, `"e"`, `"w"`, `"ne"`, `"nw"`, `"se"`, `"sw"` (resizes it; it becomes the only selection unless it was the primary). `dx`, `dy` are the pointer's travel in preview pixels. One undo step. `cancel` ends it as Esc would; `nosnap` is Ctrl held through the drag. It snaps as the panel's switches say, so a test that wants exact landings turns them off first. `true` when it edited, `false` when refused (Play, locked, not drawn, unknown name or handle). |
| `ui_designer_set_snap{ smart =, grid =, grid_size =, pixel = }` | Sets the snapping switches (`grid_size` in UI units) for the session, never the saved preferences; what the table leaves out stays. The saved defaults are smart guides on, grid off (size 8), pixel snap off, so a test sets all three. |
| `ui_designer_guides()` | `guides, gaps` of the last move or resize (replaced by the next one). A guide is `{ axis = "x"\|"y", pos =, kind = "sibling"\|"parent"\|"canvas", from =, to = }` in canvas pixels (`x` is a vertical line at `pos`, spanning `from` to `to`); a gap is `{ axis, start, stop, cross, equal }`, its size `stop - start`. |
| `ui_designer_set_zoom(zoom)` | Zooms the canvas about its centre (1 is one canvas pixel to one screen pixel), so a test can see the snap reach follow it. |
| `ui_designer_drag_anchor(name, corner, dx, dy [, nosnap [, cancel]])` | One whole anchor-petal drag on the named element, by the code the mouse takes; one undo step. `corner` is `"tl"` or `"min"` (AnchorMin), `"br"` or `"max"` (AnchorMax), `"tr"`, `"bl"`, or `"point"` (the whole anchor rect shifted as one, which is how a point anchor is dragged). `dx`, `dy` are the pointer's travel in preview pixels. `nosnap` is Ctrl, `cancel` is Esc. The element becomes the only selection unless it was the primary. `true` when it edited, `false` when refused (Play, locked, not drawn, unknown name or corner). |
| `ui_designer_drag_pivot(name, dx, dy [, free [, cancel]])` | The same for the pivot circle; `free` is Alt (no snapping to 0, 0.5, 1). `false` for an element stretched on both axes. |
| `ui_designer_apply_preset(preset, mode)` | Applies an anchor preset to every editable selected element, as one undo step. `preset` is a name from the picker (`"Top Left"`, `"topleft"`, `"full_rect"`, `"VCenter Wide"`, ... ignoring case, spaces, `_` and `-`); `mode` is `"keep"` (the rect stays), `"keep_pivot"` (and the pivot becomes the preset's) or `"snap"` (anchors and pivot set, offsets zeroed). `false` for an unknown name, nothing selected or refused. |
| `ui_designer_palette()` | The palette's item names: the primitives, then the project's UI prefabs (rescanned on each call). |
| `ui_designer_create(item, x, y [, parentName])` | Creates the item by the code a drop takes: a Relative child of the element under the preview pixel `x, y` (or of `parentName`; top level over nothing), its top-left at the point, selected, one undo step. Answers the new element's name (unique), `nil` when refused (Play, a locked parent, an unknown item or parent, the panel closed). |
| `ui_designer_nudge(dx, dy)` | What an arrow key does: moves every editable selected element `dx`, `dy` UI units, as one `Nudge UI` undo step. `false` when there is nothing to move or it was refused. |
| `ui_designer_mouse(x, y, down [, shift [, alt [, ctrl [, vpivot]]]])` | A virtual pointer in place of the mouse over the canvas: at preview pixel `x`, `y`, left button down or up, from the next frame until `ui_designer_mouse()` with no arguments gives the mouse back. `vpivot` is the V key held. The panel's per-frame mouse code (press, drag threshold, a frame of movement at a time, release, handles, Esc, arrow keys, Ctrl+A) runs on it as on the mouse; `key("ESCAPE", ...)` and `key("LEFT", ...)` supply the keys. Not ImGui's own pointer: a window that has the OS focus re-reads the real cursor every frame. `false` while the panel is closed. |
| `ui_designer_rect(name)` | `x, y, w, h` of the element in the preview's layout (preview pixels, at the panel's resolution and with its overrides), `nil` when it is not drawn there. `get_ui_rect` answers for the game's own viewport instead. |
| `ui_designer_fields(name)` | Every `UITransformComponent` field of the named element as a table (`anchor_min`, `anchor_max`, `pivot`, `offset`, `offset_max`, `size` as `{x, y}`; `layer`, `visible`, `relative`, `clip_children`, `interactive`), so a test can compare all of them across an undo. `nil` for a name that has none. |
| `ui_resize_rect(x0, y0, x1, y1, handle, dx, dy [, shift [, alt [, minWidth, minHeight]]])` | The pure maths behind a resize, with no panel: the rect `x0, y0, x1, y1` after dragging `handle` by `dx, dy`. An edge moves only that edge; a corner two; `shift` keeps the start aspect (a corner follows the axis that changed more, an edge grows the other axis about its centre); `alt` mirrors about the centre; it stops at the minimum size (default 1 by 1, or the start size when that is smaller) and never flips. `"body"` translates. `nil` for an unknown handle. |
| `ui_designer_slice(name)` | The named sprite's 9-slice as `l, t, r, b` (texels), `scale`, `fill` (`"stretch"`, `"tile"` or `"tile_fit"`), `draw_center`, then the texture's `width, height` in texels (`0, 0` with no texture that loads). `nil` for a name that is not a sprite. |
| `ui_designer_drag_slice(name, edge, dtexels)` | One whole drag of a margin line by the code the pane's mouse drag takes: `edge` is `"l"`, `"t"`, `"r"` or `"b"`, `dtexels` the line's travel (right and down are positive, so dragging the right line right makes `r` smaller). It lands on a whole texel and stops at the opposite line; one undo step, `Edit 9-Slice`. `true` when it edited; `false` when refused (Play, locked, no sprite or texture, unknown edge or name) or when it moved nothing (nothing is recorded). Works on an unsliced sprite too: its lines are at the texture's edges. |
| `ui_designer_enable_slice(name)` | Turns 9-slice on with a quarter of the texture each way (rounded, at least one texel), one undo step. `false` when the sprite is sliced already, has no texture that loads, or it is refused. |
| `ui_designer_reorder(name, index)` | Moves a child a layout container places to the 0-based position `index` among the container's laid-out children (clamped), by the code a drag's release takes: Order is rewritten 0..n-1 over all of them in the new order (a UI Layout Element is added where missing), one undo step, `Reorder`. `false` when refused (Play, locked, not placed by a container, the name is nothing) or when it is already there (nothing is recorded). A drag out of the container is exercised through `ui_designer_mouse` (see `tests/ui_designer_containers.lua`). |
| `ui_designer_drag_container(name, handle, dx, dy)` | One whole drag of a selected container's own handle by `dx`, `dy` preview pixels: `handle` is `"pad_l"`, `"pad_t"`, `"pad_r"`, `"pad_b"` (the left and top bars grow the padding when dragged right / down, the right and bottom ones shrink it) or `"spacing_x"`, `"spacing_y"`; a padding or spacing handle answers to the travel along its own axis only, in whole UI units, never below 0. `"columns_minus"` / `"columns_plus"` step a Grid's column count by one (never below 1; `dx`, `dy` are ignored). One undo step (`Container padding`, `Container spacing`, `Grid columns`). `false` when refused, unknown, not a container (or not a Grid, for columns) or nothing changed. |
| `ui_designer_handles()` | The handles the canvas offers on the primary selection, as a list of names: the resize handles its axes allow (`"n"`, `"se"`, ...; none on an element a container places, and none on a Fit Content axis, where a corner needs both), and for a container `"pad_l"` ... `"pad_b"`, `"spacing_x"` / `"spacing_y"` where it has gaps, and on a Grid `"columns_minus"` / `"columns_plus"`. |
| `ui_designer_right_tab(tab)` | Clicks the right-hand column's tab, `"create"` or `"slice"` (the latter needs a sprite with a texture selected to show); the panel then stops choosing the tab itself. |
| `ui_designer_locked(name)` | Why the canvas will not move the element: `"Placed by <container>"` for a child a layout container arranges, `"Driven by <widget>"` for a slider's Fill and Handle and a toggle's Graphic; `nil` when it places itself (or the name is nothing). |
| `ui_designer_widget(name)` | The widget components on the element as a table -- `button = true`, `toggle = { is_on, graphic }`, `slider = { min, max, value, fill, handle }`, `layout = { type = "hbox"\|"vbox"\|"grid", columns, spacing = {x, y}, padding = {l, t, r, b} }`, `scroll = { content, vbar, hbar, horizontal, vertical }`, `scrollbar = { handle }`, `textfield = { text, placeholder, text_entity, placeholder_entity, filter, max_length, select_all_on_focus }` -- each key absent when the element lacks the component, the entity references resolved to the referenced element's name (empty when unset, `"?"` when they name nothing). `nil` for a name that is not an entity. What a test reads to check a palette-made widget is wired to its own children. |
| `ui_designer_scroll_preview(list, x, y)` | Shows the named scroll list scrolled to `x, y` UI units (clamped to what it can scroll) -- the Designer's own, designer-only offset, never the game's. The list becomes the selection when the selection is not already in it, so the offset has something to belong to. `false` for a name that is not a list, in Play, or while the panel is closed. |
| `ui_designer_scroll_state()` | `{ list, x, y, max_x, max_y, content_w, content_h, overflow, read_only }` for the list the primary selection is in (`x, y` the offset shown: the preview's, or the game's live one in Play, where `read_only` is true), `nil` when it is in none. |
| `ui_designer_show_overflow(on)` | The toolbar's **Show overflow**: the selected list's clipped content is outlined and pickable in the Designer's layout (the preview's `ui_pick` keeps the clip). |
| `ui_designer_show_focus_map(on)` | The toolbar's **Focus map** toggle. |
| `ui_designer_focus_map()` | The focus links of the panel's layout (the isolated screen) as a list of `{ from, dir, to, explicit }`: where each of `"up"`, `"down"`, `"left"`, `"right"` goes from every focusable widget, by `Scene::ComputeUIFocusMap` (the same `UIFocusMath` the game uses; compare `ui_focus_pick`). `explicit` marks a neighbour named by the widget's UI Focus. |
| `ui_designer_set_neighbour(name, dir, target)` | Makes the element named `target` the `dir` neighbour of widget `name`, or clears it when `target` is `nil`; one undo step, `Set focus neighbour`, and a repeat that changes nothing records nothing. `false` when refused (Play, locked, not a Button / Toggle / Slider, its own neighbour, an unknown name or direction). |
| `ui_designer_focus_handle(name, dir)` | The preview pixel `x, y` of the selected widget's arrow handle for `dir` at the current zoom -- where `ui_designer_mouse(x, y, true, false, true)` (Alt held) starts the Alt+drag that sets a neighbour -- `nil` for an element that is not a focus widget. |
| `ui_designer_hierarchy()` | The rows the Hierarchy pane shows, in display order (closed rows' children and, with a filter, what does not match are left out): each a table `{ name, id, depth, kind ("Panel", "Image", "Label", "Button", "Toggle", "Slider", "Scroll List", "Scrollbar", "Text Field", "HBox", "VBox", "Grid" or "Empty"), expanded, has_children, selected, root, visible_row = true, parent (nil for a screen), layer, container_index (0-based, only for an element a container lays out) and container, lock ("Placed by X" / "Driven by X"), sliced, default_focus, hidden (its Visible flag is off), editor_hidden, clip_children, locked (the editor lock on it), locked_inherited, eye ("authored", "hidden" or "shown": the preview override), isolated }`. A container's children are listed by Order, everything else by paint order, back to front. |
| `ui_designer_hierarchy_drop(names, target, where)` | A drop of the rows named (a name or a table of names) on the `target` row -- `"on"` (into it), `"before"` or `"after"` (the line above / below it; after an open row is the top of what it holds); `target` nil is empty space (the top level) -- by the code a mouse's drop takes, one undo step (`Reparent`, or `Reorder` when no parent changes). Answers `{ done, changed, reason }`: `done` false when refused (into itself or a descendant, a locked or driven element, Play), `changed` false when nothing was there to change. Into a layout container: its last slot, or at the line, with Order rewritten 0..n-1; into anything else: the rect on the canvas is kept, and between two siblings the element's own Layer is changed if that alone puts it there (else kept, and `reason` says so). |
| `ui_designer_rename(name, newName)` | The Hierarchy's rename (one undo step, `Rename UI Element`): `{ done, changed, reason }`. An empty name, Play and a locked element are refused. |
| `ui_designer_wrap(names, kind)` | Wrap in: `kind` is `"empty"`, `"hbox"`, `"vbox"`, `"grid"` or `"scroll_list"`. The elements (which must share a parent) go into a new container at their union rect; `{ done, changed, reason, element }` with `element` the wrapper's name. One undo step, `Wrap in <kind>`. |
| `ui_designer_unwrap(name)` | Moves the element's children to its parent keeping their rects and deletes it (refused when it carries anything beyond a UI Transform, Sprite and Layout, or holds nothing). `{ done, changed, reason }`; one undo step, `Unwrap`. |
| `ui_designer_set_layer(names, op)` | Edits Layer: `op` is `"forward"`, `"backward"`, `"front"`, `"back"` or a number. One undo step for all the elements; `{ done, changed, reason }`. |
| `ui_designer_hierarchy_filter(text)` | Sets the filter box (names and kinds, case-insensitive); `""` clears it. |
| `ui_designer_hierarchy_expand(name, open)` | Opens or closes a row; a nil name does every row. Screens are open until closed, everything else closed until opened. |
| `ui_designer_hierarchy_click(name, ctrl, shift)` | A click on the row by the pane's code: plain selects, `ctrl` toggles, `shift` selects the rows between the last click and this one. |
| `ui_designer_hierarchy_key(key)` | A key the pane handles: `"up"`, `"down"`, `"left"`, `"right"`, `"shift+up"`, `"shift+down"`, `"enter"` (frames the selection), `"f2"` (starts a rename: `ui_designer_state().renaming`), `"ctrl+a"`, `"escape"` (clears the filter). `false` when it did nothing. |
| `ui_designer_hierarchy_lock(name, locked)` | Sets the editor lock on the element: the very flag the Scene Hierarchy's padlock sets. |
| `ui_designer_hierarchy_hover(name)` | A virtual pointer over the row (nil clears): `ui_designer_state().hierarchy_hover` is the element it outlines on the canvas. |
| `ui_designer_hierarchy_drag(names, target, where)` | A stand-in for a mouse drag over the target row: the pane draws the line and works out the verdict, which `ui_designer_state()` carries a frame later as `drop_valid`, `drop_reason` and `drop_text` (the tooltip); nothing is dropped. nil target clears it. |
| `ui_designer_hierarchy_menu(name)` | Opens the row's context menu (selecting the row first, as a right-click does). |
| `ui_designer_hierarchy_pane(open, width)` | Opens or collapses the pane and sets its width (140 to 520; a session copy, never saved). |
| `ui_designer_state()` | A table: `open`, `visible` (on screen), `w`, `h` (the canvas), `zoom` (1 is one canvas pixel to one screen pixel), `slice_pane` (the 9-Slice tab is on screen for the primary selection) and `slice_tab` (the column has the tab at all: a sprite with a texture is selected), and `hovered` and `isolated` (names, absent when there is none). `hovered` needs a real cursor, so a test does not see one. The Hierarchy adds `hierarchy_open`, `hierarchy_width`, `hierarchy_focused`, `hierarchy_hover` (the element a row hover outlines on the canvas), `row_highlight` (the row lit because the canvas pointer is over its element), `renaming` (the element whose name is in an edit box), `drop_valid` / `drop_reason` / `drop_text` (what the last drop feedback said) and `hierarchy_build_ms` / `hierarchy_draw_ms` (what building the tree and drawing the pane cost last frame). |

### MCP tools from a test

| Call | What it is |
|---|---|
| `mcp_call(tool, args)` | Runs one of the editor's MCP tools -- what an AI agent calls ([AI Agents](../../../../07-projects-and-tools/04-editor-ai-agents.md)) -- right here on the frame loop, with no socket. `args` is a table (a nested table is an object, or an array when it is keyed 1..n) or JSON text, which is the way to pass a `null` (`'{"target": "R2", "parent": null}'`). Returns `is_error`, then the answer -- the tool's JSON as a table, or its text when that is not JSON -- then the number of images it returned. A tool that waits for frames (`ui_preview`, `wait`, `send_input`) is refused. The `ui_*` tools work whether or not the UI Designer panel is open. `tests/ui_mcp_tools.lua` is the working example. |

`tests/ui_designer_panel.lua` is the working example for the panel, `tests/ui_designer_edit.lua` for the gestures `tests/ui_designer_snap.lua` for snapping and `tests/ui_designer_anchors.lua` for anchors, the pivot and presets and `tests/ui_designer_palette.lua` for the palette and the V-key pivot, `tests/ui_designer_slice.lua` for the 9-slice pane and `tests/ui_designer_widgets.lua` for the widget palette and the locks and `tests/ui_designer_containers.lua` for container reorder, move out and handles.

## Play mode

The editor-scene-to-copy transition, which no runtime test can reach, because the runtime is only ever playing. It is also the one transition that has hidden a real bug before, so it is worth a test of its own.

| Call | What it is |
|---|---|
| `play()` | Enters Play from the editor. Answers whether play mode is now running. `false` when it is already playing. |
| `stop()` | Leaves Play. `false` when not playing. |
| `is_playing()` | Whether the editor is playing. |

## Build and packaging

Build Standalone from a script, run to completion: a test steps frames, and a build that spanned them would make "did it finish" the test's problem. Packaging stays a deliberate step on the agent route that reaches the same editor state ([AI Agents (MCP)](../../../../07-projects-and-tools/04-editor-ai-agents.md) refuses it on purpose), so this is its scripted form. See [Projects & Builds](../../../../07-projects-and-tools/01-projects-and-builds.md).

| Call | What it is |
|---|---|
| `build_standalone(directory [, options])` | Runs Build Standalone to completion and answers whether it published. The reason a build failed is in `last_build_result()`, the line the modal would show. `options` takes `strip_unreferenced`, `all_scenes`, `scene` (a start scene override), `target` (`"windows"` or `"linux"`), `runtime` (a LABRuntime to ship) and, for the build target, `demo`, `steam`, `steam_app_id`, `demo_app_id` and `require_ownership`, each overriding the project's Build and Steam setting for that one build. |
| `build_linux_runtime([options])` | Build Standalone's Build Linux runtime button: starts a Linux runtime build. Options: `native` and `config` (default `"Dist"`). `false` when a package build is in progress, because a package may be about to copy the very binary this would relink. |
| `linux_runtime_build_state()` | `"idle"`, `"running"`, `"succeeded"`, `"failed"` or `"cancelled"`. |
| `linux_runtime_path()` | The runtime binary a Linux package would ship. |
| `last_build_result()` | The last build's result line. |
| `set_pack_into_archive(value)` | Sets the project's `PackIntoArchive`. Whether a build packs into a `.Lpak` is project-config-only, so a test needs a way to flip it that is not a `build_standalone` argument. |
| `set_build_options(table)` | Changes the open project's demo and Steam build settings in memory, never saved by itself: `{ demo, steam, steam_app_id, demo_app_id, require_ownership, website, input_manifest }`, a field left out stays. A later `build_standalone` without those options builds with them. |
| `build_summary([target])` | The line the Build dialog shows under its Build button, for `"windows"` (the default) or `"linux"`: `Windows, Steam (app 123456), demo`. |
| `set_native_module(path)` | Sets the project's `NativeModule`. Packaging resolves it beside the target runtime, so a test can point a build at a module already built there without editing the project file. |

`build_standalone`'s optional table carries `strip_unreferenced`, `all_scenes` and `scene` (a start-scene override, asset-relative), plus `target` (`"windows"` or `"linux"`, empty for the host) and `runtime` (a runtime binary to ship, empty for the one the packager finds). The last two are how a script packages for the other OS, or ships a particular build of the runtime. With no options at all, the project's own Build settings apply, exactly as the modal does. A packaged module is staged beside the runtime and kept out of the archive, the same treatment the Steam library gets.

## Browsers and the import dialog

The two browsers and the Import Settings modal. The listing filter is the whole point of having two browsers, and it is UI code no runtime test can reach, so these calls answer what each browser would show and, where a panel-only ImGui assert needs its chance to fire, drive the panels for real. See [Importing Assets](../../../../02-building-worlds/04-importing-assets.md).

| Call | What it is |
|---|---|
| `list_assets(folder)` | What the Asset Browser would list for a folder, asset-relative: a 1-indexed table of file names, native assets only. |
| `list_sources(folder)` | What the Source Browser would list for a folder: raw files only. |
| `show_source_browser(show)` | Shows or hides the Source Browser window. Answers nothing. |
| `browse_assets(folder)` | Navigates the Asset Browser to a folder and shows it, tiles and container thumbnails included. A folder that does not resolve leaves the panel where it was. |
| `browse_sources(folder)` | The same for the Source Browser, and a folder that is not under the asset directory is looked for under the project's source directory. |
| `open_import_dialog(source)` | Opens the Import Settings modal for a source file, resolved like every source path. Answers whether the dialog is open. |
| `import_dialog_open()` | Whether the modal is open or visible. |
| `close_import_dialog()` | Asks the modal to close; the panel closes it on its own next pass. |
| `select_browser_files(sources, names)` | Multi-selects in either browser (`sources` picks the Source Browser) by file name in the folder that browser is showing. Answers how many of them matched and are now selected. Ctrl-click and shift-click themselves are ImGui input no test can produce, so this is the seam. |
| `reimport_browser_selection(sources, queue)` | What Reimport N on that selection would reimport, as absolute asset paths, and queued as one batch when `queue` is true. |

## Input and gestures

The editor's own gestures, queued through ImGui at a low enough level to exercise press versus drag, modifier keys and shortcut routing rather than simulating the outcome. Sending a down and an up several frames apart at one position is a click; moving between them past the pick-drag threshold is a drag. What still cannot be produced this way is anything ImGui or ImGuizmo infer from continuous pointer motion within a single frame, or anything about how the UI looks.

| Call | What it is |
|---|---|
| `viewport_mouse(u, v, down)` | Queues a mouse button at a viewport fraction across frames. How a test drives a gizmo drag or a marquee. Answers nothing. |
| `hierarchy_mouse(name, lock, down)` | Clicks a hierarchy row's visibility button (`lock` false) or lock button (`lock` true), by entity name. Answers whether the button was drawn. |
| `key(key, down, ctrl, shift)` | Queues a key through ImGui, so shortcut routing (Ctrl+Z, Ctrl+Y) is actually exercised. Takes a single letter, A to Z, or one of `"ESCAPE"`, `"LEFT"`, `"RIGHT"`, `"UP"`, `"DOWN"`, `"DELETE"`, with the two modifiers set alongside it. Answers nothing. |

## Shaders, console and probes

The remaining odds and ends: the two shader-reload readers, the script console, the named probe commands, and the platform question.

| Call | What it is |
|---|---|
| `reload_shaders()` | Asks for a shader reload. The work is deferred to the top of the next frame, because pipelines are being destroyed and the frame it would land in the middle of is not a legal place to do it, so the answer is only that the request was made. |
| `shader_reload_ok()` | Whether the last completed reload succeeded. |
| `console_eval(line)` | Evaluates one line in the script console, the way typing into the panel would. Answers the evaluated result text, or `"ERROR: <message>"` on a compile or runtime error. |
| `command(name [, arg])` | A named editor-state command with one string argument, for probes that need to put the editor into a realistic UI state without a dedicated hook each. Unknown names answer `"unknown"`. |
| `platform()` | `"windows"` or `"linux"`. The sandbox opens no `os` and no `package`, so a test that has to name a platform-specific file, and a native module is the one that matters, has no other way to ask. |

What `command` knows:

| Name | |
|---|---|
| `material_thumbnail_pixel <path>` | Renders the real browser thumbnail and answers the centre pixel as `"r,g,b"`, or `"pending"` when the render is still queued, or `"unavailable"` with no renderer. |
| `focus_window <name>` | Brings a docked tab forward and focuses it. Several in one frame are applied in order, `'\|'`-separated. |
| `expand_hierarchy 1\|0` | Opens or closes every hierarchy row on the next frame. |
| `browser_view <assets\|sources>:<thumbs\|list>` | Switches a browser's view, not saved. |
| `clear_thumbnails` | Drops every cached thumbnail, so the next draw uploads again. |

## UI scale and the font atlas

The editor's UI scale rebuilds its font atlas when it settles, and these read and drive that. A test run starts from default editor settings (it reads no `editor.yaml`), so the scale is 1.0 at the start.

| Call | What it is |
|---|---|
| `set_ui_scale(scale)` | Settles a scale the way letting go of the Editor Settings slider does, clamped to 0.5 to 2.5. The atlas is rebuilt at the start of the next frame; settling the scale already in use rebuilds nothing. |
| `ui_scale()` | The scale in use. |
| `font_atlas_serial()` | How many times the font atlas has been built, 1 for the startup build. A change means a new font texture. |
| `font_atlas_size()` | The atlas texture's `width, height` in pixels. |
| `font_texture_id()` | The font texture's id as a number, to tell one texture from another. |
| `font_size_pixels(name)` | The pixel size the font `"Default"`, `"Bold"` or `"Italic"` is built at now (20 times the scale), or `nil` for another name. |
| `ui_scale_for_display_scale(s)` | The scale a monitor display scale `s` seeds: rounded to a quarter step, clamped to the slider range. |

## The same state from an agent

The editor's MCP server reaches the same editor, the same panels and the same undo history, with `select_entities`, `set_camera`, `undo`, `redo`, `play`, `stop`, `send_input`, `track_entities`, `get_errors` and `screenshot` covering most of what this page exposes, and it refuses Build Standalone and any edit while playing. See [AI Agents (MCP)](../../../../07-projects-and-tools/04-editor-ai-agents.md). The difference is what can be asserted: an agent looks at a screenshot, a test compares numbers it can re-run.
