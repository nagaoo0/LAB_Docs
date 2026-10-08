---
title: "Scene streaming"
---

Loading another scene at runtime: warming a scene file into the cache, adding its entities to the
running scene, and switching the running scene over to it. See
[Lua Scripting](../../index.md).

| Call | What it does | When it runs |
|---|---|---|
| `preload_scene` | Parses and caches the file, creating nothing | Immediately |
| `append_scene` | Adds the file's entities to the running scene | Queued to the end of the frame |
| `open_scene` | Destroys everything without a `Persistent` flag, then appends the file | Queued to the end of the frame |

Paths are project-relative, the same string a scene file holds: `"scenes/level_04.Lscene"`, never
a path with a drive letter on it. That is what keeps a project folder movable and lets a packaged
build resolve the same string out of its archive.

The same three operations exist for C++ modules, with the same queuing rules, so a level
transition designed in one language behaves identically in the other: see
[Scene streaming](../../../03-cpp/api/streaming/01-scene-streaming.md).

## `scene.preload_scene(path)`

```lua
local ready = scene.preload_scene("scenes/level_04.Lscene")
if not ready then
    log.warn("the next level did not resolve")
end
```

Parses a scene and caches it, without touching the running one, and answers **true** when it
loaded and **false** when the path does not resolve or does not parse. Nothing becomes visible and
no entity is created.

**This one runs immediately**, because it only parses and caches: nothing is created or destroyed,
so there is nothing for it to corrupt. It is the call that makes a loading screen honest, since
preloading the next level while the player is still in this one means the open that follows costs
no disk time. The cache is the engine's ordinary scene cache, so preloading something already
cached is cheap and harmless, and a later append or open against the same path skips the read and
the parse.

## `scene.append_scene(path)`

Adds the file's entities to the running scene, alongside whatever is already there. **Nothing is
destroyed**: this is additive, which is what makes it the right call for a room that opens onto
another, a streamed chunk, or a boss arena added to the level around it.

```lua
function on_overlap_begin(other)
    scene.append_scene("scenes/arena_b.Lscene")
end
```

**It answers nothing and it is queued**, so the call returns before the entities exist. The check
for "did it work" is what the scene looks like next frame. See [Why two of them are
queued](#why-two-of-them-are-queued) below.

An appended scene's entities are ordinary entities: mesh uploads, physics bodies and their
scripts' `on_create` all happen exactly as they would for a `scene.spawn`. The running scene's own
name and physics settings are left alone, because this is an addition rather than a takeover.

## `scene.open_scene(path)`

Switches the scene over to another one wholesale: every entity without a `Persistent` flag is
destroyed first, then the file's entities are appended. The scene's name, its gravity and its
collision layers are replaced from the file, since this is a level change rather than an addition.

```lua
function on_update(dt)
    if input.is_key_pressed("Escape") then
        scene.open_scene("scenes/main_menu.Lscene")
    end
end
```

**It answers nothing and it is queued**, like `append_scene`.

**Everything that is not marked persistent goes.** Call `entity:set_persistent(true)` on anything
that has to keep running across the switch: a game manager, a player controller, a HUD script. The
entities' scripts get their `on_destroy`, and a persistent entity whose parent is not persistent
is detached to the scene root first, keeping its world placement, rather than being taken down
with its parent.

## Why two of them are queued

`append_scene` and `open_scene` are queued to the end of the frame, not run on the spot.
`preload_scene` is the exception, for the reason above.

The queue is not a nicety. Both can destroy and create entities, and a script reaching either from
inside its own `on_update` is running from inside the very loop the engine is stepping every
script instance through. An entity destroyed there would free the instance whose function is still
on the stack. Queuing them means the work lands after that frame's scripts have finished, in the
same slot a C++ module's `AppendScene`/`OpenScene` uses.

Three consequences for calling code:

- **Read the result, not the return.** Nothing is returned, so the check for "did it work" is what
  the scene looks like next frame. `Dev/Tests/assets/tests/streaming.lua` is a working example of checking a
  frame later.
- **Do not assume it happened before the next line.** After `open_scene`, the current scene is
  still there for the remainder of your callback, and so is the script that called it.
- **Only the most recent request in a frame is applied.** Queueing an append and then an open in
  one frame applies the open, and two opens apply the second.

## A complete example

A level exit that preloads the next scene at the start and opens it on contact.

```lua
local NEXT_SCENE = "scenes/level_04.Lscene"

local ready = false

function on_create()
    entity:set_persistent(true)
    ready = scene.preload_scene(NEXT_SCENE)
end

function on_overlap_begin(other)
    if not ready then
        return
    end

    scene.open_scene(NEXT_SCENE)
end
```

Note the `set_persistent` call. The entity that asks for the level change is destroyed by it unless
it carries the flag, so the flag is the mechanism, not the call. Note also that the `open_scene`
call is the last statement in its callback on purpose: everything below it would still run against
the scene that is on its way out.
