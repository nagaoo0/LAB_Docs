---
title: "Lua API Reference"
---

`../09-1-scripting-lua.md` is the chapter: what a script is, how it is attached, and how it
runs. This folder is the reference behind it, one folder per area of the API, so a call can be
looked up rather than remembered.

Every name here is one the engine actually registers. If something is not in these pages, a
script cannot call it.

| Folder | What is in it |
|---|---|
| [`entity/`](entity/) | The `entity` object: identity, transform, hierarchy, material, physics, character, animation, HUD (including terminals, lines and scene captures), particles, sound, splines, AI, lights and meshes (including custom depth and stencil), and adding components |
| [`math/`](math/) | `vec3` and `quat` |
| [`scene/`](scene/) | The `scene` table (including `create_entity`), streaming, picking and `globals` |
| [`timer/`](timer/) | `timer.after`, `timer.every`, `timer.cancel` |
| [`events/`](events/) | `events.on`, `events.off`, `events.emit`: a bus between scripts |
| [`physics/`](physics/) | The `physics` table: rays, casts, overlaps, gravity |
| [`input/`](input/) | Keyboard, mouse, gamepad and project input actions |
| [`ui/`](ui/) | HUD hit testing, the widget event queue, `ui.features`, and the text language |
| [`audio/`](audio/) | Playback and mix buses |
| [`game/`](game/) | Saves, `save_data` tables and the save folder, the blackboard, the window, wall-clock time |
| [`render/`](render/) | Post-process material parameters, and render target (`.Lrt`) assets |
| [`ai/`](ai/) | Behaviour trees and the navmesh |
| [`debug/`](debug/) | Debug draw, and the log |
| [`data/`](data/) | Reading a file out of the project's assets |
| [`native/`](native/) | Calling into a C++ module |
| [`steam/`](steam/) | Steam Input, and the game language Steam runs in |
| [`assets/`](assets/) | Cooking, importing and inspecting assets (editor only) |
| [`testing/`](testing/) | The test harness, the counters it reads, and the renderer's own diagnostics |
| [`editor/`](editor/) | Driving the editor from a test |

## How these pages are written

A page is a list of calls, each as a section: the signature, what it does, what it answers when
it cannot do it, and the trap worth knowing. Examples are complete enough to paste into a
script.

Three conventions run through all of them, and they are the ones that catch people out:

- **Degrees, not radians.** `get_rotation`, `set_rotation`, `rotate` and `quat.from_euler` are
  all in degrees. Only `quat`'s own internals and the C++ side of the engine think in radians.
- **A call that cannot do what it says answers a neutral value rather than raising.** A method
  on an entity with no such component is a no-op, an unknown animator parameter reads `0`, an
  unknown key is never down, an unknown input action never fires. That is deliberate: a script
  generally does not know what its entity carries, and an error every frame would be worse than
  nothing happening.
- **Most calls must be treated as writes to live state.** A transform write goes through the
  same path the editor uses, so physics does not immediately overwrite it, but a value read
  back this frame may not be what the physics step produces at the end of it.

## Where a page disagrees with the C++ reference

[`../C++/`](../../03-cpp/api/) documents the native module API, which is a different, smaller surface: the
ABI a C++ module is given. The Lua API is larger and, in a few places, answers differently on
purpose, notably `physics.raycast`, whose Lua result carries a `distance` the ABI's hit struct
does not have. Where both exist, this folder describes the Lua one.
