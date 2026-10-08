---
title: "Scene streaming"
---

Preloading, appending and opening scenes from a behaviour.

These are the same three operations a Lua script has, with the same queuing rules, so a level
transition designed in one language behaves identically in the other.

## `PreloadScene`

```c
int (*PreloadScene)(const char* path);
```

```cpp
bool LABNative::Streaming::PreloadScene(const char* path);
```

Parses a scene and caches it, without touching the running one. Answers 1 on success, 0 when the
path does not resolve or does not parse.

**This one runs immediately**, because it only parses and caches: nothing is created or destroyed,
so there is nothing to defer. It is the call to make a loading screen honest: preload the next
level while the player is still in this one, and the open that follows costs no disk time.

```cpp
void OnCreate() override
{
	// Get the next level into memory while there is time to spare.
	LABNative::Streaming::PreloadScene("scenes/level_03.Lscene");
}
```

The cache is the engine's ordinary scene cache, so preloading something already loaded is cheap
and harmless.

## `AppendScene`

```c
void (*AppendScene)(const char* path);
```

```cpp
void LABNative::Streaming::AppendScene(const char* path);
```

Adds the scene's entities to the running one. **Nothing is destroyed**: this is additive, which is
what makes it the right call for a room that opens onto another, a streamed chunk, or a boss arena
added to the level around it.

```cpp
LABNative::Streaming::AppendScene("scenes/arena_b.Lscene");
```

An appended scene's entities are ordinary entities. Its scripts and behaviours start as they
would in a scene open, and its persistent entities follow the persistent rules.

## `OpenScene`

```c
void (*OpenScene)(const char* path);
```

```cpp
void LABNative::Streaming::OpenScene(const char* path);
```

Replaces the running scene, **destroying every entity without a `PersistentComponent`** and every
script and behaviour running on one.

```cpp
LABNative::Streaming::OpenScene("scenes/level_04.Lscene");
```

This is a level change. Everything that is not marked persistent goes away, including the
behaviour that called this, so the rest of that callback runs with the callback's own frame
already committed to dying. Nothing after this call in the same callback should be relied on.

## Why two of them are queued

`AppendScene` and `OpenScene` are **queued to the end of the frame**, not run on the spot, and
they answer immediately. `PreloadScene` is the exception, for the reason above.

The queue is not a nicety. Both can destroy and create entities, and a behaviour reaching either
from inside its own `OnUpdate` is running *inside the very loop that is walking its instances*. A
destroy there would free the object whose method is still on the stack. Queuing them means the
work lands after the frame's own iteration is done, in the same slot a Lua script's
`scene.append_scene` and `scene.open_scene` use.

Two consequences for calling code:

- **Read the result, not the return.** There is nothing to return, so the check for "did it work"
  is what the scene looks like next frame.
- **Do not assume it happened before the next line.** After `OpenScene`, the current scene is
  still there for the remainder of your callback.

## Persistent entities

A `PersistentComponent` is what survives an `OpenScene`: the player's stats, an inventory, a
score, anything a level change should not reset. The engine carries those entities' live state
across rather than re-reading them from the new scene file, which is the whole point of the
component.

That makes `OpenScene` a **level transition**, not a reset. If you want a reset, the save game's
`Clear` plus a scene reload is the combination that produces one; see
[Save slots](../savegame/02-save-slots.md).

A persistent entity parented under a non-persistent one is reparented to the root on the way
across rather than being destroyed with its parent, so an entity that has to survive does, even
if something above it in the hierarchy does not.

## Paths

Scene paths are **project-relative**, the same string a scene file holds: `"scenes/level_04.Lscene"`,
never a path with a drive letter on it. That is what keeps a project folder movable, and what lets
a packaged build resolve the same string out of its archive.

## A complete example

A level exit that preloads the next scene while the player approaches and opens it on contact.

```cpp
class LevelExit : public LABNative::Behaviour
{
public:
	const char* NextScene = "scenes/level_04.Lscene";   // an authored field in practice

	void OnCreate() override
	{
		// Cheaper to do once at the start than to do at the door.
		m_Ready = LABNative::Streaming::PreloadScene(NextScene);
		if (!m_Ready)
			LABNative::LogWarn("the next scene did not resolve");
	}

	void OnOverlapBegin(LAB_EntityId other) override
	{
		if (!m_Ready)
			return;

		LABNative::Streaming::OpenScene(NextScene);
		// Queued: this behaviour is still alive for the rest of this callback, and will be
		// destroyed with everything else at the end of the frame.
	}

private:
	bool m_Ready = false;
};
```

Note that a behaviour needing to survive that transition is one on an entity with a
`PersistentComponent`, not one that calls `OpenScene`. The component is the mechanism, and it is
what the engine carries.
