---
title: "ABI and versioning"
---

The module and the engine are built independently, possibly by different compiler versions.
One header is the contract between them, and its version number is how the two agree about
what it says.

## The version

```c
#define LAB_NATIVE_API_MAJOR 2
#define LAB_NATIVE_API_MINOR 13
#define LAB_NATIVE_API_VERSION ((LAB_NATIVE_API_MAJOR * 1000) + LAB_NATIVE_API_MINOR)
```

A module reports the version it was built against by returning `LAB_NATIVE_API_VERSION` from
`LAB_ModuleInit`. Both numbers are in the macros, so a module that returns the macro is always
telling the truth about which header it compiled against.

## Major and minor answer different questions

**Major is about layout.** A different major is refused outright, because a struct one side has
and the other does not means reading past the end of the memory the other side filled. There is
no compatibility shim and there is not meant to be.

**Minor is about capability.** It says whether the engine is new enough for what the module
asks of it:

| The module was built against | The engine | Result |
|---|---|---|
| An older minor | A newer minor | Loads. The engine still has every entry that module knows about |
| The same minor | The same minor | Loads |
| A newer minor | An older minor | Refused. The entry points it wants were not there when this engine was built |

That asymmetry is deliberate: a module built against an older header keeps working forever,
because everything it can name still exists. A module built against a newer one is refused
rather than allowed to read a field the engine never filled, which would be a call through
whatever happened to be in that memory.

## `LAB_EngineAPI`

The table the engine fills before calling `LAB_ModuleInit`.

| Field | |
|---|---|
| `uint32_t StructSize` | `sizeof` of the table the engine actually built |
| `uint32_t Major` | The major the engine implements |
| `uint32_t Minor` | The minor the engine implements |
| then every function pointer, and the sub-table pointers (`AI` to `UI`, `Lua`, `Net`, `SteamLobby`, `Physics`, `Audio`) | |

`StructSize` is what makes appending an entry point a minor change rather than a major one.
The engine reports how much of the table it filled, and a module asks whether the field it
wants lies inside that, rather than assuming a layout.

```c
if (LAB_ENGINE_API_HAS_ABI(api, SetPosition))
	api->SetPosition(id, LAB_Vec3{ 1.0f, 2.0f, 3.0f });
```

## The `HAS_ABI` macros

```c
#define LAB_ENGINE_API_HAS_ABI(api, field) \
	((api) != NULL && (api)->StructSize >= (uint32_t)(offsetof(LAB_EngineAPI, field) + sizeof((api)->field)) \
		&& (api)->field != NULL)
```

It asks **two** questions, and both are needed:

- The size says the field existed in the header this code was compiled against. That is the
  version gate.
- The pointer test catches a cell the engine never filled. An entry a build forgot to fill, or
  one a newer engine left at zero, is a null pointer that passes the size test. Without the
  pointer test the call would go through null instead of reading as absent.

A grouped sub-table has its own form, because plain C cannot spell "the type of whatever this
pointer is":

```c
if (api && LAB_ENGINE_API_HAS_ABI(api, UI) && LAB_UI_API_HAS_ABI(api->UI, SetText))
	api->UI->SetText(entity, "hello");
```

| Macro | For |
|---|---|
| `LAB_ENGINE_API_HAS_ABI(api, field)` | An entry on the root table, or a sub-table pointer |
| `LAB_AI_API_HAS_ABI(table, field)` | An entry in `api->AI` |
| `LAB_SPLINE_API_HAS_ABI(table, field)` | An entry in `api->Spline` |
| `LAB_STREAMING_API_HAS_ABI(table, field)` | An entry in `api->Streaming` |
| `LAB_SAVEGAME_API_HAS_ABI(table, field)` | An entry in `api->SaveGame` |
| `LAB_UI_API_HAS_ABI(table, field)` | An entry in `api->UI` |
| `LAB_LUA_API_HAS_ABI(table, field)` | An entry in `api->Lua` |
| `LAB_NET_API_HAS_ABI(table, field)` | An entry in `api->Net` |
| `LAB_STEAMLOBBY_API_HAS_ABI(table, field)` | An entry in `api->SteamLobby` |
| `LAB_PHYSICS_API_HAS_ABI(table, field)` | An entry in `api->Physics` |
| `LAB_AUDIO_API_HAS_ABI(table, field)` | An entry in `api->Audio` |

`LABNative.hpp`'s wrappers already make both checks, so a module using the C++ layer never
writes one of these by hand. They are needed only when calling the ABI directly.

## The shape of the table

The root table carries the calls every behaviour might need. Since minor 8, the areas that do
not fit that description live in one sub-table each, pointed at from the root:

| Sub-table | Entries | Area |
|---|---|---|
| `api->AI` | 7 | Navigation and behaviour trees |
| `api->Spline` | 6 | Spline queries |
| `api->Streaming` | 3 | Scene preload, append and open |
| `api->SaveGame` | 18 | The save game and its blackboard |
| `api->UI` | 50 | The HUD and its widgets (widgets added at minor 12) |
| `api->Lua` | 2 | Calling Lua |
| `api->Net` | 31 | Hosting, joining, replication, RPC (minor 11); tick rate, interpolation delay, client address and dropping a client (minor 13) |
| `api->SteamLobby` | 24 | Steam lobbies (minor 11) |
| `api->Physics` | 5 | The solver's own body position, angular velocity, wake-up, `RaycastEx` (minor 13) |
| `api->Audio` | 2 | Per-voice volume and pitch, and whether a voice is playing (minor 13) |

Each leads with its own `StructSize`, for the same reason the root table does.

### Minor 13

Minor 13 added `api->Physics` and `api->Audio`, six entries at the end of `api->Net`, and changed
**when** `OnFixedUpdate` runs without changing its signature: it is now called once before each
physics step instead of once per step after all of a frame's steps. A module built against an
older minor keeps loading and now sees its hook run ahead of the step rather than behind it, which
is the order a force-applying hook always wanted. See [the lifecycle page](01-module-lifecycle.md).

Every entry point added from here on goes **at the end of the sub-table for its area**, never
into the middle and never onto the root table. Inserting one in the middle would move every
entry after it, and every module already built would call the wrong function at that offset.
The root table does not grow any more; an area that needs more room grows its own sub-table.

There is one apparent exception worth knowing about, because it is not one:
`LAB_ModuleRegistry::RegisterFunction` is a new *field on an existing struct* rather than a
sub-table entry. A registration call needs the `self` the registry already carries, and a plain
function pointer on a sub-table has nowhere to put one.

## What a refusal looks like

A refused module is logged and skipped, never crashed into. The log line names which of the
three refusals it was and gives both versions, so "my module does not load" is answerable from
the log alone.

| Message | What to do |
|---|---|
| `LAB_ModuleInit` missing | Check the export is `extern "C"` and marked `LAB_NATIVE_EXPORT` |
| Major differs | The header in the engine tree is a different generation from the one the module compiled against. Rebuild the module |
| Module's minor is newer | The module was built against a newer engine header than this engine implements. Rebuild the engine, or build the module against this tree's header |

The third one is the trap worth naming: it is most often seen right after the header gained an
entry point, by someone who rebuilt the module and not the engine. The module is correct and
the engine is the stale half.

## The changelog

What each minor added, so a module's required engine version is readable.

| Minor | Added |
|---|---|
| 1 | The engine's clock: `ElapsedTime`, `FrameIndex`, `FixedDeltaTime` and the fixed-step callback |
| 2 | Generic component access: `GetComponent`, `SetComponent`, `ComponentSize` and the mirror structs |
| 3 | Mirror structs for the three light components. No new entry point |
| 4 | `ZoneAdd` |
| 5 | Declared fields: `LAB_FieldType`, `LAB_FieldDesc`, and `Fields`/`FieldCount` on the behaviour descriptor. No new entry point |
| 6 | `SpawnEntity`, `SpawnEntityAt`, `SetParent`, `GetChildren` |
| 7 | Audio, the animator, particles, and `GetParent` |
| 8 | The five grouped sub-tables: AI, Spline, Streaming, SaveGame, UI |
| 9 | Lua interop in both directions: `LAB_Variant`, `api->Lua`, and `RegisterFunction` |

**Major 2** is where `StructSize`, `Major` and `Minor` themselves arrived. Nothing external had
been built against version 1, so nothing needed a shim.

A minor is bumped for a change a module can *notice*: a new entry point, a new component type,
a field added to a mirror struct. A comment, a rename inside the engine, or anything a module
cannot observe is not an ABI change and bumps nothing, because bumping it reports every module
as "built against a newer engine" until each one is rebuilt. That is correct for a real
addition and noise for anything else.

## Changing the ABI

If you are working on the engine rather than a module, three rules cover it:

- **Append, never insert.** An entry goes at the end of its area's table.
- **A new area gets its own sub-table**, with a `StructSize` field first, a pointer to it on
  the root table appended at the end, and a `LAB_<AREA>_API_HAS_ABI` macro alongside the
  others.
- **Bump the minor** for anything a module can observe, and bump the major only for a layout
  change.

A sub-table entry that declares a struct member and forgets to fill it is a null pointer, and
the `HAS_ABI` pointer test is what turns that into "absent" rather than a call through null. So
a new sub-table wants a test that counts its non-null entries, which is how the engine's own
fixture holds itself to the numbers the header declares.
