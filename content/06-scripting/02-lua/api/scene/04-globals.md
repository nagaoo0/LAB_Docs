---
title: "globals"
---

A plain table shared by every running script in the scene, for values one script wants another to
see. See [Lua Scripting](../../index.md).

## `globals`

It is a table like any other: read and write any key, and declare nothing.

```lua
function on_create()
    globals.count = (globals.count or 0) + 1
end
```

Two entities running that same script see `1` and `2`, not `1` and `1` each, because there is one
table for the whole scene rather than one per entity.

### Shared, not per entity and not per script file

A script's own `local` variables are private to that one running instance, which is why two
entities running the same file do not share them. Anything set on `globals` is visible to all of
them, which is what makes it the place for a value one script publishes and another reads: a turn
counter, a game phase, a flag a trigger sets and a door reads. A script written as a node graph
shares it too, since a graph is compiled to Lua and loaded the same way.

**Write the prefix.** `globals.count` is a field of that table, not a bare variable called
`count`, so every script reaches it as `globals.count`. Dropping the prefix in one place and
keeping it in another is how a shared value mysteriously reads as nil.

### When it is reset

**It starts empty every time play starts**, because starting play builds the scene's Lua state
afresh. That makes it a session-scoped scratch space: a value is either something a script set
this session, or it is nil. Nothing clears it for you while play runs, and it is not written into
a save file.

**It survives a level switch.** `scene.open_scene` and `scene.append_scene` destroy and create
entities but do not rebuild the Lua state, so the table and everything on it are still there
afterwards. The scripts that are gone are the ones that used to read it.

### Where else state can live

| | Scope | Survives |
|---|---|---|
| A `local` in a script | one running instance | nothing beyond that instance |
| `globals` | every script in the scene | a level switch within one play session |
| `game.set_state` / `game.get_state` | the whole process | a level switch and a save/load |

When a value has to outlive play or be carried by a save file, it does not belong here. The
blackboard behind the [`game`](../game/) table is one per process rather than one per play
session, it reads the same across a level switch and a load, and a save slot round-trips it; the
C++ side of that store is [the blackboard](../../../03-cpp/api/savegame/01-blackboard.md).

### Not the test-harness globals

The test harness registers global functions of its own alongside the ordinary API (`expect`,
`read_asset_text`, `entity_count` and the rest). Those are plain functions a test script is given,
and they have nothing to do with this table.
