---
title: "Animation and particles"
---

Animator parameters and particle emitters.

## `SetAnimParameter` / `GetAnimParameter`

```c
void (*SetAnimParameter)(LAB_EntityId entity, const char* name, float value);
float (*GetAnimParameter)(LAB_EntityId entity, const char* name);
```

```cpp
void LABNative::SetAnimParameter(LAB_EntityId entity, const char* name, float value);
float LABNative::GetAnimParameter(LAB_EntityId entity, const char* name);
```

Sets and reads a named parameter on an entity's animator.

These are the values an animation graph's transitions and blend spaces read. A `.Lanimgraph`
declares its parameters (a `Speed` float, a `Grounded` bool, a `Jump` trigger), and a behaviour
driving one is doing exactly what a Lua script does with the same call:

```cpp
const LAB_Vec3 v = GetVelocity();
const float speed = std::sqrt(v.x * v.x + v.y * v.y);
LABNative::SetAnimParameter(GetEntity(), "Speed", speed);
LABNative::SetAnimParameter(GetEntity(), "Grounded", m_Grounded ? 1.0f : 0.0f);
```

Both calls are no-ops, and answer 0, on an entity with **no `AnimatorComponent`**. That is the
same tolerance a mistyped parameter name already gets inside the animator: a graph that does not
have a parameter called `Speed` reads 0 and ignores a write.

So a wrong name and a wrong entity look identical, and both are quiet. If an animation is not
changing, `GetAnimParameter` reading back what you just set is the first check: if it does not,
the name is not one the graph has.

Parameters are **floats**. A trigger or a bool is 1.0 or 0.0, which is how the graph's conditions
read them.

Reading is genuinely useful rather than only diagnostic: a graph can drive a parameter from its
own logic, and a behaviour wanting to know what the animation decided reads it back.

## `EmitParticles` / `GetParticleCount`

```c
void (*EmitParticles)(LAB_EntityId entity, int count);
int (*GetParticleCount)(LAB_EntityId entity);
```

```cpp
void LABNative::EmitParticles(LAB_EntityId entity, int count);
int LABNative::GetParticleCount(LAB_EntityId entity);
```

Emits `count` particles from the entity's `ParticleEmitterComponent`, and answers how many
particles that emitter is holding.

```cpp
// A burst on impact, on top of whatever the emitter's own rate is producing.
LABNative::EmitParticles(GetEntity(), 24);
```

Both are no-ops, and the count 0, for an entity with no emitter.

`EmitParticles` is a **burst**: it adds `count` particles now, on top of the continuous
`EmissionRate` the component is already producing. An emitter with a rate of zero and no bursts
is silent; one with a rate of zero that a behaviour pokes is a purely event-driven effect, which
is the usual shape for impacts, dust and celebrations.

`GetParticleCount` is the live count, so it rises with a burst and falls as particles expire. It
is the honest way to ask "is this emitter still busy?" without assuming anything about the
emitter's lifetime settings.

## Which clock

Both belong in `OnUpdate`, not `OnFixedUpdate`.

Particles are simulated by the frame's own update, not by the physics step, and a scene with no
rigid bodies takes no physics steps at all. An animator parameter set from `OnFixedUpdate` would
also arrive several times per frame on a slow frame and then not at all on a still one, which is
the opposite of what an animation wants.

```cpp
void OnUpdate(float deltaTime) override
{
	const LAB_Vec3 v = GetVelocity();
	LABNative::SetAnimParameter(GetEntity(), "Speed", std::sqrt(v.x * v.x + v.y * v.y));

	m_Fuse -= deltaTime;
	if (m_Fuse <= 0.0f)
		LABNative::EmitParticles(GetEntity(), 40);
}
```

## What is not here

There is no way to read an animation's current state, time or normalized phase from a module, no
way to play a specific clip, and no access to a skeleton or its bones. The animator is driven
through its parameters and nothing else.

For particles there is no per-particle access and no way to set the emitter's fields: an emitter
is authored in the component, and a behaviour's two tools are "emit a burst" and "how many are
alive".

Both are the same split the Lua facade has, so a module and a script reach the animator and the
emitter by exactly the same route.
