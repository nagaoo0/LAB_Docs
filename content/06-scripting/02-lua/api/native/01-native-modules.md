---
title: "Native modules (native)"
---

Calling into a loaded C++ gameplay module, and the globals that answer what it registered. See
[Lua Scripting](../../index.md).

A project can ship a C++ gameplay module: real compiled code, loaded beside the executable or out
of the project, which registers behaviour classes an entity references and, optionally, plain
functions for Lua to call. Those functions are this page. The module side of the boundary is in
[Lua interop](../../../03-cpp/api/lua/01-lua-interop.md), and the module's own lifecycle in
[Module lifecycle](../../../03-cpp/api/module/01-module-lifecycle.md). Writing one is
[Scripting in C++](../../../03-cpp/index.md).

**One name is on the table, the rest are globals.** `native.call` is the only function under
`native`. The reflection readers are plain globals, so they are called bare
(`native_module_loaded()`, not `native.module_loaded()`).

## `native.call(name, ...)`

```lua
local ok = native.call("set_difficulty", 3)
```

Calls the function the loaded module registered as `name` and returns what it returned, converted
to the matching Lua type. `name` is the name the module passed to `RegisterFunction`, not a path
and not an entity.

**Arguments cross as one of five shapes**: a number, a boolean, a string, a `vec3` or an entity.
Anything else a script hands over (a table, a function, `nil`, a quat) is not coerced into
something plausible: it arrives on the module side as "an argument that is not one of the five
shapes". So pass numbers, strings, booleans, vectors and entity handles, and put anything richer
into several calls.

**The return is the module's own value**, converted the same way: a number, a boolean, a string, a
`vec3`, or an entity (which arrives as a normal entity reference with the same methods as
`entity`). A returned string is copied into a fixed buffer before Lua sees it, so a very long one
is truncated rather than refused.

**A call that cannot be answered is `nil`.** There is no error and nothing to catch:

| Situation | Answer |
|---|---|
| The name is not one the module registered | `nil` |
| No module is loaded, or no scene is bound | `nil` |
| The module function threw | `nil`, with the throw written to the log as `NativeModuleHost: behaviour '<name>' threw from native.call: <what>` |

Nil is also what a module function that returns nothing gives, and that is honest rather than
confusing: a function that returns nothing and a function that does not exist are both "no value"
to a caller that only looks at the result. A caller that has to tell them apart asks
`native_function_names()` (below) before the call, or, from the module side,
`api->Lua->HasFunction`.

```lua
-- Call it, and use the answer as a flag the module decided.
if native.call("is_level_unlocked", "level3") then
    scene.preload_scene("scenes/level3.Lscene")
end
```

```lua
-- A name nothing registered is a nil, not an error: check when it matters.
if native.call("does_not_exist") == nil then
    log.warn("that function is not in the module")
end
```

## The reflection globals

These report what is loaded and what it registered. All of them answer a neutral value rather than
faulting when there is no scene bound, which is the state before play starts: a test asking about
native behaviours before anything is running gets `false`, `0` or an empty table rather than an
error.

### `native_module_loaded()`

True when a module is loaded into the current scene, false when none is (including when nothing
has bound a scene yet). It is the answer to "did my module load at all", which is otherwise only a
line in the log.

A module is normally loaded from the project's own setting; `load_native_module` below loads one by
path instead.

### `native_behaviour_count()`

How many native behaviours are **running**: instances, not classes and not entities. One entity
running three classes counts as three, and it answers 0 when no module is loaded.

### `native_disabled_count()`

How many behaviours have been switched off after throwing. A behaviour that throws is disabled
rather than called again next frame, because the exception says its own state is suspect and
calling it sixty more times a second turns one bug into a log flood and a scene nobody can use.
The counter climbing while the frame keeps running is the signal that a module is misbehaving.

### `native_classes()`

The class names the loaded module registered, as a one-indexed table (`#native_classes()` is how
many), **sorted alphabetically** so a tool and a test see the same order twice running.

This is the answer to "why is my class unknown": it is the list of names that would have worked,
and a `NativeScriptComponent` naming anything else instantiates nothing.

```lua
for i = 1, #native_classes() do
    log.info("behaviour class: " .. native_classes()[i])
end
```

### `native_function_names()`

The names the module exported for Lua to call, as a one-indexed table. It is the same registry
`native.call` resolves against, which is what makes an unknown name answerable rather than just
`nil`.

```lua
-- Before calling into a module that may not be there.
local names = native_function_names()
local known = false
for i = 1, #names do
    if names[i] == "spawn_wave" then known = true end
end
if known then native.call("spawn_wave", 2) end
```

### `native_last_behaviour()`

The last native behaviour callback to start, as `"<class> <phase>"` (for example
`"Mover OnUpdate"`), or an empty string when no module code has run. The phases are the callback
names (`"Create"`, `"OnCreate"`, `"OnUpdate"`, `"OnFixedUpdate"`, `"OnDestroy"`, `"Destroy"`,
`"contact callback"`) plus `"native.call"` for a call made through this table.

It reads a process-global rather than the scene, so it needs no bound scene and answers in the
editor too. That makes it the one reader that can say what a module was doing at the moment
something went wrong, which is exactly what the crash report uses it for and what a test asserts
against.

### `load_native_module(path)`

```lua
load_native_module("gameplay") -- no extension: this platform's .dll or .so
```

Loads a module by path and answers **true** when it loaded, **false** otherwise (including when no
scene is bound at all).

The path resolves the way the project's own `NativeModule` setting does: the open project's
directory first, then beside the executable. So a test can point at a module built next to the
runtime rather than authoring a whole second fixture project only to name it, which is what this
call exists for: it is the test and editor override for the project setting, and the project
setting is the normal way a game ships a module.

Loading is not starting:

- **In the editor**, nothing is instantiated. The registry becomes readable, so the readers above
  answer for the module, and no behaviour runs.
- **While play is running**, every `NativeScriptComponent`'s `on_create` is run again against the
  freshly loaded module, because play's own start already ran against whatever the project setting
  pointed at, usually nothing.

Failure answers `false`. Anything already loaded is unloaded first, so a refused load leaves no
module loaded rather than the previous one; the reason (a path that does not resolve, a module that
will not open, an ABI major that does not match) is written to the log and reported as a failure,
while a refused hot reload of a running module leaves that module exactly where it was. The ABI
rules are in [ABI and versioning](../../../03-cpp/api/module/02-abi-and-versioning.md).

## What is not here

There is no way for Lua to register a behaviour class, to read a behaviour's declared fields
(that is the editor's inspector, see
[Declared fields](../../../03-cpp/api/module/04-declared-fields.md)), or to unload a module by hand. A
script can call what a module exported and read what it registered; the module's own instances
belong to the `NativeScriptComponent`s in the scene.
