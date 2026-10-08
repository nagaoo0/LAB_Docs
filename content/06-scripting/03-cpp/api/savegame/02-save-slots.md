---
title: "Save slots"
---

Checkpoints and profile saves.

Two different things share this table, and the difference matters more than the calls do:

| | A **save** | A **state** |
|---|---|---|
| Calls | `Save`, `Load`, `SaveExists`, `DeleteSave` | `SaveState`, `LoadState`, `StateExists` |
| Holds | The running scene **and** the blackboard | The blackboard alone |
| File | `<Saves>/<slot>.Lsave` | `<Saves>/<slot>.Lstate` |
| Is | A checkpoint: where everything was | A progression profile: what the player has done |
| Loading it | Reopens the scene it saved | Reloads no scene at all |

A game usually wants both. The profile is what the options screen's "continue" reads to know which
level to load; the checkpoint is the position within that level.

## Save and load

```c
int  (*Save)(const char* slot);
void (*Load)(const char* slot);
int  (*SaveExists)(const char* slot);
int  (*DeleteSave)(const char* slot);
```

```cpp
bool LABNative::SaveGame::Save(const char* slot);
void LABNative::SaveGame::Load(const char* slot);
bool LABNative::SaveGame::SaveExists(const char* slot);
bool LABNative::SaveGame::DeleteSave(const char* slot);
```

`Save` writes the running scene and the blackboard into a slot. It answers **0 when the running
scene has no scene file behind it**, because a load would then have nothing to reopen. A scene
opened from a file and never saved is fine; a scene that only ever existed in memory is not.

`Save` **runs immediately**: it only reads the registry and writes the filesystem.

```cpp
if (!LABNative::SaveGame::Save("checkpoint"))
	LABNative::LogWarn("this scene has no file to reopen");
```

`Load` **is queued to the end of the frame**, exactly like `OpenScene`, and for the same reason:
loading destroys and creates entities. It answers immediately, so there is no return value to
read; what to read is the scene it leaves behind, next frame.

```cpp
LABNative::SaveGame::Load("checkpoint");
// Still the old scene until the end of this frame.
```

`SaveExists` and `DeleteSave` are both immediate and both answer whether the slot existed.

## Profile saves

```c
int (*SaveState)(const char* slot);
int (*LoadState)(const char* slot);
int (*StateExists)(const char* slot);
```

```cpp
bool LABNative::SaveGame::SaveState(const char* slot);
bool LABNative::SaveGame::LoadState(const char* slot);
bool LABNative::SaveGame::StateExists(const char* slot);
```

The blackboard alone: no scene, no entities. This is a progression profile, and it is what a game
that tracks "which levels are unlocked, how many stars, what the options are" keeps on disk.

**`LoadState` replaces every key this process holds**, and reloads no scene. It is a complete
replacement rather than a merge, so a load leaves the blackboard exactly as it was saved with
nothing of the current session left behind.

`LoadState` also **runs immediately**, unlike `Load`: with no entities to create or destroy there
is nothing to defer.

```cpp
// Read the profile to decide which level to open.
LABNative::SaveGame::LoadState("profile");

double level = 1.0;
LABNative::SaveGame::GetNumber("Progress", &level);
LABNative::Streaming::OpenScene(("scenes/level_0" + std::to_string(static_cast<int>(level)) + ".Lscene").c_str());
```

## Where the files go

Both live in the save directory, which is created **lazily by a save**. `Load`, `SaveExists` and
`DeleteSave` never create it, so a project that has never been saved simply has no directory and
every slot reads as absent.

That directory is `<project>/Saves` by default. A project with `SaveLocation: user` in its `.lab`
file saves to `<user data dir>/<SaveFolder>/Saves` when run as a packaged game (`%APPDATA%` on
Windows, `$XDG_DATA_HOME` or `~/.local/share` on Linux). The editor, Play mode included, always
uses the project folder. See [Where the files go](../../../02-lua/api/game/01-saves.md#where-the-files-go).

## A save is a fixed point in time

Worth knowing when persistent entities are involved, because the two features interact.

A `PersistentComponent` normally means "carry this entity's live state across a level change",
and the engine does that on a scene open. A **save** is a snapshot, so loading one has to mean the
file wins rather than the session.

So a load starts by clearing the way: it removes persistence from every entity in the running
scene *before* reopening the saved scene, so the open genuinely rebuilds everything from the file
rather than carrying the current session's version of anything over it. Only then are the save's
own entity snapshots replayed on top.

The visible consequence: an enemy that moved, a door the player opened, or a value a behaviour
computed since the save was taken do **not** survive a load. Reloading an old save puts back
exactly what was there, which is what a player expects a checkpoint to do.

Two things follow for a module author:

- **Anything that should be in the save must be on an entity or in the blackboard.** A behaviour's
  own member is its own business and is not saved; a declared field is part of the entity's
  snapshot and is.
- **A live persistent entity's state does not survive a load**, deliberately, even though it
  would survive a level change. The two are different operations and the file wins the second.

## A complete example

A checkpoint trigger and a profile write, which is the pair a level end actually needs.

```cpp
void OnOverlapBegin(LAB_EntityId other) override
{
	// What this session is currently doing.
	LABNative::SaveGame::Save("checkpoint");

	// What the game as a whole remembers.
	LABNative::SaveGame::SetNumber("Progress", 4.0);
	LABNative::SaveGame::SetBool("LevelFourUnlocked", 1);
	LABNative::SaveGame::SaveState("profile");

	LABNative::LogInfo("checkpoint saved");
}
```

And the main menu reading it back:

```cpp
void OnCreate() override
{
	if (LABNative::SaveGame::StateExists("profile"))
	{
		LABNative::SaveGame::LoadState("profile");
		ShowContinueButton(true);
	}
	else
	{
		ShowContinueButton(false);
	}
}
```
