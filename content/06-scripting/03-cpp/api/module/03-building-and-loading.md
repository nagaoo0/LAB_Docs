---
title: "Building and loading a module"
---

How the library is produced, what configuration it must be, and where the engine looks for it.

## The three ways to build

### From the editor

**Build → Game Module…** compiles the project's `Source/` module with the engine's premake5 and
MSBuild, or `make` on Linux, in the configuration the editor was built in. Compiler errors are
parsed into a clickable list, so a build failure is a list of lines rather than a wall of text.

The same modal writes the module source when a project has none yet, and offers to set the
project's **Native Module** setting to the library it produced.

### From a terminal

The generated `Source/Build-<Module>.lua` is self-describing, and the engine's own build runner
does the same work as the dialog:

```powershell
scripts/Build-GameModule.ps1 -SourceDir C:\game\Source -PremakeFile Build-GameModule.lua `
    -Config Release -EngineRoot C:\dev\MadLab
```

```bash
scripts/linux/Build-GameModule.sh
```

### By hand

```
Windows:
  set LAB_ENGINE_ROOT=C:\path\to\MadLab
  %LAB_ENGINE_ROOT%\vendor\bin\premake\Windows\premake5.exe --file=Build-MyModule.lua vs2022
  msbuild MyModule.sln -p:Configuration=Release -p:Platform=x64 -p:PreferredToolArchitecture=x64
Linux:
  LAB_ENGINE_ROOT=/path/to/MadLab "$LAB_ENGINE_ROOT/vendor/bin/premake/Linux/premake5" --file=Build-MyModule.lua gmake2
  make config=release -j"$(nproc)"
```

The engine root comes from the environment rather than being baked into the script, so the same
file builds on another machine and in CI.

## Configuration: the one rule

**A module is always built in the configuration of the runtime that loads it.**

The two share one C runtime, which is why both premake scripts set `staticruntime "off"`. Mix a
Debug module with a Release runtime and the result is not a clean error: it is heap corruption
that surfaces somewhere unrelated, long after the cause. The editor's dialog defaults to its own
configuration for exactly this reason and warns when the two disagree, and `Packager.cpp`
refuses a module whose configuration does not match the runtime's.

The same rule is what gets you symbols. A module built in the same configuration as the runtime
is a module you can attach a debugger to.

## Where the library lands

| Built by | Output |
|---|---|
| **Build → Game Module**, or `Build-GameModule.ps1` / `.sh` | `Source/Bin/<Module>.dll` (or `.so`), inside the project |
| The engine tree's own build | `bin/<Config>-windows-x86_64/LABRuntime/` beside the runtime, and a copy in `Dev/Out/modules/` |

The engine's own fixture, `TestGameModule`, takes the second route, because a scene test runs
the real runtime and the runtime resolves a module beside its own executable. Its build script
copies the library to both places on purpose: a test run (`--test`) looks in `Dev/Out/modules/`
as well as beside the executable, which is how an editor test that asks for the registry reaches
the module without it being copied beside the editor, an executable that gets shipped.

## Telling the project to load it

One line in the `.lab` file, written for you by **Use as this project's Native Module**:

```yaml
NativeModule: Source/Bin/MyModule.dll
```

`Project::GetNativeModulePath` resolves it **relative to the open project's own directory
first**, then falls back to **beside the executable**, exactly as a shader binary is found. The
second case is what a packaged build relies on, since its module is staged next to the runtime
rather than inside the archive.

A `NativeModule` that resolves to nothing fails a standalone build rather than warning. That is
deliberate: without the module there is no gameplay at all, so a build that quietly omitted it
would run and do nothing.

## When the module is loaded

`NativeModuleHost` mirrors the Lua script engine's lifecycle.

| Moment | What happens |
|---|---|
| The editor opens a scene | The library is loaded and its registry read, so the inspector can offer class names. No instance is created |
| Play starts (`Scene::OnRuntimeStart`) | The project's `NativeModule` is loaded for real, and `OnCreate` runs on every instance |
| Every frame (`OnUpdateRuntime`) | `OnFixedUpdate` per physics step, `on_update` after the step, contacts dispatched after that |
| An entity is destroyed | `OnDestroy` for whatever is being destroyed |
| Play stops (`OnRuntimeStop`) | `OnDestroy` on everything still alive, then the instances are destroyed and the library unloaded |

Only play mode runs behaviours. Opening a scene in the editor cannot move an entity under the
cursor, which is the point of splitting those two.

### A test can load its own module

`load_native_module(path)` is a test-only hook that resolves the same way and re-runs every
`on_create`. It is registered only in the test API table, reached through `LoadTest`, never for
an ordinary `ScriptComponent`, so gameplay Lua in a shipped game cannot load an arbitrary
shared library.

The runtime otherwise loads whatever the project's `NativeModule` names. A scene test whose
scene carries a `NativeScriptComponent` does nothing at all until it calls this, and nothing
reports a failure, because no entity was ever asked to start anything. Every native scene test
calls it, and a new one that forgets looks exactly like a feature that does not work.

The editor's route is separate: `editor.load_native_module`, `editor.native_classes`,
`editor.is_native_module_loaded` and `editor.native_behaviour_count`, because the play-time hooks
go through the bound scene and that binding only exists once play has started.

## Hot reload

`NativeModuleHost::PollHotReload` runs once a frame and stats the module path. When its write
time changes, the module is reloaded under a scene that keeps running, with no stop and start.

**It is off by default.** Re-running `OnCreate` is a real side effect, so it is opt-in per run:

```
set_perf_toggle("editor.native_hot_reload", true)
LAB_PERF_TOGGLES=editor.native_hot_reload=1
```

What a reload does, in order:

1. Read every instance's declared fields out.
2. Call `OnDestroy`, then `Destroy`, on every instance.
3. Close the old image.
4. Load the freshly built one.
5. Recreate each instance, writing the preserved field values back **after** the scene's own
   authored values and **before** `OnCreate`.
6. Call `OnCreate` again.

Only declared fields survive. A behaviour's own members, timers and handles are its business,
and a declared field is the one thing both compilers already agree on. That is the same
trade-off a Lua script's reload makes with its top-level locals.

### Why a module is loaded from a copy

Two facts force it:

- Windows' `LoadLibrary` keeps a loaded DLL, and a debugger its PDB, locked. Without a copy the
  build could not overwrite its own output while the game was running.
- `dlopen` caches by path, so reopening the same path after a rebuild hands back the **old**
  image.

So the host copies the library before opening it. The copy keeps the module's own directory, so
sibling dependencies still resolve beside it, and Windows gets its `.pdb` copied alongside.
Copies are named `<module>.hotreload<generation>.<ext>`, and the ones an earlier run left behind
are swept once on the first load. A copy that cannot be made falls back to loading in place
rather than refusing to load.

### A refused reload leaves the running module alone

The candidate image is loaded and interrogated **first**. Only a module whose major matches and
whose minor is not newer than the engine's gets to replace a running one.

This ordering is the whole point. Checking after the swap would mean a mismatched build leaves
the game with no behaviour running and the previous module already gone, which is worse than
refusing to swap at all. A refusal is logged, reported into the script failure list the way a
native failure already is, and visible through `GetReloadCount()` and `GetLastReloadResult()`.
So "why did my rebuild not take?" is answerable without a debugger.

## Packaging

**Build Standalone** stages the module beside the runtime executable and keeps it **out of
`Game.Lpak`**. That is the same treatment the Steam library gets, and for the same reason:
`LoadLibrary` and `dlopen` want a path, not an entry in an archive.

A module packed into the archive is not a build that fails. It is a build that looks complete,
lists the module in `BuildManifest.yaml`, and then cannot load it. A shared library staged into
a packaged build has to be excluded from the archive by name.

The dependency walker cannot see a behaviour loading an asset by a string. Name those assets in
`Build.AdditionalIncludes`, or arrange for a declared asset-path field once that type exists.

A packaged build resolves the module beside the **target runtime**, not beside the process that
packaged it. Packaging a Linux build from a Windows editor otherwise picks up the Windows module
sitting next to the editor, or finds nothing at all.
