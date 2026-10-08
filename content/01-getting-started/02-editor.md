---
title: "The Editor"
---

The editor is a dockable ImGui workspace. Every panel can be dragged out, tabbed, or
docked anywhere; the layout is remembered between sessions in `LAB/imgui.ini`.

The first time the editor starts, with no saved layout, it opens in a default arrangement:
the Scene Hierarchy on the left, the viewport in the middle, Properties on the right, and
the Asset Browser along the bottom with the Source Browser and the Console as tabs beside
it. The side columns take roughly a fifth of the window each and the bottom row a little
over a quarter of its height. Panels you open later, such as Performance or the graph
editors, join the right column or the viewport as tabs rather than floating. **Windows >
Reset Layout** (also in the Command Palette) puts everything back in that arrangement and
shows the four core panels again. A layout you saved is never replaced except by that
command.

## The menu bar

### File

| Item | Notes |
|---|---|
| New Project… / Open Project… | See [Projects & Builds](../07-projects-and-tools/01-projects-and-builds.md) |
| Open Recent | Last ten projects, stored per user, not per project |
| Save Project | Writes the `.lab` manifest |
| Close Project | Falls back to the executable's own `assets/` folder |
| Set Current Scene As Start Scene | The scene a build (and a project reopen) starts in |
| New Scene `Ctrl+N` | |
| Open Scene `Ctrl+O` | Also lists the `.Lscene` files it finds under the asset folder |
| Save Scene `Ctrl+S` | |
| Save Scene As… `Ctrl+Shift+S` | |
| Recover Unsaved Work | Autosaved scenes from an earlier session of this project; greyed out when there are none. See below |
| Exit | Asks first if anything is unsaved. See below |

#### Closing with unsaved work

File → Exit, the title-bar X, the window's own close button and `Alt+F4` all do the same thing.
If nothing is unsaved the editor closes. Otherwise one **Unsaved changes** prompt lists every
document that has changes: the scene, an object you are editing, the script graph, the material
graph and material instance, the animation graph, and the project settings (the Project Settings
window shows a `*` in its title while they differ from the file). Each line names the document and
its file, or "untitled".

**Save All and Exit** writes every one of them, and asks for a name first when the scene is
untitled; cancelling that name dialog cancels the exit too. If one of them cannot be written the
prompt stays up with the reason and nothing is closed. **Discard All and Exit** closes without
writing anything and removes the recovery snapshot, so the discarded work is not offered next time.
**Cancel** (or `Escape`) goes back to the editor; `Enter` is Save All and Exit.

Opening another script graph while the open one has unsaved changes, and closing the Node Graph
window on it, ask the same way (Save and Open, Discard and Open, Cancel).

#### Crash recovery

While a scene has unsaved changes the editor writes a recovery snapshot every 45 seconds, and
once more when it closes. Saving the scene (or undoing back to the saved state) removes it.
A "Recovery snapshot saved" toast appears for the first one of a session only; a snapshot that cannot
be written always shows an error toast.
Each running editor keeps its own snapshot, named for its process and start time, with a small
`.meta` file beside it recording the project and scene, so several editors on one machine (your
own, plus ones an AI agent started) never overwrite each other. They live in
`%APPDATA%\LAB\recovery` on Windows and `~/.config/LAB/recovery` on Linux (`LAB_RECOVERY_DIR`
overrides it).

When a project opens, the editor looks for snapshots of that project whose editor is no longer
running, and a snapshot belonging to a live editor is never offered. If it finds any it logs a
line, shows a toast, and enables **File → Recover Unsaved Work**, which lists them newest first
(the time, the scene, and "agent editor" for one started with `--mcp`). Choosing one opens it as
an unsaved scene, so nothing is written over the saved file until you save. It will not replace a
scene that has unsaved edits of its own: save or discard first. **Discard These Snapshots** deletes
the listed ones. Automated runs (`--headless`, `--test`) neither write nor offer snapshots.

Snapshots older than 30 days, and all but the newest ten per project, are deleted when a project
opens. A `latest.Lscene` left by an older editor is offered once and then moved to
`recovery/retired`.

### Build

**Build Standalone…** packages the runtime, the compiled shaders and your project's assets
into a folder that runs without the editor. Covered in
[Projects & Builds](../07-projects-and-tools/01-projects-and-builds.md).

### Engine

- **Reload Shaders** — rebuilds every pipeline from the `.spv` files on disk. It reads
  SPIR-V and does *not* invoke a compiler, so run `scripts/Build-Shaders.ps1` first. A file
  that fails to load leaves the running pipelines untouched, so a typo does not black-screen
  the editor.
- **Ray Tracing** — a checkbox, not a window: whether the device's hardware ray-tracing
  path is used at all this session. Greyed out with a tooltip when the device has no
  acceleration-structure extensions. It used to live inside the stats panel, which made a
  panel you open to *measure* into one that changes what it measures.

### Edit

Undo, Redo, Copy, Duplicate, Delete, Paste. The Undo and Redo entries name what they will
actually undo ("Undo Move Entity"), which is usually faster than guessing. **Editing is
disabled while playing** — play mode simulates a throwaway copy of the scene, so an undo
there would change something that is about to be discarded.

### Windows

Every panel toggle lives here, plus two commands that just open rather than toggle:

- **Command Palette** (`Ctrl+P`) — a searchable flat list of scenes, panels, workspace
  presets, viewport overlays, every entity in the open scene by name, and recently used
  assets. Type to filter, arrow keys to move the highlight, Enter to run it. Faster than
  hunting through menus once you know roughly what you want by name.
- **Keyboard Shortcuts** — the reference sheet, in-app. It also lists the keys the Node Graph,
  Material Graph, Material Instance and Anim Graph panels handle while they have focus.
- **Script Console** — one line of Lua at a time, run against the open scene (the live one while
  playing). It has a close button; open it again from the command palette.
- **Reset Layout** — rebuilds the default dock layout described at the top of this chapter.
- **Scene Hierarchy**, **Properties**, **Asset Browser**, **Source Browser** (raw files —
  OBJ, glTF, PNG, HDR — and their import status; import and reimport happen here, and the
  Asset Browser shows only the native assets that come out of it; see
  [Importing Assets](../02-building-worlds/04-importing-assets.md)), **Node Graph**, **Project Settings**,
  **Editor Settings** (per-user preferences — UI scale and others — not saved into the
  project).
- **Performance** — frame time, GPU pass timings, CPU stalls and scene counts. Read-only by
  design: it used to carry a ray-tracing checkbox and a way to select an entity, which made
  a panel you open to measure into one that changes what it measures. Those moved to the
  Engine menu and the viewport respectively. See below.
- **G-Buffer Inspector** — the deferred renderer's intermediate targets, below.
- **Diagnostics** — presentation, capture and editor-state readouts, below.
- **ImGui Demo** — the vendored ImGui's own demo window. Not part of LAB; useful only for
  seeing what a widget looks like in isolation.

## The Performance panel

Frame time (current, average, median, 99th percentile — the one that actually predicts a
hitch — and worst-in-window), a rolling graph, GPU pass time from timestamp queries, the
two CPU stalls in `BeginFrame`, draw calls and triangle counts, cache sizes (material
descriptor sets, cached textures, shared meshes), and render target resolution. **Copy
report** puts all of it on the clipboard as text, for comparing two runs without retyping
numbers off the screen.

**Turn VSync off before you believe any number here.** With it on, everything looks free
because the frame is waiting on the display either way — see Diagnostics, below.

## The G-Buffer Inspector

Shows the deferred renderer's intermediate targets: normal (RGB) + roughness (A), base
colour (RGB) + metallic (A), emissive (RGB) + ambient occlusion (A), velocity (RG, signed —
a stationary image is normally near black), the editor's own unlit overlay layer, and
reversed-Z depth. **Inspect** shows one target at a time at full size; **Mosaic** tiles all
six. Invaluable when something looks wrong and you cannot tell whether the fault is
geometry, normals or lighting — see [Troubleshooting](../07-projects-and-tools/05-troubleshooting.md).

## The Diagnostics panel

- **Presentation** — **VSync** (off is what you want before judging any renderer change;
  on is the default and pins the frame rate to the display refresh) and **HDR10 output**
  (only selectable when the current display and driver advertise it; recreates the
  swapchain).
- **Capture** — RenderDoc status and a **Capture frame** button (`F12` does the same when
  RenderDoc is attached), and a **Take screenshot** button. `F12` is shared between the
  two: with RenderDoc attached it captures a frame, otherwise it writes a screenshot — to
  the Steam screenshot library if Steam is running, otherwise `<project>/screenshots/` as a
  PNG. The button works either way, so you are never without a screenshot just because a
  debugger happens to be attached.
- **Record video**, in the same Capture section, captures the viewport live while you play
  or move the camera. `Shift+F12` starts and stops it, the same as the **Stop recording**
  button; a **Backend** choice and an **FPS** slider sit beside those. The render loop is
  capped to that frame rate while recording, so the capture cadence matches it instead of
  dropping or duplicating frames to compensate. **Auto** tries FFmpeg first (it needs
  `ffmpeg` on `PATH`), then a PNG sequence, then an uncompressed AVI, which needs nothing
  installed at all. While it runs, a red line reports the elapsed time, the frames captured
  and the frames dropped.
- **Export Video...** is the deterministic sibling: a modal that renders a fixed number of
  frames rather than capturing in real time. Width, height, FPS and duration decide the
  result, **Timing** chooses between **Realtime** and **Fixed-step (exact frame count)**, and
  **Output path** takes a file for FFmpeg or AVI and a directory for a PNG sequence, or
  generates a name under `<project>/videos/` when left blank. Capture happens at whatever the
  viewport is actually rendering, and is resampled to the resolution you asked for. A
  progress bar shows the elapsed time, the frame count and the drops, and **Cancel** stops
  it. A new export can start straight away; the previous file finishes closing in the background. Closing the modal does not pause or cancel an export: it keeps advancing every frame
  until it finishes or you cancel it, and the panel remembers the path of the last one.
- **Steam** — connection status, app ID, persona name, and whether the screenshot hook is
  installed. Rich presence is pushed on every change, but what a friend actually *sees*
  depends on that app's localisation table on the Steamworks partner site — see
  [`docs/STEAM.md`](../STEAM.md). None of this needs Steam running; the editor behaves
  identically without it.
- **Editor state** — the open scene's name, the gizmo's hover/use state, and the editor
  camera's position and angles (copy them from here, or from the viewport's **Camera**
  popup).

**Viewport overlays — Show Grid, Show Colliders, DDGI volume and probes, the DDGI-only
debug view, the selection outline and the entity icons — moved to the Overlays button in the
viewport toolbar.** They are not in this panel or the Engine menu; see
[Viewport & Navigation](03-viewport.md).

## The panels

### Viewport

The 3D view, with the play controls and gizmo toolbar across the top. Its own chapter:
[Viewport & Navigation](03-viewport.md).

### Scene Hierarchy

Every entity in the scene as a tree, with parent/child nesting. A **Search** box filters the
list by name, and **Create Entity** adds one; **Primitive...** adds a cube, sphere, plane,
cylinder, capsule or cone from the engine's content, in front of the editor camera (see
[Asset Browser](#asset-browser)). Right-click the empty part of the list for the same entries.
Drag a row onto another to reparent it. Right-click a row for:

- **Edit Object…** — open an `.Lobj` instance for editing (see [Objects](../02-building-worlds/03-objects.md))
- **Unparent** — detach from the current parent, keeping world position
- **Add Child** — create a new entity under this one
- **Save as Object…** — write this entity and its children out as a reusable `.Lobj`
- **Delete Entity**

Toggle with `F5`.

### Properties

Everything about the selected entity: its name, its transform, and one collapsible section
per component. **Add Component** at the bottom offers every component in
[Components](../02-building-worlds/02-components.md)'s reference table.

Right-clicking a component header gives **Copy Component** / **Paste Component**, which is
how you copy a tuned material or light onto another entity without retyping it.

`F4` opens it.

### Asset Browser

A directory tree on the left, the contents of the selected folder on the right. Two tabs above
the tree choose which tree that is:

- **Project** is your project's asset folder. It is where everything you make and import lives, and
  the only place the browser writes to. With no project open there is no Project tab.
- **Engine** is the content that ships with the engine (`Engine/Content`): the cube, sphere, plane,
  cylinder, capsule and cone, the default textures (white, black, grey, flat normal, UV grid, checker),
  the default materials and the default scene. It is **read-only**. You can browse it, search it, see its
  thumbnails and drag it anywhere a project asset goes; the scene, material or field it lands in stores an
  `engine:` reference (`engine:Primitives/Cube.Lmesh`), which resolves on any machine that has the engine.
  Delete, Rename, Cut, New..., Paste and drag-moving into it are greyed out with a tooltip, Reimport and the
  Blender entries are not offered, and a script or an MCP tool that tries the same is refused the same way.
  **Duplicate to Project** (the context menu, or Paste after Copy) copies a file or a folder into the
  project's current folder as assets of their own, with new GUIDs, so you can change them.

Each tab remembers the folder it was in. Favourites and recent assets work in both; an engine item is
remembered as its `engine:` reference. **It lists only native assets** — the things the engine actually loads
at runtime (`.Lmesh`, `.Ltex`, and everything that needs no cooking at all: `.Lscene`,
`.Lobj`, `.lua`, `.Lgraph`, `.Lshader`). A source file you have not imported yet — an
`.obj`, a `.gltf`, a `.png`, an `.hdr` — does not show up here; it lives in the **Source
Browser** instead, and stays there until you import it. See
[Importing Assets](../02-building-worlds/04-importing-assets.md) for the whole workflow.

| Action | Effect |
|---|---|
| Double-click `.Lscene` | Opens that scene |
| Double-click `.Lobj` | Opens that object for editing |
| Double-click `.lua` / `.Lgraph` | Opens it in the node graph / script editor |
| Drag a cooked mesh (`.Lmesh`) into the viewport or the hierarchy | Creates an entity drawing it where you dropped it (one undo step) |
| Drag a texture onto a material slot | Assigns it |
| Drag an `.Lobj` into the viewport | Instantiates it where you dropped it |

**Add > Primitive** needs no asset to drag: the hierarchy's **Primitive...** button, the right-click
menu on the viewport (**Add > Primitive**) and the command palette (`Ctrl+P`, "Add Cube" ...) create an entity
with a mesh component on `engine:Primitives/<Shape>.Lmesh` and the default material, selected, as one undo step.
The viewport menu puts it where you clicked; the others in front of the editor camera. The meshes are unit
sized (a cube is 1 m, a sphere 1 m across, a plane 1 x 1 m, a capsule 2 m tall) and nothing is written into the
project.

Dropped items land where the cursor points, projected onto the ground plane. If you are
looking along the horizon so the ray never meets the ground, the item is placed in front of
the camera instead.

With no project open the editor starts on the engine's default scene
(`engine:Scenes/Default.Lscene`) and the Asset Browser shows the Engine tab only; the Source Browser
has nothing to show until a project is open. The default scene is engine content, so it opens as an
untitled scene and Save asks for a place to put it. A file from the engine's `Engine/Content` folder that is
dropped onto a slot is stored as an `engine:` reference, never as a file path (see
[Projects & Builds](../07-projects-and-tools/01-projects-and-builds.md#engine-content)).

Toggle with `F2`.

### Project Settings

Project name and version, the asset directory, the start scene, **scene physics** (gravity,
gravity scale, terminal velocity), the standalone build's executable name, the runtime
window configuration (title, size, fullscreen, VSync, icon), and the packaging settings
(package only referenced content, strip editor data, additional includes) covered in
[Projects & Builds](../07-projects-and-tools/01-projects-and-builds.md). Note that scene physics is saved *in the
scene*, while everything else on this panel is saved in the `.lab`.

### Editor Settings

Per-user preferences, stored at `%APPDATA%/LAB/editor.yaml` rather than in the project —
opening the same project on another machine, or as another user, starts from the defaults.
Today that file holds a **UI Scale** slider, 0.5x–2.5x with a **Reset** button back to 1.0x:
it scales ImGui's widget spacing and sizing (recomputed from an unscaled snapshot every
frame, so it never compounds) together with text size. While you drag the slider the text is
the current font stretched; when you let go (or type a value, or press Reset) the editor
re-rasterizes its fonts at the new size, so text is sharp at any scale. A fresh install, with
no saved scale, starts at your monitor's display scale (150% gives 1.5x) and keeps following the
monitor until you move the slider. Everything else `EditorSettings` holds — the selection outline style,
Asset/Source Browser layout, Import Settings defaults, the Node Graph style — is edited from
its own panel instead of being duplicated here.

### Source Browser

Raw files — the same directory tree, but showing everything the Asset Browser leaves out:
source models, source images, anything not yet a native asset. Each entry shows its import
status; import and reimport happen from here. See
[Importing Assets](../02-building-worlds/04-importing-assets.md).

### Node Graph

The visual scripting editor. Its own chapter: [Visual Scripting](../06-scripting/01-visual-scripting.md).

The Performance panel, the G-Buffer Inspector and the Diagnostics panel are covered above,
under the Windows menu — that is where they are opened from.

## Logs

LAB writes two log files, both flushed per record so they survive a crash:

- `LAB/logs/LAB.log`
- `LAB/logs/LABEngine.log`

Vulkan validation messages go through the same log. Reading the file is strictly better
than watching the console, which only ever shows you half of it.
