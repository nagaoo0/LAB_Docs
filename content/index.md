---
title: "LAB User Manual"
---

How to *use* LAB: open a project, build a scene, light it, make it move, and ship it.

This is the user-facing manual. It assumes you have a working build — if you do not, start
with the README at the root of the LAB repository, which covers prerequisites and compiling.
If you are changing the engine's source rather than using it, read `docs/ARCHITECTURE.md`
in the repository instead.

## Contents

### [Getting Started](01-getting-started/index.md)

| Page | What it covers |
|---|---|
| [First Steps](01-getting-started/01-first-steps.md) | First launch, projects, your first scene |
| [The Editor](01-getting-started/02-editor.md) | Panels, menus, layout, logs |
| [Viewport & Navigation](01-getting-started/03-viewport.md) | Camera, selection, gizmos, snapping |
| [Play Mode](01-getting-started/04-play-mode.md) | Playing, pausing, game cameras |
| [Keyboard Reference](01-getting-started/05-keyboard-reference.md) | Every shortcut on one page |

### [Building Worlds](02-building-worlds/index.md)

| Page | What it covers |
|---|---|
| [Scenes & Entities](02-building-worlds/01-scenes-and-entities.md) | Hierarchy, parenting, undo, saving, runtime scene streaming |
| [Components](02-building-worlds/02-components.md) | Reference for every component, including Scene Capture, UI Line, UI Terminal and custom depth and stencil |
| [Objects (Prefabs)](02-building-worlds/03-objects.md) | Reusable `.Lobj` assets |
| [Importing Assets](02-building-worlds/04-importing-assets.md) | The Source Browser, import settings, skinned meshes and their `.Lskel` and `.Lanim`, reimporting, what ships in a build |
| [Blockout Tools](02-building-worlds/05-blockout-tools.md) | Convex CSG brushes for level design: additive and subtractive brushes, groups, the bake, the agent tools |
| [Blender](02-building-worlds/06-blender.md) | The LAB Bridge add-on (Send Selection, Send Scene, export on save), Open in Blender, headless jobs for LODs and convex hulls, the coordinate rule |

### [Rendering](03-rendering/index.md)

| Page | What it covers |
|---|---|
| [Materials & Textures](03-rendering/01-materials.md) | Colour, PBR maps, emissive, render targets (live feeds), material graphs, post-process materials |
| [Lighting & Shadows](03-rendering/02-lighting.md) | Light types, shadows, sky, HDRI, DDGI |
| [Hardware Ray Tracing](03-rendering/03-ray-tracing.md) | Traced shadows, AO, reflections and GI; ray budgets and device tiers |

### [Gameplay](04-gameplay/index.md)

| Page | What it covers |
|---|---|
| [Physics](04-gameplay/01-physics.md) | Rigid bodies, colliders, triggers, gravity, character controllers, joints |
| [Animation](04-gameplay/02-animation.md) | Animation graphs: states, blend spaces, IK, transitions, root motion, sockets |
| [Navigation and AI](04-gameplay/03-navigation-and-ai.md) | Baking a navmesh, Nav Agents, AI Controllers, behaviour trees |
| [Networking](04-gameplay/04-networking.md) | Hosting and joining, the Network panel, launching clients, the `net` API, RPC, Steam lobbies, the limits of v1 |

### [User Interface](05-ui/index.md)

| Page | What it covers |
|---|---|
| [HUD (On-screen UI)](05-ui/01-hud.md) | On-screen elements: UI Transform, Sprite, Text, UI Line, rich text and links, live camera feeds, hit testing |
| [UI Designer](05-ui/02-ui-designer.md) | The canvas panel for laying out the HUD: selecting, moving, anchors, the Create palette and widgets, the 9-slice margin editor |
| [UI Widgets](05-ui/03-ui-widgets.md) | The per-frame UI event queue, Button, Toggle, Slider, text field, Terminal and the layout containers |

### [Scripting](06-scripting/index.md)

| Page | What it covers |
|---|---|
| [Visual Scripting](06-scripting/01-visual-scripting.md) | Node graphs |
| [Lua Scripting](06-scripting/02-lua/index.md) | The script API, with a [reference page for every call](06-scripting/02-lua/api/index.md) |
| [C++ Scripting](06-scripting/03-cpp/index.md) | Native modules: gameplay as C++ the engine loads, and the [whole module API](06-scripting/03-cpp/api/index.md) |

### [Projects & Tools](07-projects-and-tools/index.md)

| Page | What it covers |
|---|---|
| [Projects & Builds](07-projects-and-tools/01-projects-and-builds.md) | `.lab`, the save location, standalone builds, CLI |
| [Testing](07-projects-and-tools/02-testing.md) | The Lua test harness |
| [Plugins](07-projects-and-tools/03-plugins.md) | Installing and enabling editor plugins, the Plugin Manager and Plugins menu, plugins in a game, starting your own, and the Terrain plugin |
| [AI Agents in the Editor (MCP)](07-projects-and-tools/04-editor-ai-agents.md) | Let Claude or another AI agent build in the editor: setup, tools, limits |
| [Troubleshooting](07-projects-and-tools/05-troubleshooting.md) | When something looks wrong |

## Conventions used throughout

- **LAB is Z-up.** Z is vertical, X is right, Y is forward. Gravity is `(0, 0, -9.81)`.
- **Angles are degrees in the editor and in Lua**, radians only inside the `.Lscene` files.
- **Content paths are project-relative.** A scene stores `models/cube.obj`, never a path
  with a drive letter on it, which is what makes a project folder movable.
- Units are metres, kilograms and seconds. Nothing enforces that, but the physics
  defaults assume it.
