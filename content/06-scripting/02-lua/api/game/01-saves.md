---
title: "Save slots"
---

Checkpoints and profile saves, from Lua. See [Lua Scripting](../../index.md).

The `game` table's save calls snapshot the running game into a named slot and put it back. Two
different things share that table, and the difference matters more than the calls do:

| | A **save** | A **state** |
|---|---|---|
| Calls | `game.save`, `game.load`, `game.save_exists`, `game.delete_save` | `game.save_state`, `game.load_state`, `game.state_exists` |
| Holds | The running scene **and** the blackboard | The blackboard alone |
| File | `<Saves>/<slot>.Lsave` | `<Saves>/<slot>.Lstate` |
| Is | A checkpoint: where everything was | A progression profile: what the player has done |
| Loading it | Reopens the scene it saved | Reloads no scene at all |

`<Saves>` is `<project>/Saves` unless the project asks for a per-user folder, see [Where the files go](#where-the-files-go).

A game usually wants both. The profile is what a menu's "Continue" reads to know which level to
open, and the checkpoint is the position within that level. The state calls live with the
blackboard itself: see [The blackboard](02-blackboard.md).

[C++ equivalent](../../../03-cpp/api/savegame/02-save-slots.md)

## `save(slot)`

Writes the running scene and the blackboard into `<project>/Saves/<slot>.Lsave`, and answers
`true` when it did.

Three things land in the file: the scene path a load will reopen, every blackboard key, and a
snapshot of every entity carrying a `PersistentComponent`. Everything else in the scene comes
back from the scene file itself, which is why the scene needs one.

**It answers `false` when the running scene has no scene file behind it**, because a load would
then have nothing to reopen. A scene opened from a file and never saved is fine. A scene that
only ever existed in memory is not. It also answers `false` with no project open, and logs the
reason in both cases.

The call runs immediately: it only reads the registry and writes the filesystem. The slot is a
bare name, with no path separators and no extension.

```lua
if not game.save("checkpoint") then
    log.warn("this scene has no file to reopen")
end
```

A persistent entity whose parent is not itself being saved is detached to the root first,
keeping its world placement, so the file never records a parent that is not in it. That is the
same rule a scene switch uses.

## `load(slot)`

Puts a save back: reopens the scene the file names, replaces the blackboard with the file's,
and replays the saved entity snapshots on top.

**The load is queued to the end of the frame**, exactly like `scene.open_scene`, and for the
same reason: it destroys and creates entities, which cannot happen while the engine is still
walking the scripts that called it. The call itself answers immediately, so there is no return
value to read; what to read is the scene it leaves behind, a frame later:

```lua
game.load("checkpoint")
-- Still the old scene until the end of this frame.
```

Only the most recent `load` asked for during a frame is applied. A slot with no file behind it
changes nothing and reports the failure to the log; the script that asked sees no error.

## `save_exists(slot)`

`true` when `<project>/Saves/<slot>.Lsave` is there, and `false` otherwise: a slot that was
never saved, or a project that has never saved at all, since the `Saves` directory itself only
appears once a save creates it. Answers `false` with no project open.

```lua
if game.save_exists("checkpoint") then
    ShowContinueButton(true)
end
```

## `delete_save(slot)`

Removes `<slot>.Lsave` and answers whether it was there. Answers `false` for a slot that does
not exist, and with no project open. It touches neither the blackboard nor the scene, so
deleting the slot a game is currently running from is harmless until something loads it.

## `save_data(slot, table)` / `load_data(slot)` / `has_data(slot)` / `delete_data(slot)`

```lua
game.save_data("inventory", {
    gold = 120,
    items = { "key", "map", { name = "potion", count = 3 } },
    settings = { music = 0.8, subtitles = true },
})

local inv = game.load_data("inventory")   -- nil when there is no such slot
if inv then log.info(inv.items[3].name) end

if game.has_data("inventory") then game.delete_data("inventory") end
```

For data the blackboard can't hold, which only keeps bools, numbers, strings and vec3. A data
slot holds any nested table and is not tied to a scene: it is just JSON in
`<Saves>/<slot>.Ldata`, written to a temp file and renamed over the old one.

`save_data` and `delete_data` answer true or false. `load_data` answers the table, or nil when
the slot is missing or the file does not parse.

What goes in and what comes back:

- A table keyed 1..n is an array, any other table is an object. An empty table comes back as an
  empty table.
- Keys of an object are strings. An integer key of a table that is not an array comes back as a
  string (`[10] = "x"` reads back as `["10"]`).
- Integers stay integers and floats stay floats, a whole float like `3.0` included. Strings that
  look like numbers (`"007"`) stay strings.
- Booleans, strings (any UTF-8) and nested tables are kept. Functions, entities and other
  userdata are written as null, so they are gone on the way back.
- NaN, infinity, a table that contains itself and nesting deeper than 64 levels are refused:
  `save_data` logs the reason and answers false, and the old file is left alone.

A slot is a bare name. An empty one, one with a path separator, `..`, a colon or other characters
a file name can't have, or one that starts or ends with a dot, is refused by all four calls.
Nothing is written outside the save folder.

## `save_directory()`

The absolute path of the folder the save, state and data slots live in, as a string, or an
empty string with no project open. Useful for a "show save folder" button or a log line.

```lua
log.info("saves go to " .. game.save_directory())
```

## Where the files go

By default all three kinds live in the open project's `Saves/` directory, which is created
lazily by a save. `load`, `save_exists` and `delete_save` never create it, so a project that has
never been saved simply has no directory and every slot reads as absent.

A project installed where its folder is read-only (Program Files) can set `SaveLocation: user`
in its `.lab` file, with an optional `SaveFolder: MyGame` (the project name when empty). A
packaged game then writes to `<user data dir>/<SaveFolder>/Saves`, where the user data dir is
`%APPDATA%` on Windows and `$XDG_DATA_HOME` (else `~/.local/share`) on Linux. The environment
variable `LAB_USER_DATA_DIR` overrides it, which lets a test redirect the saves. If no user data
dir can be found the project folder is used.

**The editor ignores this**, Play mode included: it always uses `<project>/Saves`, so editing
stays reproducible and a save that shows a bug travels with the project. Only a standalone
runtime uses the user location. `game.save_directory()` says which one is in effect.

## A load is a fixed point in time

Worth knowing when persistent entities are involved, because the two features interact.

A `PersistentComponent` normally means "carry this entity's live state across a level change",
and the engine does that on a scene open. A save is a snapshot, so loading one has to mean the
file wins rather than the session.

So a load starts by clearing the way: it strips the persistent flag from every entity in the
running scene **before** reopening the saved scene, so the open genuinely rebuilds everything
from the file rather than carrying the current session's version of anything over it. Only then
are the save's own entity snapshots replayed on top.

The visible consequence: an enemy that moved, a door the player opened, or a persistent flag
granted at runtime after the save all do **not** survive a load. Reloading an old save puts back
exactly what was there, which is what a player expects a checkpoint to do.

For a script, that means a `local` variable is not part of the file at all (the serializer
records a script component's path, not its running state), and a value that should come back has
to be on a persistent entity or in the blackboard.

## A complete example

Two halves of a level end: the checkpoint trigger, and the menu that reads the profile.

```lua
-- The level-end trigger, on a script that watches the goal.
function on_overlap_begin(other)
    game.save("checkpoint")                 -- where everything is

    game.set_state("Progress", 4)           -- what the game as a whole remembers
    game.set_state("LevelFourUnlocked", true)
    game.save_state("profile")

    log.info("checkpoint saved")
end
```

```lua
-- The main menu, choosing whether Continue is offered.
function on_create()
    if game.state_exists("profile") then
        game.load_state("profile")
        ShowContinueButton(true)
    else
        ShowContinueButton(false)
    end
end
```

Reloading that checkpoint goes through `game.load("checkpoint")`, and the scene it puts back is
the one the save was taken in. What a load cannot do is pick a scene: for a level switch that is
not a checkpoint, use `scene.open_scene` (see [`scene/`](../scene/)).
