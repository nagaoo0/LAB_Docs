---
title: "AI Agents in the Editor (MCP)"
---

LAB can let an AI agent build in the editor with you. Claude Code, Claude Desktop, or any
other client that speaks the **Model Context Protocol (MCP)** can do the following in the
editor you have open:

- create and edit entities
- write Lua gameplay scripts
- press Play, **drive the game with real key presses**, and measure what moved
- look at the viewport
- read the errors

It is meant for prototyping. Describe a game or a feature ("a ball the player rolls with
WASD, three ramps and a goal that prints a message") and let the agent block it out while
you watch and correct it.

Everything the agent does goes through the same code paths you use:
- **Edits are normal undo steps.** Ctrl+Z works on them.
- **Its scripts are ordinary `.lua` files** in your assets.
- **Nothing is saved until the agent (or you) saves the scene.**

## Turning it on

Open **Windows → Editor Settings** and expand **AI Agents (MCP)** (or press **Ctrl+P** and pick "AI Agents (MCP)", which opens it there) and tick **Start the server with the editor**. The panel
shows three things:
- the address it listens on, `http://127.0.0.1:7801/mcp` by default
- which client is connected
- the last tool it called

To start it for one session without changing the setting, launch the editor with `--mcp`,
or with `--mcp-port <port>` to use another port.

The server listens on this machine only (127.0.0.1) and rejects requests from web pages. It
is **off by default**, because while it runs any program on your computer can drive the
editor.

**Switching off `run_lua` is not a sandbox.** It stops that one tool evaluating Lua directly,
which is worth having -- it is the difference between a slip and a deliberate act. But an
agent can still `write_file` a `.lua` script and `play` the scene to run it, so anything
connected to this server can execute code in the editor either way. The control that matters
is the server being off, or being on only while you are watching. Treat a connected agent as
something running on your machine with your permissions, because that is what it is.

## Connecting a client

**Claude Code.** Run this once in a terminal:

```bash
claude mcp add --transport http lab-editor http://127.0.0.1:7801/mcp
```

Then start `claude` from anywhere and ask it to build something in LAB. The editor has to be
running with the server on; if you restart the editor, the client reconnects on its next
request.

**Other clients** that read an `mcpServers` config (a project `.mcp.json`, for example) take
the snippet the panel's **Copy config** button puts on the clipboard:

```json
{
  "mcpServers": {
    "lab-editor": { "type": "http", "url": "http://127.0.0.1:7801/mcp" }
  }
}
```

## What the agent can do

| Tool | What it does |
|---|---|
| `get_editor_state` | Project, scene, play state, unsaved changes, selection, undo history, which viewport overlays are showing, the editor camera |
| `list_entities`, `get_entity` | Read the scene: ids, names, hierarchy, world position and world matrix, every component as scene-file YAML |
| `describe_components` | Every component and its default fields, with the conventions the fields do not show (enum meanings, and that a directional light shines along its entity's +X axis, so its direction is its rotation) |
| `create_entity`, `create_primitive` | Add entities. Primitives are a cube, sphere, plane, cylinder, capsule or cone (the engine's own meshes), with a colour and optional physics. A directional light can be aimed with `Direction` or `LookAt` inside its component instead of working out the rotation |
| `edit_entity` | Rename, change component fields, add or remove components |
| `delete_entities`, `duplicate_entities`, `set_parent` | Structure |
| `ui_tree`, `ui_preview`, `ui_create`, `ui_set_rect`, `ui_set_anchors`, `ui_reorder`, `ui_reparent`, `ui_rename`, `ui_wrap`, `ui_unwrap`, `ui_set_layer`, `ui_slice`, `ui_pick`, `ui_nudge`, `ui_align`, `ui_distribute`, `ui_set_neighbour`, `ui_focus_map` | Author game UI through the UI Designer's own code: read it, build it, place it, look at it. See [Authoring game UI](#authoring-game-ui) |
| `save_as_object`, `instantiate_object` | Objects (prefabs): save a built entity with its script and children as a `.Lobj`, place copies. Scripts spawn them with `scene.spawn`. `instantiate_object` also takes `rotation` (`[x, y, z, w]`), `scale` and `name` |
| `select_entities`, `set_camera` | Point the editor at something. `set_camera` puts the viewport camera exactly at a position and aim at once (no smoothing) and reports the pose it now has |
| `set_overlays` | Show or hide the viewport overlays (grid, blockout wireframes, colliders, splines, skeletons, probes, the selection outline, entity icons, gizmos) for a clean shot. `preset: "clean"` turns them all off. Session only, nothing is saved |
| `undo`, `redo` | The editor's own history |
| `play`, `stop`, `wait` | Run the game and let it simulate |
| `send_input`, `release_input` | Press keys and mouse buttons in the running game -- real presses with real edges, read by `input.is_key_down`, `input.is_key_pressed` and your input actions alike. Play mode only |
| `track_entities` | Record entities while the game runs and report what happened: distance travelled, lowest point reached, peak speed, destroyed or not |
| `get_errors` | Script failures with their message and entity, Vulkan validation errors, recent warnings -- with a cursor, so an agent sees what *it* broke |
| `get_audio_state` | Every sound currently playing, with its clip, bus, volume and position |
| `list_node_types` | Every visual-script and material-graph node type, generated from the engine's own tables |
| `screenshot` | The viewport as rendered, as an image. It waits for the editor to draw two fresh frames first, so it shows every edit and camera move made before it |
| `record_video`, `stop_video`, `video_status` | Record the viewport (gameplay while playing) as MP4, a PNG sequence or AVI, with the editor's Export Video. `seconds` is optional (up to the `max_seconds` safety limit, 2 hours by default); `stop_video` ends a take early. For a long take use `split_seconds` to write it as `_part001`, `_part002`... files inside one recording, so nothing is lost between them. Chaining separate `record_video` calls lets the game run between takes. A new take can start as soon as the last one ends, even while its file is still being finalised (`video_status` shows `finalising`) |
| `view_image` | Look at an image in the project: a frame of a PNG-sequence recording, a texture, a screenshot |
| `set_render_size` | Fix the render resolution, e.g. 1280x720 before recording |
| `get_log` | Recent log lines, including script errors |
| `list_files`, `read_file`, `write_file` | Text files inside the project (scripts, materials, scenes). `root` can be `engine` to read the engine's own content (`Engine/Content`): it needs no project and is never written, `write_file` there is refused |
| `import_asset` | Import a raw file from the project's source folder (`root` picks another folder). A model also becomes an object. Returns a `job_id` |
| `get_import_status`, `reimport_asset`, `reimport_source` | Follow an import by `job_id`; reimport a native asset, or everything imported from one source file. A model imported as an object has the object rebuilt and its placed copies refreshed (`rebuilt_object`, or `kept_object` for one edited since it was built; `rebuild_object: false` skips it) |
| `blender_export_blend`, `make_lods`, `make_convex_hull`, `assign_lods` | Headless Blender jobs and what to do with their output. The first three start Blender in the background and return a `job_id`; `get_import_status` reports the job as pending until Blender and the import after it are both done, with the script's own report under `result` and `refused: true` when the script declined (shape keys, an armature). `assign_lods` fills an entity's LOD meshes from the files `make_lods` wrote. See [Blender](../02-building-worlds/06-blender.md#headless-jobs) |
| `get_project_info`, `get_asset_selection`, `open_in_blender`, `blender_status` | The project's directories, what is selected in the Asset and Source Browsers, and the Blender hand-off. These are the calls the Blender add-on makes; see [Blender](../02-building-worlds/06-blender.md) |
| `open_scene`, `new_scene`, `save_scene` | Scene files |
| `blockout_create_model`, `blockout_create_brush`, `blockout_create_on_surface`, `blockout_get_brush`, `blockout_edit_brush`, `blockout_clip`, `blockout_group`, `blockout_ungroup`, `blockout_set_order`, `blockout_get_model`, `blockout_rebake`, `blockout_tool` | Level blockout with convex CSG brushes: boxes, cylinders, stairs, arches, polygons; add, subtract, bevel, clip, reshape faces. `blockout_tool` drives the interactive Place tool with pointer events, like a mouse. See [Blockout Tools](../02-building-worlds/05-blockout-tools.md) |
| `get_manual` | This manual; also offered as MCP resources |
| `run_lua` | Evaluate Lua in the editor's script console. Can be switched off in the panel -- though see the warning below |

`create_primitive` makes a cube, sphere, plane, cylinder, capsule or cone from the engine's own
primitives (`engine:Primitives/<Shape>.Lmesh`, see [Projects & Builds](01-projects-and-builds.md#engine-content)).
Nothing is written into the project, and the scene stores the engine reference, so a build
ships only the primitives it uses. A primitive baked to a size (`size`, with `uv_meters` for
world-scaled UVs) is geometry of its own: that one is written once per project to
`models/primitives/` as a plain OBJ, an ordinary asset you can replace or cook. `create_primitive`
works without a project open as long as it is not baked.

## Authoring game UI

A game's HUD -- panels, labels, buttons, health bars, menus -- is **authored through the `ui_*` tools, or in the
[UI Designer](../05-ui/02-ui-designer.md), not generated by scripts**. The tools do what the Designer does, through the same
code: the layout an element is read from, the lock rules (a container's child, a slider's Fill), "no edits while
playing", and one undo step per call are the Designer's own, so an agent and a person edit one document with one
history, and the panel does not have to be open. Scripts are for behaviour (what a button *does*), not for
laying out what the player sees.

| Tool | What it does |
|---|---|
| `ui_tree` | The HUD as a tree: every element with its id, name, resolved rect, anchors, pivot, offsets, size, layer, visibility, a sprite / text / widget / container summary, and why it is locked. A scroll list has a `scroll` summary (its Content and scrollbar ids, the window, the `content_size` and `max_scroll`), its bar a `scrollbar` one (the list, the axis, the thumb), a widget with a UI Focus a `focus` one (explicit neighbours, default focus), a scope `focus_scope`, and a Text Field a `text_field` one (text, placeholder, filter, max length, password and the ids of the two elements that draw them -- which carry `shows_text_of`, since their own Text is ignored). Children are listed in the order the UI Designer's Hierarchy shows them -- `children_order` says which: `layout` (a layout container's children, by Order: the order it lays them out in, with `container.index` the 0-based slot) or `paint` (any other element's, and the `roots`: back to front, so the last is drawn on top and a click reaches it first). Start here |
| `ui_preview` | A PNG of the UI alone, drawn by the same renderer as the in-game HUD. Waits for a fresh frame, so it shows every edit made before it. Options: `isolate` one screen, `show` / `hide` elements for this picture, a `background` |
| `ui_create` | Create an item from the Designer's palette (Panel, Label, Image, Empty, Button, Toggle, Slider, HBox, VBox, Grid, Scroll List, Text Field, 9-Slice Panel) or a UI prefab, inside a parent, at a position or an exact rect |
| `ui_set_rect` | Place an element so its rect is exactly `{x, y, w, h}`, keeping its anchors and pivot. The numbers go in a `rect` object, the one `ui_create` takes, or flat as `x`, `y`, `w`, `h`; any left out keep their value |
| `ui_set_anchors` | An anchor preset (Top Right, Full Rect ...) or raw anchors, keeping the element where it is or letting it move |
| `ui_reorder`, `ui_reparent` | Order a container's children; move an element under another (or to the top level) |
| `ui_rename` | Rename an element (F2 in the Hierarchy). One undo step |
| `ui_wrap` | Wrap `targets` (elements that share a parent) in a new `container` -- `empty` (a plain group: their rects are kept), `hbox` / `vbox` / `grid` (they are laid out, in the order `ui_tree` lists them) or `scroll_list` -- made at their bounds; it takes the first one's place in a container above and the lowest of their layers so paint order holds. Returns the `wrapper` and the wrapped elements with their new rects. One undo step |
| `ui_unwrap` | Move an element's children to its parent keeping their rects (layers added back), then delete it. Refused when it carries anything beyond a UI Transform, a Sprite and a UI Layout, or has no children. One undo step |
| `ui_set_layer` | Edit Layer, the paint order among siblings: `layer` is a number, or `forward` / `backward` (one step in paint order, changing only that element's Layer when one Layer does it), `front` / `back` (past every sibling). One undo step for all the targets |
| `ui_slice` | A sprite's 9-slice borders, scale, fill and centre |
| `ui_pick` | The elements under a point, topmost first |
| `ui_nudge`, `ui_align`, `ui_distribute` | Move by an offset; align edges or centres; space three or more elements evenly |
| `ui_set_neighbour` | Where keyboard / gamepad focus goes from a widget in a direction (`up`, `down`, `left`, `right`): another widget, or `null` for the automatic (nearest) one. One undo step; refused in Play |
| `ui_focus_map` | For every widget that can take focus, where each direction goes -- worked out as the game does it -- with `explicit` marking the ones a UI Focus names, plus the default-focus widgets and the scopes. Read-only |

Text, colours, widget values (a slider's value, a toggle's group), container padding and spacing, and layer
stay on `edit_entity`; `describe_components` lists every HUD component with every optional field.

### The loop

1. **`ui_tree`** to see what is there (a new scene has none).
2. **`ui_create`** the pieces, **`ui_set_rect`** / **`ui_set_anchors`** to place them, `edit_entity` for their
   text and colours.
3. **`ui_preview`** to look. Fix what looks wrong; `ui_tree` shows the numbers behind the picture.
4. **`save_scene`** when it is right. `undo` takes back one call at a time.

### The design resolution

Every `ui_*` call works on a canvas of a **design resolution** -- 1920 x 1080 unless a call says otherwise
(`width`, `height`; the last one given is remembered for the session). `(0, 0)` is the canvas's top-left and y
grows downward. Elements are placed against it by their anchors, so a HUD laid out at 1920 x 1080 and one laid
out at 1280 x 720 are the same document; the resolution only says which one you are looking at. Numbers are in
**UI units** -- what a UI Transform stores, pixels divided by the project's UI scale (Project Settings, UI
Reference Height; the scale is 1 when it is unset). A call may say `"units": "px"` for pixels of the canvas
instead; `ui_tree` always gives both (`rect` and `rect_px`).

### Examples

A health bar, top-left, then a look at it:

```json
{ "tool": "ui_create", "item": "Empty", "name": "Hud", "rect": { "x": 0, "y": 0, "w": 1920, "h": 1080 } }
{ "tool": "ui_create", "item": "Slider", "name": "Health", "parent": "Hud", "rect": { "x": 32, "y": 32, "w": 360, "h": 28 } }
{ "tool": "ui_create", "item": "Label", "name": "Score", "parent": "Hud", "x": 1700, "y": 32 }
{ "tool": "edit_entity", "entity": "Score", "components": { "TextComponent": { "Text": "Score 0", "FontSize": 36 } } }
{ "tool": "ui_set_anchors", "target": "Score", "preset": "Top Right", "mode": "keep" }
{ "tool": "ui_preview" }
```

(`Empty` is a group that fills its parent -- here the whole canvas, so the bar and the label are positioned
against the screen. `ui_set_anchors` pins the score to the top-right corner so it stays there at other
resolutions; `"mode": "keep"` leaves it where it is on the canvas.)

Three buttons in a column with equal gaps, lined up on the left edge:

```json
{ "tool": "ui_align", "targets": ["Play", "Options", "Quit"], "edge": "left" }
{ "tool": "ui_distribute", "targets": ["Play", "Options", "Quit"], "axis": "vertical" }
```

A scrolling list of rows, and a menu whose focus order is pinned by hand (the rest stays automatic):

```json
{ "tool": "ui_create", "item": "Scroll List", "name": "Quests", "rect": { "x": 40, "y": 120, "w": 420, "h": 480 } }
{ "tool": "ui_set_neighbour", "target": "Play", "dir": "down", "to": "Options" }
{ "tool": "ui_set_neighbour", "target": "Quit", "dir": "up", "to": null }
{ "tool": "ui_focus_map" }
```

(A Scroll List comes with its Content VBox, six sample rows and a wired scrollbar -- `created` lists them; replace or
add rows inside "Quests Content". Focus navigation only does anything when the project turns it on.)

A row that arranges its own children: create an `HBox`, then create the buttons with the HBox as `parent`
(the container places them; `ui_reorder` changes the order, `edit_entity` on the HBox changes spacing).

Order has two meanings, and `ui_tree` says which. In a layout container it is **Order** (`ui_reorder`, or `ui_reparent`
with an `index`): the container lays its children out first slot first. Everywhere else it is **paint order**, which is
**Layer** (`ui_set_layer`) and then the scene's own order for elements on one Layer: the later one in the list is
drawn on top. `ui_wrap` and `ui_unwrap` keep it: a wrapper takes the lowest layer of what it wraps, and the layers go back on an
unwrap. Between two siblings that share a Layer no single Layer change can put an element, so `forward` and `backward` pass
the neighbour's Layer instead, and a drop in the Hierarchy keeps the element's own.

### What is refused, and why

- **While playing.** The tools say "call `stop` first": the play-mode HUD is a copy.
- **An element a layout container places** ("Placed by Row"): its own anchors, offset and size are not used, so
  `ui_set_rect` and `ui_set_anchors` refuse it and tell you what to change instead (the container's padding and
  spacing, the child's UI Layout Element, or `ui_reorder` / `ui_reparent`). `ui_tree` shows this as `lock`.
- **An element a widget drives** ("Driven by Health"): a slider's Fill and Handle and a toggle's Graphic are
  placed from the widget's value. Change the value with `edit_entity`.
- **An entity locked in the hierarchy.**

`ui_align`, `ui_distribute` and `ui_nudge` skip such elements and list them under `skipped`, doing the rest.

Every mutating tool returns the element's new rect (`element`, or `elements`), so the result of a call can be
checked without another one, and `changed: false` when the call left things as they were.

## What it will not do

- **Edit while playing.** The play-mode scene is a copy that Stop throws away, so edits are
  refused until the agent stops.
- **Touch files outside the project,** or write binary asset formats (`.Lmesh`, `.Ltex`, ...).
- **Discard your unsaved work.** Opening another scene with unsaved changes is refused
  unless the agent explicitly asks to discard them. A scene `new_scene` just made has none
  until something in it is edited.
- **Build Standalone** for you. Packaging stays a deliberate, human step
  (see [Projects & Builds](01-projects-and-builds.md)).

## How an agent tests what it built

A screenshot of a game nobody is playing shows a player standing still, which is not evidence
the controls work. The tools for the rest of the loop:

- **`send_input`** presses keys in the running game. They are real presses through the same
  path your keyboard takes, so `input.is_key_down`, `input.is_key_pressed`, `get_key_axis`
  and your project's input actions all see them, edges included. Play mode only -- in edit
  mode those keys would be driving the editor's own shortcuts instead.
- **`track_entities`** records entities on the frame loop while the game runs, then reports
  distance travelled, straight-line displacement, the lowest point reached, peak speed, and
  whether anything was destroyed. That is what answers *did the jump clear the gap* and *did
  the crate fall through the floor* -- questions a screenshot cannot answer and two polled
  positions get wrong.
- **`get_errors`** returns script failures as data: the message, the script, the entity and
  which callback it was in. It carries a cursor, so an agent can tell its own breakage from
  what was already broken.
- **`get_audio_state`** lists what is actually playing. A cue that never fires and a cue that
  fires silently look identical otherwise.

So the loop is: `play`, `send_input`, `track_entities`, `get_errors`, `screenshot`, `stop`.

The `ui_*` tools have end-to-end smoke scripts that start a Debug editor on a spare port and call them over HTTP: `python scripts/mcp/ui_mcp_smoke.py` (every tool, pixel checks on the `ui_preview` PNG, undo, Play; it also builds a small menu and saves `LAB/logs/ui_mcp_preview.png` and `ui_mcp_tree.json`) and `python scripts/mcp/ui_mcp_panel_smoke.py` (the same while the UI Designer panel is open at another size). `Dev/Tests/assets/tests/ui_mcp_tools.lua` covers the same handlers inside the frame loop through `editor.mcp_call`.

## Tips

- Keep the editor visible while the agent works; everything happens live in front of you.
  The **AI Agents** panel shows any keys the agent is currently holding.
- Ask the agent to take a screenshot after each change and fix what looks wrong. That loop
  catches most mistakes.
- For a HUD, have it call `ui_preview` after every few edits and `ui_tree` when something is off: the picture shows what
  is wrong, the tree shows the numbers behind it.
- Ask it to *test*, not just to build. "Add a double jump and show me it working" gets a very
  different result from "add a double jump".
- A script that fails is disabled rather than spamming the log (see
  [Lua Scripting](../06-scripting/02-lua/index.md)), so one broken script does not hide the rest.
- Undo works on everything the agent did. If a whole direction was wrong, undo back to
  where it started.
- Pressing **Stop** releases anything the agent was holding down, as does switching the
  server off. A key cannot stay stuck after the game it was driving has gone.
