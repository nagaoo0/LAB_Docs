---
title: "Module lifecycle and registration"
---

What a module exports, what the engine gives it, and how a behaviour class comes into being.

## `LAB_ModuleInit`

```c
int LAB_ModuleInit(const LAB_EngineAPI* api, LAB_ModuleRegistry* registry);
```

The module's one required export, and the only symbol the engine looks up. Called once when
the library is loaded.

```cpp
extern "C" LAB_NATIVE_EXPORT int LAB_ModuleInit(const LAB_EngineAPI* api, LAB_ModuleRegistry* registry)
{
	LABNative::Init(api);
	LAB_REGISTER_BEHAVIOUR(registry, Mover);
	return LAB_NATIVE_API_VERSION;
}
```

**Return `LAB_NATIVE_API_VERSION` as your own copy of the header defines it.** Never a number
written by hand, and never a value this engine is known to accept. The return is a claim about
which entry points the module may call, and a typed-in number is a lie about that which fails
silently later. [ABI and versioning](02-abi-and-versioning.md) has the detail.

`LABNative::Init(api)` must come first. It stores the table every wrapper function forwards to,
so a call made before it finds no engine.

The engine calls this at two moments:

| When | Why |
|---|---|
| The editor opens a scene | So the inspector can offer the registered class names. No instance is created |
| Play starts, or the runtime starts | For real: instances are created, callbacks are ticked |

Only play mode runs behaviours. The editor loads the library to read its registry, which is
why a module with a static initializer that touches the scene is a bad idea.

## `LAB_ModuleRegistry`

Handed to `LAB_ModuleInit`. The engine allocates it, so a module built against an older header
never reads a field the engine appended.

| Field | |
|---|---|
| `void* Self` | An opaque host pointer. Pass it back unchanged to every call below |
| `int (*RegisterBehaviour)(void* self, const LAB_BehaviourDesc* desc)` | Registers a behaviour class. 1 on success |
| `int (*RegisterFunction)(void* self, const char* name, LAB_LuaFunctionFn fn)` | Makes a function reachable from Lua as `native.call(name, ...)`. 1 on success |

`RegisterBehaviour` answers 0 when the descriptor is malformed (no `StructSize`, `Name`,
`Create` or `Destroy`, or a `StructSize` that does not reach the callbacks it claims) or when
the name is already registered.

`RegisterFunction` answers 0 for a null or empty name, a null function, or a duplicate name.
`name` has to outlive the module, like a behaviour's own name. See
[Lua interop](../lua/01-lua-interop.md).

## Registering a behaviour

The macro form registers a class under its own type name, which is the string an author picks
in the inspector:

```cpp
LAB_REGISTER_BEHAVIOUR(registry, Mover);
```

The explicit form chooses the name, and is the only way to declare authorable fields:

```cpp
LABNative::RegisterBehaviour<Tuner>(registry, "Tuner", kTunerFields, 4);
```

Both live in `LABNative.hpp` and expand into a `LAB_BehaviourDesc` plus one thunk per callback,
so the engine never has to know the module was written in C++ at all.

The name must match what an entity's `NativeScriptComponent` names, or the behaviour never
runs. That mismatch is the most common mistake there is, and it is quiet: the scene loads, the
entity looks right, nothing moves. The inspector's status line and `native_classes()` both
answer it without a rebuild.

## `LAB_BehaviourDesc`

What the module fills in and the engine keeps.

| Field | |
|---|---|
| `uint32_t StructSize` | `sizeof` of the descriptor the module actually built. The engine reads only the callbacks inside it |
| `const char* Name` | The class name an author picks. Must outlive the module |
| `LAB_BehaviourCreateFn Create` | Required. Returns an opaque instance pointer the module owns |
| `LAB_BehaviourDestroyFn Destroy` | Required. Frees it |
| `LAB_BehaviourOnCreateFn OnCreate` | May be null |
| `LAB_BehaviourOnUpdateFn OnUpdate` | May be null |
| `LAB_BehaviourOnDestroyFn OnDestroy` | May be null |
| `LAB_BehaviourOnContactFn OnCollisionBegin` | May be null |
| `LAB_BehaviourOnContactFn OnCollisionEnd` | May be null |
| `LAB_BehaviourOnContactFn OnOverlapBegin` | May be null |
| `LAB_BehaviourOnContactFn OnOverlapEnd` | May be null |
| `void (*OnFixedUpdate)(void* instance, float dt)` | May be null |
| `const LAB_FieldDesc* Fields` | May be null. [Declared fields](04-declared-fields.md) |
| `uint32_t FieldCount` | Zero when `Fields` is null |

`StructSize` is the mechanism that lets the descriptor grow. A module built against an older
header builds a shorter struct and zero-fills what it does not know about, and the engine reads
a callback only where the size reaches it. Appending an entry to this struct is therefore a
minor change, and entries go at the end.

`LABNative.hpp`'s `RegisterBehaviour<T>` sets all of it for you.

## Callbacks

```c
typedef void* (*LAB_BehaviourCreateFn)(LAB_EntityId entity);
typedef void  (*LAB_BehaviourDestroyFn)(void* instance);
typedef void  (*LAB_BehaviourOnCreateFn)(void* instance);
typedef void  (*LAB_BehaviourOnUpdateFn)(void* instance, float deltaTime);
typedef void  (*LAB_BehaviourOnDestroyFn)(void* instance);
typedef void  (*LAB_BehaviourOnContactFn)(void* instance, LAB_EntityId other);
```

In C++ you override names on `LABNative::Behaviour` instead, and never touch these pointers:

| Callback | Called |
|---|---|
| `OnCreate()` | Once, right after construction, after any authored field values are applied |
| `OnFixedUpdate(float dt)` | Once before each physics step the solver takes, ahead of that frame's `OnUpdate` |
| `OnUpdate(float dt)` | Every frame while playing, after the physics step |
| `OnCollisionBegin(other)` | A solid contact started |
| `OnCollisionEnd(other)` | A solid contact ended |
| `OnOverlapBegin(other)` | Something entered a trigger |
| `OnOverlapEnd(other)` | Something left a trigger |
| `OnDestroy()` | Once, when the entity is destroyed or play stops, before `Destroy` frees the instance |

`other` is the entity on the far side of the contact and is never 0, since a contact always
names two real bodies. `OnDestroy` is never called twice for one instance, and never followed
by another `OnUpdate`.

### The two clocks

`OnUpdate` is the frame. It runs once per rendered frame with whatever that frame took, which
is a variable number: 16 ms one frame, 33 ms the next.

`OnFixedUpdate` is the physics step. It runs **once before each step** the solver is about to
take, with the fixed timestep, and the callback receives that step rather than the frame's
duration. A frame that takes three steps calls it three times, each call ahead of its own step,
so a force, an impulse or a velocity applied there is exactly what that step integrates, and
reading a body's position through `api->Physics` at the next call shows the result of the step
in between. `LABNative::FixedDeltaTime()` reports the same value outside a callback.

Until ABI minor 13 the engine ran the hook after the solver instead, once per step but all of
them at the end of the frame: a hitch frame called it N times against the same finished world,
and what a call applied belonged to the next frame's first step. The signature did not change, so
a module built earlier keeps loading and now runs ahead of the step. The hook still runs outside
the solver's own update, on the main thread, so every physics call is legal from it.

A scene with no rigid bodies takes no steps at all, so `OnFixedUpdate` never runs there. That
is the literal contract rather than a quirk: the hook is a physics step, not an
independent fixed-rate tick.

Move anything that has to agree with a rigid body in `OnFixedUpdate`. A force applied in
`OnUpdate` is applied on a different clock from the one the body integrates on, which shows up
as movement that is subtly wrong rather than obviously broken.

Contact callbacks (`OnCollisionBegin` and the rest) are not part of this: they are delivered
once per frame after all of that frame's steps, with the velocities the solver has already
changed, so a handler that wants the pre-impact velocity keeps a short history of its own.

## The instance and its entity

The engine constructs an instance with `new T()` and assigns the entity id immediately
afterwards. A constructor therefore runs **before** the entity id exists, so anything that
needs the entity belongs in `OnCreate`.

The instance pointer is opaque to the engine, and the module owns its memory: `Create`
allocates, `Destroy` frees. The engine never allocates or frees a module's instance, which is
the boundary a shared library's separate heap always needs.

## `LABNative::Behaviour`

The base class a module's classes derive from.

| Member | |
|---|---|
| `LAB_EntityId GetEntity() const` | This instance's entity |
| `LAB_EntityId m_Entity` | The same value. Set by the engine's registrar, not by your constructor |
| `GetPosition` / `SetPosition` | [Transform](../entity/02-transform.md) |
| `GetRotation` / `SetRotation` | [Transform](../entity/02-transform.md) |
| `GetScale` / `SetScale` | [Transform](../entity/02-transform.md) |
| `GetWorldPosition` | [Transform](../entity/02-transform.md) |
| `std::string GetName() const` | [Entity queries](../entity/01-entity-queries.md) |
| `void Destroy() const` | [Entity queries](../entity/01-entity-queries.md) |
| `AddForce` / `AddImpulse` | [Physics](../physics/01-physics.md) |
| `GetVelocity` / `SetVelocity` | [Physics](../physics/01-physics.md) |
| `GetTransform` / `SetTransform` | [Components](../entity/04-components.md) |
| `GetCamera` / `SetCamera` | [Components](../entity/04-components.md) |
| `GetRigidBody` / `SetRigidBody` | [Components](../entity/04-components.md) |
| `GetDirectionalLight` / `GetPointLight` / `GetSpotLight` | [Components](../entity/04-components.md), each takes an entity id |
| and their `Set` forms | |

The light getters take an entity id rather than using the behaviour's own, because a light is
usually on another entity: reaching the sun from a behaviour on a player is the ordinary case.

`Destroy()` is safe to call from inside `OnUpdate`, a contact callback or an overlap callback.
The engine defers the actual free until the callback that asked for it returns, so `this`
stays valid for the rest of the call it was made from.

## Threading

A behaviour runs on the main thread, the same thread as the scene, the renderer and Lua. It
must not block. Spawning, destroying an entity and timing a span are all safe from inside a
callback; waiting on a file, a socket or another thread is not, because the frame cannot
proceed until the callback returns.

Engine state is not thread-safe, and a module has no supported way to reach it from a thread
of its own. If you need background work, compute into your own memory on your own thread and
publish the result in a callback.

## Failures

| What happens | What the engine does |
|---|---|
| `LAB_ModuleInit` is missing | Refuses to load the library, and logs why |
| The major version differs | Refuses, and logs both versions |
| The minor is newer than the engine's | Refuses, and logs both versions |
| A behavioural class name is unknown | Reports it, skips that instance, keeps the others |
| A callback returns normally | Nothing |
| A callback throws | Catches it, logs it with the behaviour's name, disables that instance |

A disabled instance is not retried. It still receives `OnDestroy` when its entity goes away,
and `native_disabled_count()` reports how many are off. Its siblings on the same entity keep
running, because the mark is per instance.

A hard fault is different: a callback is native code with no engine frame of its own, so a
segfault cannot be contained in-process. The crash report names the behaviour and the phase
that were running.

A module's callback must not catch its own exceptions. The engine is the single catch on that
path, and it disables the behaviour; a catch inside the module swallows the exception first,
and the engine never learns the behaviour threw. A module that marks its own callbacks
`noexcept` is choosing `std::terminate` for itself, and the engine cannot help.
