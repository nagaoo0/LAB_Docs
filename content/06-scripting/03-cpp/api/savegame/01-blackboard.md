---
title: "The blackboard"
---

A key/value store that outlives the scene, and the storage behind both kinds of save file.

`GameState` is **one instance per process**, not per scene. A value written here survives a scene
switch, a level change and a game load. That is what makes it the place for anything a game
carries: a score, an unlocked door, a quest stage, the fact that a character has already been
met.

It is also exactly what a save file's own `GameState` block round-trips, so nothing here needs
separate saving logic.

## The four types

A value is a number, a bool, a string or a vector, and **the type is part of the key**.

```c
void (*SetNumber)(const char* key, double value);
void (*SetBool)(const char* key, int value);
int  (*SetText)(const char* key, const char* value);
void (*SetVec3)(const char* key, LAB_Vec3 value);
```

```cpp
void LABNative::SaveGame::SetNumber(const char* key, double value);
void LABNative::SaveGame::SetBool(const char* key, int value);
bool LABNative::SaveGame::SetText(const char* key, const char* value);
void LABNative::SaveGame::SetVec3(const char* key, LAB_Vec3 value);
```

Writing a key that already holds a value of another type **replaces it**, type included. So there
is no coercion anywhere: a key holds what it was last written as.

`SetText` answers 1 on success and 0 for a null key or a null value. The others are no-ops for a
null key, and write nothing.

```cpp
LABNative::SaveGame::SetNumber("Score", 1240.0);
LABNative::SaveGame::SetBool("DoorOpen", 1);
LABNative::SaveGame::SetText("LastCheckpoint", "crypt");
LABNative::SaveGame::SetVec3("SpawnPoint", LAB_Vec3{ 3.0f, 4.0f, 0.0f });
```

## Reading

```c
int (*GetNumber)(const char* key, double* out);
int (*GetBool)(const char* key, int* out);
int (*GetText)(const char* key, char* buffer, size_t capacity);
int (*GetVec3)(const char* key, LAB_Vec3* out);
```

```cpp
bool LABNative::SaveGame::GetNumber(const char* key, double* out);
bool LABNative::SaveGame::GetBool(const char* key, int* out);
int  LABNative::SaveGame::GetText(const char* key, char* buffer, size_t capacity);
bool LABNative::SaveGame::GetVec3(const char* key, LAB_Vec3* out);
```

Each answers **1 when the key holds a value of that type**, and 0 otherwise.

That is the important design decision here. A getter answers 0 both for a key that is missing and
for a key holding a value of a *different shape*, so a caller learns it asked for the wrong type
rather than reading a converted one. A number stored as text does not silently become its first
digit.

`GetText` is the odd one out, because it has a buffer to report on: it answers the **length
written**, or **-1** for a key that is not a string. So `>= 0` is "there was a string" and `-1`
is "there was not".

**A null `out` is not a way to ask about the type.** All three pointer getters answer 0 for one,
so a call with no variable to write into is always false rather than "it is there but I do not
want it". Pass a real variable and ignore it, or ask `Has` when the type does not matter.

```cpp
double score = 0.0;
if (LABNative::SaveGame::GetNumber("Score", &score))
	SetScoreText(score);
else
	SetScoreText(0.0);      // never saved, or saved as something else

char checkpoint[64];
if (LABNative::SaveGame::GetText("LastCheckpoint", checkpoint, sizeof(checkpoint)) >= 0)
	LABNative::LogInfo(checkpoint);
```

## `Has`, `Remove`, `Clear`

```c
int  (*Has)(const char* key);
void (*Remove)(const char* key);
void (*Clear)(void);
```

```cpp
bool LABNative::SaveGame::Has(const char* key);
void LABNative::SaveGame::Remove(const char* key);
void LABNative::SaveGame::Clear();
```

`Has` is the type-blind question: it is 1 for a key present whatever it holds. It is the call for
"has this game ever been played through here", where the value's shape does not matter.

**`Clear` is what a New Game calls**, and it touches no entity on its own. It empties the
blackboard and nothing else, so a caller wanting a genuinely fresh start wants a scene reload
too. Clearing without reloading leaves the world exactly as it was and only forgets what the game
knew about it:

```cpp
LABNative::SaveGame::Clear();
LABNative::Streaming::OpenScene("scenes/level_01.Lscene");
```

## When to use it

The blackboard earns its place the moment state has to outlive a scene. Anything that is
per-entity belongs on the entity, as a component or a behaviour member; anything that is per-run
belongs here.

| State | Where |
|---|---|
| This enemy's health | On the entity |
| This level's score | Blackboard |
| Whether a secret door has been opened | Blackboard |
| Player position within a level | The scene, and the save slot |
| Player position across a level change | A `PersistentComponent` |

Three mechanisms with three lifetimes, and picking the wrong one is how a game ends up forgetting
something important on a level change or remembering something it should have reset.

## A complete example

A collectable that remembers what has already been taken, so a re-entered scene does not hand the
same thing out twice.

```cpp
class Collectable : public LABNative::Behaviour
{
public:
	void OnCreate() override
	{
		// Keyed by this entity's own name, which is stable in the scene file.
		const std::string key = "taken:" + GetName();

		int taken = 0;
		if (LABNative::SaveGame::GetBool(key.c_str(), &taken) && taken)
			Destroy();     // already collected on an earlier visit
	}

	void OnOverlapBegin(LAB_EntityId other) override
	{
		const std::string key = "taken:" + GetName();
		LABNative::SaveGame::SetBool(key.c_str(), 1);

		double count = 0.0;
		LABNative::SaveGame::GetNumber("Gems", &count);
		LABNative::SaveGame::SetNumber("Gems", count + 1.0);

		LABNative::EmitParticles(GetEntity(), 20);
		LABNative::PlaySound(GetEntity());
		Destroy();
	}
};
```

The key namespacing (`"taken:"`) is worth copying. The blackboard is one flat namespace shared by
every system in the game, so a prefix per system is what keeps two of them from writing the same
key.
