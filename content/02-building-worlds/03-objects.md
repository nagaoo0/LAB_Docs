---
title: "Objects (Prefabs)"
---

An **object** is a saved entity — with all its components and all its children — that you can
place in as many scenes as you like and edit in one place. Other engines call these prefabs.

Objects are `.Lobj` files under your project's asset folder. The format is exactly the YAML
the scene serializer writes for one entity, so any component that works in a scene works in
an object automatically.

## Making one

Build the entity in a scene, then right-click it in the **Scene Hierarchy** → **Save as
Object…** and choose a path.

**Children are included.** An object with no hierarchy is barely worth having — a turret is
a base and a barrel, and saving only the base would be a trap.

## Placing one

- Drag the `.Lobj` from the Asset Browser into the viewport. It lands where the cursor
  points.
- Or from a script: `scene.spawn("objects/turret.Lobj", position)`.

Each placement is a full copy of the entity tree. The root carries a marker recording which
file it came from — that is what lets a later edit find it again.

Objects cannot be dropped while play mode is running, because the scene being simulated is a
copy that is about to be discarded.

## Editing one

Right-click an instance in the hierarchy → **Edit Object…**, or double-click the `.Lobj` in
the Asset Browser.

This opens the object **on its own**. The scene you had open is stashed and comes back
untouched when you close. The viewport toolbar shows an **OBJECT** badge, the object's path,
and **Save Object** / **Close Object** buttons.

A few things about object editing mode:

- **`Ctrl+S` saves the object, not the scene.** The scene is stashed and nothing has been
  done to it.
- **Play is unavailable.** The scene that would be simulated is the object's throwaway one,
  not the scene you have open.
- **A light is added for you.** An object opened on its own in a scene with no lights would
  look broken, so the editor adds one. It is a sibling of the root rather than a descendant,
  so it is never written into the file — the object is exactly what was in it.
- **Only one object at a time.** Nested object editing would need a stack of stashes, and an
  object containing another object is a reference the format does not have.

Deleting the root while editing leaves nothing sensible to write, so save first if you meant
to keep it.

## What happens when you save

Saving an object **rebuilds every instance of it already placed in the open scene**. Without
that, editing an object would appear to do nothing: the copies would keep whatever they were
instantiated with.

What survives the refresh is the instance's **placement**:

| Kept from the instance | Taken from the file |
|---|---|
| Its transform | Every component on the root |
| Its parent | The entire hierarchy below the root |
| Its name, if you renamed it | |

**Local edits below the root are lost.** If you moved a child of an instance, that change is
overwritten by what is in the file. This is the honest contract for a format with no override
tracking — silently keeping local edits while claiming to have updated would be worse.

Instances in *other* scenes update when you next open them, since they are rebuilt from the
file at load.

## When to use one

Use an object when the same thing appears more than once and you expect to change it: a
crate, an enemy, a light fixture, a modular wall segment. Use a plain duplicated entity when
the copies are meant to diverge.

Because an object is just an entity tree, an object can carry a Script, a Rigid Body, lights,
and any hierarchy you like. `scene.spawn` gives you a complete, live copy — its meshes upload
and its rigid bodies fall without any further work.

## Test scene

`Dev/Tests/assets/scenes/object_test.Lscene` exercises placing, editing and refreshing objects.
