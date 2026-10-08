---
title: "Keyboard Reference"
---

## Editing

Work anywhere in the editor, except while a text field has the keyboard.

| Shortcut | Action |
|---|---|
| `Ctrl+Z` | Undo |
| `Ctrl+Y` or `Ctrl+Shift+Z` | Redo |
| `Ctrl+C` | Copy selection |
| `Ctrl+V` | Paste |
| `Ctrl+D` | Duplicate selection |
| `Ctrl+A` | Select all |
| `Del` | Delete selection *(viewport focused)* |
| `Ctrl+P` | Open the Command Palette — search scenes, panels, workspace presets, overlays, entities and recent assets by name |

While the **node graph** has focus, `Ctrl+Z` / `Ctrl+Y` act on the graph's own history
instead. Editing shortcuts are disabled while playing.

## Files

| Shortcut | Action |
|---|---|
| `Ctrl+N` | New scene |
| `Ctrl+O` | Open scene |
| `Ctrl+S` | Save — the **object** if one is open for editing, otherwise the scene |
| `Ctrl+Shift+S` | Save scene as |

## Viewport camera

Only while the pointer is over the viewport.

| Input | Action |
|---|---|
| Right-drag | Look around |
| Middle-drag | Pan |
| Scroll wheel | Adjust fly speed |
| `W` / `S` | Forward / back |
| `A` / `D` | Left / right |
| `E` / `Q` | Up / down |

## Viewport selection

| Input | Action |
|---|---|
| Left-click | Select |
| Left-click again, same spot | Cycle to the next candidate under the cursor |
| `Shift`- or `Ctrl`-click | Add to selection |
| Left-drag on empty space | Marquee select |
| `Shift`/`Ctrl` + marquee | Add the rectangle to the selection |
| `F` | Frame selection |
| `Esc` | Clear selection |
| `H` | Hide the current selection |
| `Alt+H` | Unhide every hidden entity |

`F` and `Esc` require the viewport to be focused — `F` is an ordinary letter to be typing into
a filter box, and `Esc` closes any open modal first. `H` and `Alt+H` also require the viewport
focused, and additionally do nothing while playing.

## Gizmo

Require the viewport focused and something selected.

| Key | Mode |
|---|---|
| `T` | Translate |
| `R` | Rotate |
| `Y` | Scale |
| `G` | No gizmo |

| Input | Action |
|---|---|
| `Alt` + drag the gizmo | Duplicate and drag the copy |

The gizmo uses `T`/`R`/`Y` rather than `W`/`E`/`R` because `W`/`A`/`S`/`D` are the camera.

## Panels

| Key | Toggles |
|---|---|
| `F2` | Asset Browser |
| `F4` | Properties (opens it) |
| `F5` | Scene Hierarchy |
| `F12` | RenderDoc frame capture when attached, otherwise a screenshot |
| `Shift+F12` | Start or stop a live video recording |

## In a running game

Nothing is reserved except what your scripts read — with one exception:

| Key | Action |
|---|---|
| `Esc` | Releases a captured mouse cursor |

Losing window focus releases it too, so a script that captures the cursor and forgets to
release it cannot trap you.
