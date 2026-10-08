---
title: "Scenes & Entities"
---

## What a scene is

A scene is a flat list of **entities**, each of which is a name, a transform, and whatever
**components** you have attached. There are no classes to derive from and no fixed entity
types — a light is an entity with a light component, a camera is an entity with a camera
component, and anything can have both.

Scenes are saved as `.Lscene` files: readable YAML, one block per entity, one sub-block per
component. They live under your project's asset folder, `scenes/` by convention.

```yaml
Scene: Script test
Entities:
  - Entity: 0
    TagComponent:
      Tag: Cube
    TransformComponent:
      Position: [0, 0, 0]
      Rotation: [0, 0, 0]     # radians in the file; the editor shows degrees
      Scale: [1, 1, 1]
    MeshComponent:
      MeshPath: models/cube.obj
    MaterialComponent:
      Color: [1, 1, 1, 1]
      AlbedoTexturePath: textures/checker.png
```

Being plain YAML, scenes diff and merge sensibly in version control. You can hand-edit one,
though the editor is generally faster and will not typo a component name.

## Entities

### Creating

- Press **Create Entity** in the **Scene Hierarchy**.
- Right-click an existing row → **Add Child** to create one parented to it.
- Drag a model out of the Asset Browser into the viewport — you get an entity with a mesh
  already assigned.
- Drag an `.Lobj` in — you get a whole instantiated object. See [Objects](03-objects.md).

### Naming

Every entity has a **Tag**, edited at the top of the Properties panel. Names need not be
unique, but `scene.find("Name")` in a script returns the first match, so duplicates make
scripting ambiguous.

### Identity

Every entity also has a hidden **ID**: a stable number that survives save, load, copy, undo
and play mode. This is what `entity:get_id()` returns and `scene.find_by_id()` takes. Prefer
it over names in scripts — renaming an entity breaks a `find` and nothing tells you.

Duplicating an entity mints a *new* id, because the original is still alive; undoing a
delete restores the *original* id, because nothing else is using it. Both are what you want
and neither needs any thought from you.

## Hierarchy and parenting

Drag one row onto another in the hierarchy to parent it. Right-click → **Unparent** detaches
it again.

**Transforms are local.** An entity's transform is relative to its parent; its world
transform is that composed with every ancestor's. Move a parent and the children come along.

Parenting is also how you build compound objects — a turret that is a base, a mount and a
barrel is three entities in a chain, each rotating in its parent's frame.

## Selection

The selection lives in the viewport chapter, [Viewport & Navigation](../01-getting-started/03-viewport.md), but
the hierarchy participates: clicking a row selects that entity, and `Shift`-clicking selects
the range between the last click and this one (rows have an order, which is why `Shift`
means range here and *add* in the viewport).

## Editing

| Action | Shortcut |
|---|---|
| Copy | `Ctrl+C` |
| Paste | `Ctrl+V` |
| Duplicate | `Ctrl+D` |
| Delete | `Del` (viewport focused) |
| Select all | `Ctrl+A` |
| Undo | `Ctrl+Z` |
| Redo | `Ctrl+Y` or `Ctrl+Shift+Z` |

Copy/paste and duplicate carry the whole entity — every component, and its children.

## Undo

Undo is snapshot-based: each action records the state of the entities it touched. It covers
component edits, transforms, creation, deletion and reparenting, and the Edit menu labels
each step with what it will actually undo.

Two things to know:

- **Undo is disabled while playing.** Play mode simulates a copy of the scene; undoing there
  would edit something that is about to be thrown away.
- **The node graph has its own undo stack.** While the graph editor has focus, `Ctrl+Z`
  undoes a node edit, not an entity edit. They are different documents.

## Saving

`Ctrl+S` saves whatever is open — the scene normally, or the object if you have one open for
editing. `Ctrl+Shift+S` is Save As.

**File → Set Current Scene As Start Scene** marks the open scene as the one a standalone
build (and a reopened project) starts in.

## Scene-level settings

A scene stores its own physics settings — gravity, gravity scale and terminal velocity — in
the `.Lscene` file. Edit them in **Project Settings → Scene Physics**. See
[Physics](../04-gameplay/01-physics.md).

## Scene streaming (runtime)

A running game can switch scenes on its own, without going through the editor's File menu —
this is what a level transition, a loading screen, or streaming in a sub-area is built on.
From a script or a node graph:

- **`scene.preload_scene(path)`** parses and caches a scene file ahead of time, so a later
  switch to it does not stall on disk I/O.
- **`scene.append_scene(path)`** adds that scene's entities into the one already running,
  alongside whatever is already there.
- **`scene.open_scene(path)`** replaces the running scene wholesale.

`open_scene` **destroys every entity that is not marked persistent** first. Tick
**Persistent** (or call `entity:set_persistent(true)` before the switch) on anything that
has to survive it — a game manager, a player controller, a HUD. A persistent entity whose
parent is about to be destroyed is detached to the scene root rather than taken down with
its parent, keeping its world placement. `append_scene` never destroys anything, so
persistence has no effect there.

This is a runtime-only mechanism: it has no effect in the editor outside Play mode, and it
is separate from **File → Open Scene**, which is an authoring action on the scene you have
open for editing. See [Lua Scripting](../06-scripting/02-lua/api/scene/02-streaming.md) for the full API
and [Visual Scripting](../06-scripting/01-visual-scripting.md#scene-streaming) for the equivalent nodes.
