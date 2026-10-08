---
title: "First Steps"
---

## Launching the editor

Run `LAB.exe` from the `LAB/` folder — **the working directory should be `LAB/`**,
because the sample project, the logs and the saved window layout are relative to it. (The
engine's own files, the shaders and content included, are not: they are found from the
executable's location, wherever the editor is started.) If you launch from Visual Studio, the
project is already configured this way.

You can also pass a project on the command line:

```bash
LAB.exe path/to/MyGame.lab
```

See [Projects & Builds](../07-projects-and-tools/01-projects-and-builds.md) for every command-line option.

## Opening a project

A **project** is a `.lab` file plus an asset folder beside it. Everything you make —
scenes, models, textures, scripts, objects — lives under that asset folder, and scenes
reference content by paths relative to it. That is what lets you move or zip a project
folder without anything breaking.

From the menu bar:

- **File → New Project…** — pick an empty folder; LAB writes the manifest and an
  `assets/` folder.
- **File → Open Project…** — pick an existing `.lab`.
- **File → Open Recent** — the last ten projects you opened. This list is stored per user
  in `%APPDATA%/LAB/editor.yaml`, not in any project, so it never ends up in version
  control.

Without a project open, the editor starts on the engine's default scene (a ground plane
with the checker, a cube, a sun and a sky). That scene belongs to the engine and is
read-only: Save asks where to put your copy. It works for poking around, but make a project
before doing anything you want to keep.

The engine also ships a little content of its own: the cube, sphere, plane, cylinder,
capsule and cone, a few plain textures and materials. A scene refers to them as
`engine:Primitives/Cube.Lmesh` and so on, which means a new project needs no copy of a cube
to have one. See [Projects & Builds](../07-projects-and-tools/01-projects-and-builds.md#engine-content).

The sample project shipped with the repository is `LAB/SampleProject.lab`.

## Your first scene

1. **File → New Scene** (`Ctrl+N`).
2. Press **Create Entity** in the **Scene Hierarchy** panel, or drag a model out of the
   **Asset Browser** into the viewport.
3. Select it. The **Properties** panel shows its components; **Add Component** adds more.
4. Give the scene a light — without one everything renders black. Add an entity with a
   **Directional Light** and, usually, an **Ambient Light** alongside it.
5. **File → Save Scene** (`Ctrl+S`). Scenes are `.Lscene` files, saved under your asset
   folder — `scenes/` by convention.

A minimum viable scene is: one entity with a **Mesh** and a **Material**, one entity with a
**Directional Light**, and one entity with a **Camera** if you intend to press Play.

## Moving around

With the mouse over the viewport:

| Input | Does |
|---|---|
| Right-drag | Look around |
| Middle-drag | Pan |
| Scroll wheel | Change fly speed |
| `W` `A` `S` `D` | Fly forward / left / back / right |
| `E` / `Q` | Fly up / down |
| `F` | Frame the selection |

Full details in [Viewport & Navigation](03-viewport.md).

## Where things live

```
MyGame/
  MyGame.lab          the manifest: name, asset folder, start scene, window settings
  assets/
    scenes/    *.Lscene      scenes
    models/    *.obj     meshes
    textures/  *.png     *.hdr
    scripts/   *.lua     *.Lgraph
    objects/   *.Lobj   reusable objects (prefabs)
```

Nothing forces those subfolder names — the asset browser shows whatever is there — but
they are what the samples use and what the engine's own file dialogs default to.

## Next

- [The Editor](02-editor.md) — what every panel does.
- [Components](../02-building-worlds/02-components.md) — what you can attach to an entity.
