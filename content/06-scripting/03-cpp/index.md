---
title: "C++ Scripting (native modules)"
---

Lua covers most gameplay. This chapter is for the rest: code that has to be fast, code that
has to talk to a library the sandbox will not open, or a game whose core logic you would
rather write in a language you can put a debugger on.

C++ gameplay ships as a **native module**: a shared library (`.dll` on Windows, `.so`
elsewhere) that the engine loads at run time, exactly the way it loads a `.lua` file. You put
a **Native Script** component on an entity, name a class the module registered, and press
Play. Each entity gets its own instance of that class, with its own members.

The module is ordinary C++ compiled by your own compiler. It does not link the engine, does not
include an engine header, and nothing crossing the boundary between them is an `entt` handle, a
`glm` type or a `std::string`. It talks to the engine through one pure C header,
[`LABNativeAPI.h`](../../../LAB/src/Scene/ScriptAPI/LABNativeAPI.h), and a header-only
convenience layer on top of it,
[`LABNative.hpp`](../../../LAB/src/Scene/ScriptAPI/LABNative.hpp). The convenience layer is
module-side only, so it is free to use C++ and the standard library however it likes.

## Lua or C++?

| | Lua | C++ module |
|---|---|---|
| Iterate on | save the file, replay | rebuild, replay |
| Speed | fine for a hundred entities | fine for a hundred thousand |
| Debugging | `log.info`, the console | breakpoints, step, watch |
| Third-party libraries | none, the sandbox is closed | anything you can link |
| Ships as | a text asset | a compiled library per platform |
| If it throws | that instance is disabled | that instance is disabled |

Both are first-class, and both run in the same scene at the same time. The usual split is a
module for the systems (movement, combat, AI) and Lua for the one-off (a door that opens when
three switches are lit). The two can call each other in either direction, which
[Lua interop](api/lua/01-lua-interop.md) covers.

Start with Lua. Move to C++ when you have measured something that needs it, or when you want
a debugger more than you want a fast edit loop.

## What you need

- The engine source tree, because the module builds against its headers.
- A C++ compiler: MSVC 2022 on Windows, GCC or Clang on Linux.
- `premake5`, which the engine vendors at `vendor/bin/premake/`. The module's build script
  finds it through the `LAB_ENGINE_ROOT` environment variable, so the same script builds on
  another machine.

Nothing else. A module does not need the Vulkan SDK, and it does not need the engine to be
built first.

## Quick start

### 1. Create the module

Creating a project in this editor writes `Source/` for you. For a project that already
exists, open its `.lab` file and use **Build → Game Module…**, which offers **Create sample
source** when there is no `Source/` directory yet. Three files appear beside the project
file:

| File | What it is |
|---|---|
| `Source/Build-<Module>.lua` | The premake script |
| `Source/<Module>.cpp` | A sample behaviour called `Bobber`, with two fields |
| `Source/.gitignore` | Ignores `Bin/`, `Bin-int/` and generated project files |

`<Module>` is the project name with runs of non-alphanumeric characters collapsed to an
underscore, plus `Module`. A project called *Tiny Lab* gets `Tiny_LabModule`.

### 2. Write a behaviour

A behaviour is a class deriving from `LABNative::Behaviour`, overriding the callbacks it
needs. This is the sample the editor writes:

```cpp
#include "LABNative.hpp"

#include <cmath>

namespace {

class Bobber : public LABNative::Behaviour
{
public:
	// Fields an author can set. The values here are what an entity with nothing
	// authored keeps; the field table below is what makes the inspector offer them.
	float Speed = 1.0f;
	float Height = 0.5f;

	void OnCreate() override
	{
		m_Origin = GetPosition();
		m_Phase = m_Origin.x;
	}

	void OnUpdate(float deltaTime) override
	{
		m_Elapsed += deltaTime * Speed;
		LAB_Vec3 position = m_Origin;
		position.z += Height * 0.5f * (1.0f + std::sin(m_Elapsed * 2.0f + m_Phase));
		SetPosition(position);
	}

private:
	LAB_Vec3 m_Origin{};
	float m_Elapsed = 0.0f;
	float m_Phase = 0.0f;
};

} // namespace
```

Four things worth knowing before you write more than that:

- **The entity id is not valid in a constructor.** The engine constructs the instance and
  assigns the id immediately afterwards, so a constructor runs before the entity exists.
  Anything that needs the entity belongs in `OnCreate`.
- **Override only what you need.** Every callback has an empty default. The full set is under
  [The behaviour lifecycle](#the-behaviour-lifecycle).
- **`OnUpdate` is the frame; `OnFixedUpdate` is the physics step.** They are different
  clocks, and the difference matters the moment you move a rigid body.
- **A behaviour runs on the main thread and must not block.** Spawning, destroying (your own
  entity included) and timing a span are all safe from inside a callback. Waiting on a file,
  a socket or a thread is not.

### 3. Register it

One exported function registers everything the module offers:

```cpp
extern "C" LAB_NATIVE_EXPORT int LAB_ModuleInit(const LAB_EngineAPI* api, LAB_ModuleRegistry* registry)
{
	LABNative::Init(api);
	LAB_REGISTER_BEHAVIOUR(registry, Bobber);
	return LAB_NATIVE_API_VERSION;
}
```

The name in the inspector is the argument, not the C++ type name. `LAB_REGISTER_BEHAVIOUR`
uses the type name; call `LABNative::RegisterBehaviour<Bobber>(registry, "Float")` to choose
another. `LAB_NATIVE_API_VERSION` must be returned exactly as the header defines it, never a
number typed in by hand: see [ABI and versioning](api/module/02-abi-and-versioning.md) for
why a hard-coded value is a lie about what the module may call.

### 4. Build it

**Build → Game Module…** compiles it and reports any compiler errors in a clickable list.
From a terminal, the generated script is self-describing:

```
Windows:
  set LAB_ENGINE_ROOT=C:\path\to\MadLab
  %LAB_ENGINE_ROOT%\vendor\bin\premake\Windows\premake5.exe --file=Build-MyModule.lua vs2022
  msbuild MyModule.sln -p:Configuration=Release -p:Platform=x64 -p:PreferredToolArchitecture=x64
Linux:
  LAB_ENGINE_ROOT=/path/to/MadLab "$LAB_ENGINE_ROOT/vendor/bin/premake/Linux/premake5" --file=Build-MyModule.lua gmake2
  make config=release -j"$(nproc)"
```

The library lands at `Source/Bin/<Module>.dll` (`Source/Bin/<Module>.so` on Linux).

**Build the module in the same configuration as the runtime that loads it.** The two share
one C runtime, which is the one rule here whose violation is not a clean error: a Debug
module under a Release runtime corrupts the heap somewhere that points at nothing useful. The
editor's dialog defaults to its own configuration for that reason, and warns when the two
disagree.

### 5. Tell the project to load it

**Use as this project's Native Module** in the build modal writes the setting and saves the
project. By hand it is one line in the `.lab` file:

```yaml
NativeModule: Source/Bin/MyModule.dll
```

It resolves against the project directory first, then beside the executable, which is what a
packaged build relies on: the build stages the module beside the runtime and names it there in
the packaged project file.

With no extension (`Source/Bin/MyModule`) it means this platform's library, `.dll` on Windows
and `.so` on Linux, so a project opened on both can keep one setting.

### 6. Attach it and press Play

Select the entity, **Add Component → Native Script**, and pick the class. Once a module is
loaded the names it registered are a combo, because a typo here is a behaviour that silently
never runs.

The line under the class rows reports which of three states you are in: no module loaded, a
class the module does not register, or a module running N behaviours with M disabled. Read
it. It is the difference between a behaviour that works and a scene that looks fine and is
inert.

## The behaviour lifecycle

| Callback | Called |
|---|---|
| `OnCreate()` | Once, right after construction and after any authored field values are applied |
| `OnFixedUpdate(float dt)` | Once per physics step the solver took, with the fixed timestep |
| `OnUpdate(float dt)` | Every frame while playing, after the physics step |
| `OnCollisionBegin(other)` | A solid contact started. `other` is the entity on the far side |
| `OnCollisionEnd(other)` | A solid contact ended |
| `OnOverlapBegin(other)` | Something entered a trigger volume |
| `OnOverlapEnd(other)` | Something left a trigger volume |
| `OnDestroy()` | Once, when the entity is destroyed or play stops, before the instance is freed |

`OnFixedUpdate` runs *before* that frame's `OnUpdate`, and it runs once before each physics
step rather than once per frame: a frame that took 33 ms took two steps and gets two
`OnFixedUpdate` calls, each ahead of its own step, so what a call applies is what that step
integrates (ABI minor 13; earlier engines ran every call after the frame's steps). Note the
consequence: a scene with no rigid bodies takes no steps at
all, so `OnFixedUpdate` never runs there. If you need a fixed-rate tick that is independent
of the simulation, that does not exist yet, and `OnUpdate` with your own accumulator is the
honest answer.

Anything that moves a rigid body belongs in `OnFixedUpdate`. Anything that reads input or
drives a HUD belongs in `OnUpdate`.

## Fields an author can set

A behaviour can offer values an author edits in the inspector. Declare the members, then a
table naming each one and where it lives:

```cpp
class Tuner : public LABNative::Behaviour
{
public:
	float Speed = 1.5f;
	int Steps = 3;
	bool Enabled = true;
	LAB_Vec3 Offset{ 9.0f, 9.0f, 9.0f };
};

// Namespace scope: the engine keeps this pointer for as long as the module is loaded.
const LAB_FieldDesc kTunerFields[] = {
	LABNative::Field("Speed", LAB_FIELD_FLOAT, LABNative::FieldOffset(&Tuner::Speed)),
	LABNative::Field("Steps", LAB_FIELD_INT, LABNative::FieldOffset(&Tuner::Steps)),
	LABNative::Field("Enabled", LAB_FIELD_BOOL, LABNative::FieldOffset(&Tuner::Enabled)),
	LABNative::Field("Offset", LAB_FIELD_VEC3, LABNative::FieldOffset(&Tuner::Offset)),
};

LABNative::RegisterBehaviour<Tuner>(registry, "Tuner", kTunerFields, 4);
```

The types that can cross are `float`, `int`, `bool` and `LAB_Vec3`. A string or an asset path
is not among them yet, because it would have to cross as something two compilers agree on the
representation of.

Two rules that are easy to get wrong:

- **`LABNative::FieldOffset`, not `offsetof`.** A behaviour inherits its callbacks, which
  makes it polymorphic, so the standard library's `offsetof` is not required to work on it.
  `FieldOffset` takes a pointer to member: the same arithmetic with the type checked.
- **The field table must outlive the module.** A namespace-scope array does. A local one does
  not, and the engine keeps the pointer.

Values are applied after the instance is constructed and before `OnCreate`, so the first
callback already sees them. A field nothing authored keeps whatever your constructor gave it.
A value that will not parse is reported and skipped rather than written as a zero, and a
value for a field the module no longer declares is kept rather than dropped, so changing
which fields you declare does not erase what an author had set.

The full rules, including what the inspector does with them, are in
[Declared fields](api/module/04-declared-fields.md).

## Talking to Lua

A behaviour can call a Lua function by name, which is the same place a node graph's **Call
Lua Function** node looks, so both reach the same functions:

```cpp
char buffer[256];
LAB_Variant out{};
out.StringBuffer = buffer;
out.StringCapacity = sizeof(buffer);

LAB_Variant args[1]{};
args[0].Type = LAB_VARIANT_NUMBER;
args[0].Number = 42.0;

if (LABNative::Lua::CallFunction("on_score_changed", args, 1, &out))
	LABNative::LogInfo(buffer);
```

And Lua can call the module, through a function registered at init:

```cpp
static void SetDifficulty(const LAB_Variant* args, int argCount, LAB_Variant* out)
{
	// ...
	out->Type = LAB_VARIANT_BOOL;
	out->Bool = 1;
}

registry->RegisterFunction(registry->Self, "set_difficulty", &SetDifficulty);
```

```lua
native.call("set_difficulty", 3)
```

What crosses is a `LAB_Variant`: a tag plus the value, in one fixed-size struct. It is not a
`std::variant` and it is not the engine's own `AI::Value`, because either of those would tie
the module to the engine's compiler and STL version. The rules, including which side owns a
string, are in [Lua interop](api/lua/01-lua-interop.md).

## When something goes wrong

| Symptom | What it means |
|---|---|
| The inspector says *no module loaded* | The library is not where the project says it is, or it was refused. The log names which |
| A class is unknown | The name on the entity does not match what the module registered. `native_classes()` lists what it did register |
| The scene is inert, no errors | The entity has no correct class name, or the module built in a different configuration than the runtime |
| A behaviour stops working mid-play | It threw. The engine catches, logs and disables that instance; its siblings keep running |
| The process dies | A hard fault. The crash log names the behaviour and the phase that was running |

A disabled behaviour still gets `OnDestroy`, and `native_disabled_count()` reports how many
have been switched off. Nothing is retried: a behaviour that threw once is not called again,
because the usual cause is a state it will still be in next frame.

The module is ordinary native code in the engine's process, so the debugger works. Attach it,
set a breakpoint in the behaviour, press Play. This is the second reason to build in the same
configuration as the runtime: it is also what gets you symbols.

## Packaging

**Build Standalone** stages the module beside the runtime executable and deliberately keeps it
out of `Game.Lpak`, the same treatment the Steam library gets: `LoadLibrary` and `dlopen` want
a path, not an entry in an archive.

A `NativeModule` setting that resolves to nothing fails the build rather than warning. That is
deliberate. Without the module there is no gameplay at all, so a build that quietly omitted it
would be a build that runs and does nothing.

The dependency walker cannot see a behaviour loading an asset by a string, so name those
assets in `Build.AdditionalIncludes` or the packaged build will not contain them.

## Testing a module

The same Lua harness everything else uses. A test loads the module itself and asserts on the
running behaviour:

```lua
-- scene: scenes/native_module_test.Lscene

function on_create()
	-- No extension: the host appends this platform's own (.dll or .so).
	loaded = load_native_module("MyModule")
end

function on_finish()
	expect(loaded, "the module was loaded")
	expect(native_behaviour_count() == 1, "one behaviour instantiated")
	expect(script_failures() == 0, "nothing failed")
	expect(validation_errors() == 0, "no Vulkan validation errors")
end
```

The readers a native test leans on: `native_classes()`, `native_behaviour_count()`,
`native_disabled_count()`, `native_module_loaded()`, `script_failure_details()`, `cpu_zones()`,
and the entity method `native_fields()`. The engine's own fixture,
[`Dev/Tests/native/TestGameModule/TestGameModule.cpp`](../../../Dev/Tests/native/TestGameModule/TestGameModule.cpp),
has a behaviour per capability and one test per behaviour.

## The API, page by page

Every function the engine offers a module, grouped the way the header groups it. The table
below is the whole reference in one place; [`C++/README.md`](api/index.md) is the same set of
pages as a folder index, which is where to start if you would rather browse than scan a table.

| Area | Page | What is there |
|---|---|---|
| Module | [Module lifecycle](api/module/01-module-lifecycle.md) | `LAB_ModuleInit`, registration, the behaviour descriptor and its callbacks |
| Module | [ABI and versioning](api/module/02-abi-and-versioning.md) | Major and minor, the `HAS_ABI` macros, what a refusal means |
| Module | [Building and loading](api/module/03-building-and-loading.md) | premake, `NativeModule`, where the library is looked for, hot reload |
| Module | [Declared fields](api/module/04-declared-fields.md) | Field types, offsets, when values are applied |
| Module | [Logging and time](api/module/05-logging-and-time.md) | `LogInfo`/`LogWarn`/`LogError`, `ElapsedTime`, `FrameIndex`, `FixedDeltaTime` |
| Entities | [Entity queries](api/entity/01-entity-queries.md) | `FindEntityByName`, `IsEntityValid`, `GetName`, `DestroyEntity` |
| Entities | [Transform](api/entity/02-transform.md) | Position, rotation, scale, world space |
| Entities | [Hierarchy and spawning](api/entity/03-hierarchy-and-spawning.md) | `SetParent`, `GetChildren`, `GetParent`, `SpawnEntity` |
| Entities | [Components](api/entity/04-components.md) | `GetComponent`, `SetComponent` and the mirror structs |
| Input | [Input](api/input/01-input.md) | Keys, actions, mouse |
| Physics | [Physics](api/physics/01-physics.md) | `Raycast`, forces, impulses, velocity |
| Audio | [Audio](api/audio/01-audio.md) | Sounds and bus volumes |
| Animation | [Animation and particles](api/animation/01-animation-and-particles.md) | Animator parameters, emitters |
| AI | [Navmesh and behaviour trees](api/ai/01-navmesh-and-behaviour-trees.md) | `MoveTo`, blackboard reads and writes, baking |
| Splines | [Splines](api/spline/01-splines.md) | Length, frames, closest point |
| UI | [HUD elements](api/ui/01-hud-elements.md) | Text, colour, layout, borders |
| UI | [HUD interaction](api/ui/02-hud-interaction.md) | Hover, press, click, the virtual pointer |
| UI | [HUD widgets](api/ui/03-widgets.md) | Widget events, values, text fields, focus |
| Streaming | [Scene streaming](api/streaming/01-scene-streaming.md) | Preload, append, open |
| Save/load | [Blackboard](api/savegame/01-blackboard.md) | The `GameState` key/value store |
| Save/load | [Save slots](api/savegame/02-save-slots.md) | Checkpoints and profile saves |
| Lua | [Lua interop](api/lua/01-lua-interop.md) | `LAB_Variant`, calling Lua, being called |
| Profiling | [Profiling](api/profiling/01-profiling.md) | `ZoneAdd` and `ZoneScope` |

## The API, by example

The engine's fixture has a behaviour per capability, and a Lua test holding each one to
account. Read the pair when a page above stops short of what you need.

| What you want to do | Behaviour in `TestGameModule.cpp` | Test |
|---|---|---|
| Move an entity, log on creation | `Mover` | `native_module.lua` |
| Destroy mid-play, run `OnDestroy` | `SelfDestruct`, `DestroyMarker` | `native_destroy.lua` |
| React to collisions and triggers | `ContactCounter` | `native_contacts.lua` |
| Step with physics, read the engine clock | `FixedStepper` | `native_time.lua` |
| Read and write a component | `ComponentProbe` | `native_components.lua` |
| Reach a light on another entity | `LightProbe` | `native_lights.lua` |
| Spawn an object and parent it | `Spawner` | `native_spawn_api.lua` |
| Audio, animator, particles, parent | `GameplayProbe` | `native_gameplay.lua` |
| Declare authorable fields | `Tuner` | `native_fields.lua` |
| Watch a throwing behaviour be disabled | `Thrower` | `native_exception.lua` |

## Where to go next

- [Native scripting](../../NATIVE_SCRIPTING.md) is the design document: the boundary
  contracts, threading, the ABI changelog, and how a module is debugged.
- [Native API reference](../../NATIVE_API_REFERENCE.md) is the entry list, generated from the
  ABI header, for looking a name up rather than reading about it.
- [Writing a native gameplay module](../../NATIVE_TUTORIAL.md) walks the same ground as this
  chapter with more of the reasoning shown.
- [Lua scripting](../02-lua/index.md) is the other half of this chapter, and most projects
  want both.
