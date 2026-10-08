---
title: "C++ API Reference"
---

Every function a native module can call, and everything it has to provide, grouped the way
[`LABNativeAPI.h`](../../../../LAB/src/Scene/ScriptAPI/LABNativeAPI.h) groups it.

| Folder | What is in it |
|---|---|
| [`module/`](module/) | The module itself: init, registration, the ABI, building and loading, declared fields, logging and the engine's clocks |
| [`entity/`](entity/) | Finding entities, transforms, hierarchy, spawning, components |
| [`input/`](input/) | Keys, project actions, the mouse |
| [`physics/`](physics/) | Raycasts (and `RaycastEx`), forces, impulses, velocity, spin, the solver's own body position |
| [`audio/`](audio/) | Sounds, per-voice volume and pitch, and bus volumes |
| [`net/`](net/) | Tick rate, interpolation delay, a client's address, dropping a client |
| [`animation/`](animation/) | Animator parameters and particle emitters |
| [`ai/`](ai/) | Nav agents, behaviour trees, baking a navmesh |
| [`spline/`](spline/) | Spline queries in world space |
| [`ui/`](ui/) | The HUD: elements, layout, interaction, widgets (events, values, text fields, focus) |
| [`streaming/`](streaming/) | Preloading, appending and opening scenes |
| [`savegame/`](savegame/) | The `GameState` blackboard, checkpoints and profile saves |
| [`lua/`](lua/) | Calling Lua, and being called from Lua |
| [`profiling/`](profiling/) | Timing your own spans |

## How to read a page

Each function is documented twice, because there are two ways to reach it:

- The **ABI** form is a plain C function pointer on `LAB_EngineAPI` or one of its sub-tables.
  That is what crosses between the engine and the module.
- The **C++** form is the wrapper in
  [`LABNative.hpp`](../../../../LAB/src/Scene/ScriptAPI/LABNative.hpp), which is what you
  actually write. It forwards to the ABI form and returns the type that reads best: an entry
  the C ABI spells as an `int` because plain C has no `bool` is a `bool` here.

Where a function is a method on `LABNative::Behaviour`, the page says so, and that spelling
works on the behaviour's own entity without passing an id.

Two conventions run through every page:

- **Entity ids are UUIDs**, the same ones a Lua script or a scene file would name an entity
  by. `0` is never a valid entity, so it doubles as "nothing" wherever a call returns one.
- **Every entity call is tolerant.** With no bound scene, or an entity id that does not
  resolve, a call is a silent no-op, or answers the type's zero value. Asking the wrong entity
  is not a failure and is not logged.

## The two headers

| Header | Who includes it | Why |
|---|---|---|
| `LABNativeAPI.h` | Both sides | Pure C, no engine includes, stable across compilers and STL versions |
| `LABNative.hpp` | The module only | Header-only C++ sugar. It never crosses into the engine binary |

A module includes `LABNative.hpp` and nothing else from the engine. Including an engine header
would tie the module to the engine's compiler, its STL version and its exact type layout,
which is the thing this boundary exists to avoid.

Reading the rest of the manual:

- [C++ scripting](../index.md) is the chapter: when to use a module, how to
  create one, and the lifecycle.
- [`docs/NATIVE_SCRIPTING.md`](../../../NATIVE_SCRIPTING.md) is the design document, for the
  reasoning behind the boundary rather than the calls through it.
