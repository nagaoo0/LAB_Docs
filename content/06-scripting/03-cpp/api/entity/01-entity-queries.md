---
title: "Entity queries"
---

Finding an entity, checking it is still there, reading its name, and destroying it.

## Entity ids

```c
typedef uint64_t LAB_EntityId;
```

An entity id is the **UUID** the scene itself uses, not an `entt` handle: the same value a Lua
script names an entity by and the same one written into a `.Lscene` file. So an id is stable
across a scene reload, a play-mode copy, a save and a load, which an index into a registry's
storage would not be.

**`0` is never a valid entity.** `Scene::RegisterID` never mints it, so it doubles as "nothing"
wherever a call returns one, and as the thing to compare against when checking whether a lookup
succeeded.

Ids are drawn from `[1, 2^53)`, not the full 64-bit range, because Lua is the consumer that has
to hold one exactly and anything above 2^53 stops being exact the moment it passes through a
float.

## `FindEntityByName`

```c
LAB_EntityId (*FindEntityByName)(const char* name);
```

```cpp
LAB_EntityId LABNative::FindEntityByName(const char* name);
```

The entity with this name, or `0` when there is none.

The name is the entity's tag as shown in the Hierarchy panel. Names are not unique; when two
entities share one, this answers the first it finds, so use it to find a well-known entity and
not to identify one.

There is no "find by id" call, because an id *is* the handle: keep the id you were given.

```cpp
const LAB_EntityId door = LABNative::FindEntityByName("Level Door");
if (door)
	LABNative::SetPosition(door, LAB_Vec3{ 0.0f, 0.0f, 2.0f });
```

## `IsEntityValid`

```c
int (*IsEntityValid)(LAB_EntityId entity);
```

```cpp
bool LABNative::IsEntityValid(LAB_EntityId entity);
```

Whether this id still names a live entity.

Worth calling when you hold an id across frames. An entity can be destroyed by a script, by a
gameplay rule, or by the level changing, and an id you stored is then a stale number. Storing an
id is still the right thing to do; the check is what makes it safe.

```cpp
if (LABNative::IsEntityValid(m_Target))
	LABNative::AI::MoveTo(GetEntity(), LABNative::GetPosition(m_Target));
```

Note that every other call is tolerant of a stale id anyway: it becomes a no-op or a zero
answer rather than a crash. `IsEntityValid` is for when you want to *decide* something, not for
safety.

## `GetName`

```c
int (*GetName)(LAB_EntityId entity, char* buffer, size_t capacity);
```

Fills `buffer` with the entity's name, null-terminated and truncated if it does not fit, and
answers the length written **excluding** the terminator, or `-1` when the entity does not
resolve.

```cpp
char buffer[64];
if (g_API->GetName(entity, buffer, sizeof(buffer)) >= 0)
	LABNative::LogInfo(buffer);
```

The engine never hands a module a pointer into its own string storage. That rule applies to
every string on this boundary, and it is why this call copies into a buffer you own.

In C++ this is wrapped for you:

```cpp
std::string LABNative::Behaviour::GetName() const;
```

which copies into a 256-byte local buffer and returns a `std::string`, empty when the entity
does not resolve. 256 bytes is generous for an entity tag; a longer one is truncated.

## `DestroyEntity`

```c
void (*DestroyEntity)(LAB_EntityId entity);
```

```cpp
void LABNative::DestroyEntity(LAB_EntityId entity);
void LABNative::Behaviour::Destroy() const;   // this behaviour's own entity
```

Destroys the entity and everything it owns. A no-op for an id that does not resolve.

**It is safe to destroy the entity a behaviour is running on, from inside that behaviour's own
callback.** The engine defers the actual free until the callback that asked for it returns, so
`this` and `m_Entity` stay valid for the rest of the call you made it from, and `OnDestroy` gets
called in the ordinary way.

```cpp
void OnUpdate(float dt) override
{
	m_Life -= dt;
	if (m_Life <= 0.0f)
		Destroy();          // fine: the rest of this function still has a valid `this`
}
```

That deferral is why a script or a behaviour may destroy entities from an update callback at
all. Without it, the common case (a projectile that hits something and removes itself) would
free the object whose function is still executing.

Destroying an entity **whose physics body is in the simulation** is also ordinary: the body goes
with it, and no other behaviour sees a stale contact for it.

## The tolerance contract

Every entity call in this API is a silent no-op, or answers the type's zero value, when there is
no bound scene or the id does not resolve. That is the same contract the Lua facade gives a
script.

It is deliberate: a behaviour does not necessarily know what it is attached to, and asking the
wrong entity should not be a failure. The cost is that a genuinely wrong id is quiet, so
`IsEntityValid` and a log line are how you find one.
