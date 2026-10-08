---
title: "The blackboard"
---

A key/value store that outlives the scene, reached from Lua through the `game` table's
`*_state` calls, plus the profile files that store it alone. See
[Lua Scripting](../../index.md).

`GameState` is **one instance per process**, not per scene. A value written here survives a
scene switch, a level change and a game load. That is what makes it the place for anything a
game carries: a score, an unlocked door, a quest stage, the fact that a character has already
been met.

It is also exactly what a save file's own blackboard block round-trips, so nothing here needs
separate saving logic. A checkpoint (`game.save`) carries it along with the scene; the three
profile calls below store it on its own.

The C++ module side writes the same blackboard, so a module and a script share one set of keys.

[C++ equivalent](../../../03-cpp/api/savegame/01-blackboard.md)

## The four shapes

A value is a bool, a number, a string or a `vec3`, and **the type is part of the key**.

```lua
game.set_state("Score", 1240)
game.set_state("DoorOpen", true)
game.set_state("LastCheckpoint", "crypt")
game.set_state("SpawnPoint", vec3.new(3, 4, 0))
```

Writing a key that already holds a value of another shape replaces it, type included. There is
no coercion anywhere: a key holds what it was last written as.

## `set_state(key, value)`

Stores a value under a key. The value may be a bool, a number, a string or a `vec3`.

**Anything else is refused.** A table, a function or a nil logs a warning naming the key and the
wanted shapes, and writes nothing. The call answers nothing.

```lua
game.set_state("Gems", 0)
```

## `get_state(key [, default])`

Answers the value stored under the key, in whatever shape it was stored with.

The type never converts, and that is the point. Lua has one untyped getter here, so "wrong type"
is not something the call can report: a key written as text reads back as text, and a script
that then treats it as a number gets no number out of it. The typed C++ getters put the same
rule the other way round, answering 0 for a key of another shape rather than converting it.

A key that is not there answers the second argument, and `nil` when none was given:

```lua
local score = game.get_state("Score", 0)
local name = game.get_state("LastCheckpoint", "start")
```

Since nil is never stored, a `nil` answer always means "not there", never "stored as nil".

## `has_state(key)`

The type-blind question: `true` for a key that is present whatever it holds, `false` for one
that is not. It is the call for "has this game ever been played through here", where the value's
shape does not matter.

```lua
if game.has_state("LastCheckpoint") then
    ShowContinueButton(true)
end
```

## `remove_state(key)`

Forgets one key. A key that is not there is a no-op rather than an error, and the call answers
nothing.

## `clear_state()`

Empties the blackboard and nothing else. **It touches no entity**, so a New Game that wants a
genuinely fresh start wants a scene reload too: clearing without reloading leaves the world
exactly as it was and only forgets what the game knew about it.

```lua
game.clear_state()
scene.open_scene("scenes/level_01.Lscene")
```

## `save_state(slot)`

Writes the blackboard alone, with no scene and no entities, to
`<project>/Saves/<slot>.Lstate`. This is a progression profile: what the player has unlocked,
how many stars, what the options are. It is the file a menu reads at startup without reloading
anything.

Answers `true` when it wrote. It answers `false` with no project open, for an empty slot name,
and when the file cannot be written. The file is written beside its own name and renamed over,
so a crash mid-write never loses the profile that was already there.

## `load_state(slot)`

Replaces **every** key this process holds with what the file holds, immediately, and reloads no
scene. It is a complete replacement rather than a merge, so a load leaves the blackboard exactly
as it was saved with, and nothing of the current session behind.

Answers `true` when it read the file, and `false` with no project open, for an empty slot name,
for a slot with no file behind it, or for a file that will not parse. Nothing changes when it
answers `false`. With no entities to create or destroy there is
nothing to defer, which is why this one runs on the spot where `game.load` has to wait for the
frame to end.

```lua
-- Read the profile to decide which level to open.
game.load_state("profile")

local level = game.get_state("Progress", 1)
scene.open_scene("scenes/level_0" .. level .. ".Lscene")
```

[C++ equivalent](../../../03-cpp/api/savegame/02-save-slots.md)

## `state_exists(slot)`

`true` when `<project>/Saves/<slot>.Lstate` is there, `false` otherwise. Answers `false` with no
project open, and for a project that has never saved a profile: as with saves, the `Saves`
directory is only created once a save writes into it.

## When to use it

The blackboard earns its place the moment state has to outlive a scene. Anything that is
per-entity belongs on the entity, as a component or a behaviour's own variable; anything that is
per-run belongs here.

| State | Where |
|---|---|
| This enemy's health | On the entity |
| This level's score | Blackboard |
| Whether a secret door has been opened | Blackboard |
| Player position within a level | The scene, and the save slot |
| Player position across a level change | A `PersistentComponent` |

The blackboard is one flat namespace shared by every system in the game, so a prefix per system
is worth copying from the sample scripts: a collectable keys itself `"taken:" .. name`, and a
quest system would key itself `"quest:" .. id`, which is what keeps two of them from writing the
same key.

## A complete example

A collectable that remembers what has already been taken, so a re-entered scene does not hand
the same thing out twice.

```lua
local key = "taken:" .. entity:get_name()

function on_create()
    if game.get_state(key, false) then
        scene.destroy(entity)       -- already collected on an earlier visit
    end
end

function on_overlap_begin(other)
    game.set_state(key, true)
    game.set_state("Gems", game.get_state("Gems", 0) + 1)

    entity:emit_particles(20)
    entity:play_sound()
    scene.destroy(entity)
end
```

Keyed by this entity's own name, which is stable in the scene file.
