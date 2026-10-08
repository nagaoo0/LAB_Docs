---
title: "Navmesh and behaviour trees"
---

Steering a nav agent, and reading or injecting into a behaviour tree that is already running.

Every call here is reached through `api->AI` or the `LABNative::AI` namespace. All of them need
play mode: the navmesh, the agents and the trees are all simulation state.

## `MoveTo`

```c
int (*MoveTo)(LAB_EntityId entity, LAB_Vec3 target);
```

```cpp
bool LABNative::AI::MoveTo(LAB_EntityId entity, LAB_Vec3 target);
```

Plans a path across the scene's baked navmesh and starts following it. Answers 1 on success, 0
when there is no loaded navmesh or no path exists.

**It plans and follows; it does not move anything on the spot.** An agent's movement happens in
the scene's own update, once per frame, so this call sets a destination and returns long before
the entity goes anywhere. That is why it answers "was a path found" rather than "did it arrive",
and why there is a separate call to ask about arrival.

**An entity with no `NavAgentComponent` gets one, with its defaults.** So a behaviour can steer
something that was never authored as an agent:

```cpp
if (!LABNative::AI::MoveTo(GetEntity(), patrolPoint))
	LABNative::LogWarn("no path to the patrol point");
```

An agent paired with a `CharacterControllerComponent` is steered before the physics step, so a
move takes effect on the step that follows the call. A scene that only bakes its navmesh from a
script's `on_update` will find none loaded on the first frame, which is
[a real trap](#a-navmesh-has-to-exist-before-the-first-tick) below.

## `IsMoveComplete`

```c
int (*IsMoveComplete)(LAB_EntityId entity);
```

```cpp
bool LABNative::AI::IsMoveComplete(LAB_EntityId entity);
```

1 when the agent is walking nowhere, 0 while it is following a path.

An entity with no agent at all also answers 1, which is the answer the facade gives for anything
it cannot resolve. So "the agent arrived" and "there is no agent" are the same observation, and
a behaviour that needs to tell them apart should check for the component or hold a flag from its
own `MoveTo`.

## `StopMoving`

```c
void (*StopMoving)(LAB_EntityId entity);
```

```cpp
void LABNative::AI::StopMoving(LAB_EntityId entity);
```

Cancels the current path. The agent stays where it is, and `IsMoveComplete` then answers 1.

```cpp
// Interrupt a patrol because something more important came up.
LABNative::AI::StopMoving(GetEntity());
LABNative::AI::MoveTo(GetEntity(), threatPosition);
```

## `GetCurrentState`

```c
int (*GetCurrentState)(LAB_EntityId entity, char* buffer, size_t capacity);
```

```cpp
int LABNative::AI::GetCurrentState(LAB_EntityId entity, char* buffer, size_t capacity);
```

Copies the name of the state a running behaviour tree is in into `buffer`, null-terminated and
truncated if it does not fit, and answers the length written. Answers **-1** when the tree has
not started, or has no current state.

```cpp
char state[64];
if (LABNative::AI::GetCurrentState(GetEntity(), state, sizeof(state)) >= 0)
	LABNative::LogInfo(state);
```

The same copy-into-your-buffer rule every string on this boundary follows: the engine never hands
a module a pointer into its own storage.

This is the read side of a behaviour tree, for a behaviour that wants to react to what the AI
decided (play a sound on `"Attack"`, change a HUD icon on `"Alert"`).

## `GetBlackboard` / `SetBlackboard`

```c
float (*GetBlackboard)(LAB_EntityId entity, const char* name);
int (*SetBlackboard)(LAB_EntityId entity, const char* name, float value);
```

```cpp
float LABNative::AI::GetBlackboard(LAB_EntityId entity, const char* name);
bool LABNative::AI::SetBlackboard(LAB_EntityId entity, const char* name, float value);
```

Reads and writes a behaviour tree's named parameters, which is live-value injection rather than
authoring the tree.

Both answer 0 for an entity with no `AIControllerComponent`, and for a name the tree does not
have: the same tolerance a mistyped parameter name already gets everywhere else.

```cpp
// Tell the tree the player has been seen, without touching the tree itself.
LABNative::AI::SetBlackboard(GetEntity(), "PlayerVisible", 1.0f);

// Have the tree's own task decide this, and react to it.
if (LABNative::AI::GetBlackboard(GetEntity(), "Alert") > 0.0f)
	LABNative::PlaySound(GetEntity());
```

The second half of that example is the reason these exist. A behaviour tree task can be **bound**
to a blackboard parameter rather than holding a fixed value, and a bound field re-reads the named
parameter fresh **on every tick** rather than freezing at whatever it held when the field was
authored. So a value a behaviour writes is picked up by the tree's own tasks on the next tick,
with no wiring.

## `BakeNavMesh`

```c
int (*BakeNavMesh)(LAB_EntityId volumeId);
```

```cpp
bool LABNative::AI::BakeNavMesh(LAB_EntityId volumeId);
```

Bakes a `NavMeshVolumeComponent`'s navmesh and reloads the scene's own copy from it. This is the
same pair of calls the editor's **Bake Navmesh** button makes. Answers 1 on success, 0 on
failure with the reason in the log.

**It writes the navmesh file the volume names.** A caller that does not mean to change the file
on disk should not call this: baking at run time overwrites what the project has, not a copy. For
a game that bakes procedurally, that is the point; for anything else, it is a surprise.

## A navmesh has to exist before the first tick

Worth knowing when a tree looks like it is skipping its movement state.

An `AIControllerComponent`'s **entry state starts evaluating on the very first frame**, before
that frame's own script `on_update` has run. So a `MoveTo` task in the entry state calls
`FindPath` on frame one.

A scene that bakes its navmesh from a script's `on_update`, rather than shipping with
`NavMeshVolumeComponent::BakedAssetPath` already pointing at a baked file, therefore finds no
navmesh loaded yet. The task fails immediately. And because a transition gated on "the state's
tasks are complete" does not distinguish *succeeded* from *failed*, the tree moves straight past
the state that never got to run.

The symptom is an agent that never moves and a tree that looks like it worked. The fix is to have
the navmesh baked and loaded before play starts: point the volume at a baked file, or bake from
`on_create`, which runs before frame one.

## What is not here

There is no way to create or edit a behaviour tree from a module, only to read and inject into
one that is running. Tree structure is authored: it is an asset, edited in the editor, and a
`.Lbehavior` file on disk.

There is also no navmesh query beyond `MoveTo`: no raycast against the navmesh, no "find the
nearest point on it", and no way to ask about the path a `MoveTo` planned.
