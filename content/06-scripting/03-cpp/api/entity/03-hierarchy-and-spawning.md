---
title: "Hierarchy and spawning"
---

Making new entities, attaching them to others, and walking the tree.

## `SpawnEntity` / `SpawnEntityAt`

```c
LAB_EntityId (*SpawnEntity)(const char* objectPath);
LAB_EntityId (*SpawnEntityAt)(const char* objectPath, LAB_Vec3 position);
```

```cpp
LAB_EntityId LABNative::Spawn(const char* objectPath);
LAB_EntityId LABNative::SpawnAt(const char* objectPath, LAB_Vec3 position);
```

Instantiates an object or prefab and answers the new entity's id, or `0` when the path does not
resolve.

`objectPath` is **project-relative**, the same string a scene file holds:
`"objects/crate.Lobj"`, `"scenes/level_02.Lscene"` is not one of these, and neither is a path
with a drive letter on it. Spawning is object and prefab instantiation, not scene loading, which
is [Scene streaming](../streaming/01-scene-streaming.md).

**By the time the call returns, the entity is whole.** Its rigid body is in the simulation and
its mesh upload is queued. That is not a detail: spawning goes through the same path a scene's
own instantiation uses, which already requests the mesh upload and notifies the scene that the
entity exists. A second spawn path that skipped either would produce an entity that is invisible
or does not fall, and the symptom arrives a frame later somewhere else.

```cpp
const LAB_EntityId crate = LABNative::SpawnAt("objects/crate.Lobj", LAB_Vec3{ 3.0f, 4.0f, 5.0f });
if (!crate)
	LABNative::LogWarn("crate object did not resolve");
```

The object's own transform is used as given, so `SpawnEntityAt` places the instantiated
*root* at that world position. Children of the object keep their local offsets.

Spawning is also how a behaviour starts another behaviour: an object carrying a
`NativeScriptComponent` starts its class on instantiation, exactly as it would from a scene.

## `SetParent`

```c
int (*SetParent)(LAB_EntityId child, LAB_EntityId parent);
```

```cpp
bool LABNative::SetParent(LAB_EntityId child, LAB_EntityId parent);
```

Makes `parent` the parent of `child`, and answers 1 on success. Answers `0` when either entity
does not resolve, or when the move would put a parent inside its own child.

**World placement is preserved**, the same as dragging one entity onto another in the editor. The
child's local values change so that its world position, rotation and scale stay where they were:

```cpp
const LAB_EntityId hat = LABNative::Spawn("objects/hat.Lobj");
LABNative::SetParent(hat, GetEntity());     // stays where it was, now follows the entity
```

The consequence to know about: a child's *local* values after this call are not the ones you
would get by reading the object file. They are whatever produces the same world placement under
the new parent.

Passing `0` as the parent is how you unparent, and it preserves world placement in the same way.

### The one case that cannot be exact

A world-preserving rebase cannot represent a child exactly when the new parent is rotated *and*
non-uniformly scaled. The engine's transform is translation, rotation and scale with no shear
term, and that combination needs one, so the closest fit is used and the child's scale visibly
changes.

World position is never affected. Rotation and scale can shift slightly, and only under a parent
that is both rotated and stretched. Reparenting under a uniformly scaled parent, however large,
is exact.

## `GetChildren`

```c
int (*GetChildren)(LAB_EntityId entity, LAB_EntityId* out, int max);
```

```cpp
int LABNative::GetChildren(LAB_EntityId entity, LAB_EntityId* out, int max);
int LABNative::ChildCount(LAB_EntityId entity);   // GetChildren(entity, nullptr, 0)
```

Writes the entity's children into an array you own, and answers **how many children it has**.

The count is not capped by `max`: pass a null array or a zero `max` to ask the count, then size
an array for it and ask again. More children than `max` leaves the rest out rather than writing
past the end of your array.

```cpp
const int count = LABNative::GetChildren(parent, nullptr, 0);
std::vector<LAB_EntityId> children(static_cast<size_t>(count));
LABNative::GetChildren(parent, children.data(), count);
```

The order is the order the children were attached in, which is not something to depend on
beyond reading them all.

**Do not mutate the hierarchy while walking it.** Attaching or destroying an entity while
iterating children can invalidate what you are iterating over. Collect the ids first, then act.

## `GetParent`

```c
LAB_EntityId (*GetParent)(LAB_EntityId entity);
```

```cpp
LAB_EntityId LABNative::GetParent(LAB_EntityId entity);
```

The entity's parent, or `0` when it has none. The one direction of the hierarchy that
`SetParent` and `GetChildren` did not already cover.

```cpp
// Walk to the root of whatever this entity is attached to.
LAB_EntityId root = GetEntity();
while (LAB_EntityId parent = LABNative::GetParent(root))
	root = parent;
```

## A complete example

Give every child of a named entity a marker, then detach one of them.

```cpp
void OnCreate() override
{
	const LAB_EntityId parent = LABNative::FindEntityByName("Crate Stack");
	if (!parent)
		return;

	// Collect the ids first. Attaching or destroying while walking the hierarchy can
	// invalidate what you are iterating over, so act on a copy of the list.
	std::vector<LAB_EntityId> children(static_cast<size_t>(LABNative::ChildCount(parent)));
	if (!children.empty())
		LABNative::GetChildren(parent, children.data(), static_cast<int>(children.size()));

	for (LAB_EntityId child : children)
	{
		const LAB_EntityId marker = LABNative::SpawnAt("objects/marker.Lobj", LABNative::GetWorldPosition(child));
		if (marker)
			LABNative::SetParent(marker, child);   // world placement preserved
	}

	if (!children.empty())
		LABNative::SetParent(children.front(), 0);   // detach the first, still in place
}
```
