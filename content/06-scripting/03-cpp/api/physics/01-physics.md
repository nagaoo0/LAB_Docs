---
title: "Physics"
---

Rays, forces, impulses, velocity and spin.

Everything here needs a physics world, which exists only while the scene is playing. Outside
play mode a ray finds nothing and a force does nothing, rather than failing.

## `Raycast`

```c
typedef struct LAB_RaycastHit
{
	int Hit;               // bool as int, for C ABI portability
	LAB_Vec3 Position;
	LAB_Vec3 Normal;
	LAB_EntityId Entity;   // 0 when Hit is false
} LAB_RaycastHit;

LAB_RaycastHit (*Raycast)(LAB_Vec3 origin, LAB_Vec3 direction, float maxDistance);
```

```cpp
LAB_RaycastHit LABNative::Raycast(LAB_Vec3 origin, LAB_Vec3 direction, float maxDistance);
```

Casts a ray and answers the first thing it hits. This is the flat table's cast and it has no
filters; [`RaycastEx`](#raycastex) further down is the same cast with an ignore argument, skip
flags and a distance, and is what a module that probes the ground every step wants.

```cpp
const LAB_RaycastHit hit = LABNative::Raycast(origin, LAB_Vec3{ 0.0f, 0.0f, -1.0f }, 100.0f);
if (hit.Hit)
	LABNative::LogInfo("hit the ground");
```

| Field | |
|---|---|
| `Hit` | Non-zero when something was hit. Everything else is zero when it is zero |
| `Position` | The world-space point of impact |
| `Normal` | The surface normal at that point |
| `Entity` | The entity that was hit, or 0 |

**The direction does not have to be normalised.** The engine divides the direction by its length
and scales the ray by `maxDistance`, so `(0,0,-1)` and `(0,0,-9.81)` cast the same ray with the
same reach. A zero-length direction, or a `maxDistance` of zero or less, answers a miss.

### It will hit the caster

The flat `Raycast` takes no "ignore this entity" argument (`RaycastEx` does), and the Lua
facade's optional fourth argument has no counterpart here. A ray fired from an entity's own
centre therefore hits its own collider at distance zero, and answers *that* rather than the
ground below.

Three ways around it, the first being `RaycastEx` with `ignore` set to the caster:

```cpp
// Start the ray outside the thing doing the casting.
const LAB_Vec3 origin = GetWorldPosition() + LAB_Vec3{ 0.0f, 0.0f, 1.2f };
const LAB_RaycastHit ground = LABNative::Raycast(origin, LAB_Vec3{ 0.0f, 0.0f, -1.0f }, 5.0f);
```

- Offset the origin clear of your own collider, as above.
- Or check `hit.Entity != GetEntity()` and cast again past it. Note this is worse than it looks:
  the engine filters the ignored body *during* the cast rather than after, precisely because
  discarding a result afterwards throws away everything behind it.

### No distance in the result

The Lua facade's hit table carries a `distance`; `LAB_RaycastHit` does not, and cannot gain one
without a major version bump. `RaycastEx` returns it. With the flat cast, compute it from the
origin, the way a module has to do its own vector maths:

```cpp
const LAB_Vec3 toHit{ hit.Position.x - origin.x, hit.Position.y - origin.y, hit.Position.z - origin.z };
const float distance = std::sqrt(toHit.x * toHit.x + toHit.y * toHit.y + toHit.z * toHit.z);
```

## `AddForce` and `AddImpulse`

```c
void (*AddForce)(LAB_EntityId entity, LAB_Vec3 force);
void (*AddImpulse)(LAB_EntityId entity, LAB_Vec3 impulse);
```

```cpp
void LABNative::AddForce(LAB_EntityId entity, LAB_Vec3 force);
void LABNative::AddImpulse(LAB_EntityId entity, LAB_Vec3 impulse);

void LABNative::Behaviour::AddForce(LAB_Vec3 force) const;
void LABNative::Behaviour::AddImpulse(LAB_Vec3 impulse) const;
```

Both are no-ops on an entity with no rigid body, or one that is not dynamic. That is deliberate:
a behaviour does not necessarily know what it is attached to, and an error every frame would be
worse than nothing happening.

| | `AddForce` | `AddImpulse` |
|---|---|---|
| Applied | Continuously over the step | Once, immediately |
| Unit | Newtons: a mass-scaled acceleration | Newton-seconds: a direct velocity change |
| Effect of mass | Heavier needs more force for the same motion | Independent of mass for the velocity |
| Use for | Thrusters, wind, a conveyor | Jumps, impacts, knockback |

**Call them from `OnFixedUpdate`.** Since ABI minor 13 the hook runs once *before* each physics
step, so a force or impulse applied there is what that very step integrates, and a frame that
takes three steps calls the hook three times, each ahead of its own step. Applying one from
`OnUpdate` applies it on the frame's clock rather than the physics clock, which shows up as
movement that is subtly wrong rather than obviously broken: the same push produces different
results at different frame rates. (Before minor 13 the hook ran after all of a frame's steps,
so what it applied belonged to the next frame's first step.)

```cpp
void OnFixedUpdate(float fixedDeltaTime) override
{
	// Held thrust, scaled by the step it applies over.
	if (LABNative::ActionDown("Thrust"))
		AddForce(LAB_Vec3{ 0.0f, 200.0f, 0.0f });

	// A jump is one instantaneous change, not a push.
	if (LABNative::ActionPressed("Jump"))
		AddImpulse(LAB_Vec3{ 0.0f, 0.0f, 6.0f });
}
```

## `GetVelocity` / `SetVelocity`

```c
LAB_Vec3 (*GetVelocity)(LAB_EntityId entity);
void (*SetVelocity)(LAB_EntityId entity, LAB_Vec3 velocity);
```

```cpp
LAB_Vec3 LABNative::GetVelocity(LAB_EntityId entity);
void LABNative::SetVelocity(LAB_EntityId entity, LAB_Vec3 velocity);

LAB_Vec3 LABNative::Behaviour::GetVelocity() const;
void LABNative::Behaviour::SetVelocity(LAB_Vec3 velocity) const;
```

Linear velocity in metres per second, in world space. A zero vector for an entity that does not
resolve or has no dynamic body.

```cpp
// Clamp horizontal speed without touching the falling speed.
LAB_Vec3 velocity = GetVelocity();
const float speed = std::sqrt(velocity.x * velocity.x + velocity.y * velocity.y);
if (speed > m_MaxSpeed)
{
	const float scale = m_MaxSpeed / speed;
	SetVelocity(LAB_Vec3{ velocity.x * scale, velocity.y * scale, velocity.z });
}
```

`SetVelocity` is a direct write rather than a push, so it ignores mass and cancels whatever
motion the body had. It is the right call for a hard clamp, a respawn, or a dash; `AddImpulse`
is the right one for anything that should compose with the body's current motion.

**Note that a body can exceed a speed you clamped horizontally.** Gravity is not something
`SetVelocity` bounds, so a ball rolling downhill is legitimately faster than the cap its input
respects.

## The physics sub-table (ABI minor 13)

`api->Physics` carries the calls a module steering a rigid body from `OnFixedUpdate` needs beyond
the flat table. Each is a no-op that answers 0, false or a zero vector for an entity with no body
and outside play mode, and on an engine older than minor 13 the wrappers do the same.

```c
typedef struct LAB_PhysicsAPI
{
	uint32_t StructSize;
	int (*GetBodyPosition)(LAB_EntityId entity, LAB_Vec3* out);
	LAB_Vec3 (*GetAngularVelocity)(LAB_EntityId entity);
	void (*SetAngularVelocity)(LAB_EntityId entity, LAB_Vec3 velocity);
	void (*WakeUp)(LAB_EntityId entity);
	int (*RaycastEx)(LAB_Vec3 origin, LAB_Vec3 direction, float maxDistance, LAB_EntityId ignore, uint32_t flags, LAB_RaycastHitEx* out);
} LAB_PhysicsAPI;
```

```cpp
bool LABNative::Physics::GetBodyPosition(LAB_EntityId entity, LAB_Vec3& out);
LAB_Vec3 LABNative::Physics::GetAngularVelocity(LAB_EntityId entity);
void LABNative::Physics::SetAngularVelocity(LAB_EntityId entity, LAB_Vec3 velocity);
void LABNative::Physics::WakeUp(LAB_EntityId entity);
LABNative::Physics::HitEx LABNative::Physics::RaycastEx(LAB_Vec3 origin, LAB_Vec3 direction,
	float maxDistance, LAB_EntityId ignore = 0, uint32_t flags = LABNative::Physics::HitsAll);

bool LABNative::Behaviour::GetBodyPosition(LAB_Vec3& out) const;
LAB_Vec3 LABNative::Behaviour::GetAngularVelocity() const;
void LABNative::Behaviour::SetAngularVelocity(LAB_Vec3 velocity) const;
```

### `GetBodyPosition`

The body's position **as the solver holds it right now**: after the last step, with no render
blending. `GetPosition` and `GetWorldPosition` read the entity's transform, which the engine
writes back blended between the last two steps for rendering and which can lag the solver by up
to one step, so a fixed-step behaviour that needs the body itself (a tether pulling two balls
together, a ground probe) reads this. False for an entity with no body, leaving `out` alone.

### `GetAngularVelocity` / `SetAngularVelocity`

Radians per second about each world axis. `SetAngularVelocity` **clamps the length to 46 rad/s**,
under Jolt's own cap of 47.12 rad/s (a quarter turn per fixed step at 60 Hz), above which a debug
build asserts. The direction is kept, and a component that is not a finite number reads as no
spin. The Lua call `entity:set_angular_velocity` goes through the same clamp.

It is the call that sets a rolling ball's spin. An impulse through the centre of a ball that is
already rolling changes its speed but not its spin, so the ball slips; setting the spin to
`speed / radius` about the right axis restores rolling without slip:

```cpp
// A ball of radius r rolling on level ground (Z is up) with horizontal velocity (vx, vy).
SetVelocity(LAB_Vec3{ vx, vy, vz });
SetAngularVelocity(LAB_Vec3{ -vy / r, vx / r, 0.0f });
```

A static body ignores the call. The spin of a kinematic body is how a flipper or a spinner is
driven.

### `WakeUp`

Wakes a body the solver has put to sleep. `SetVelocity`, `AddImpulse` and `SetAngularVelocity`
already do, so this is for a body that has to be awake for another reason, such as one whose
transform another system moved.

### `RaycastEx`

```c
typedef struct LAB_RaycastHitEx
{
	int Hit;
	LAB_Vec3 Position;
	LAB_Vec3 Normal;
	LAB_EntityId Entity;   // 0 when Hit is false
	float Distance;        // along the ray from its origin, 0 on a miss
	uint32_t Reserved;     // always 0
} LAB_RaycastHitEx;
```

The closest hit along the ray, written to the struct you pass; the wrapper returns it by value.
On a miss every field is zero, and the call answers the same as `Hit`.

| Argument | |
|---|---|
| `origin`, `direction`, `maxDistance` | As for `Raycast`: the direction need not be normalised |
| `ignore` | One entity the ray passes through, 0 for none. Usually the caster, so a ray from its own centre finds the ground and not itself |
| `flags` | A mask of the values below, 0 to hit everything |

| Flag | Skips |
|---|---|
| `SkipSensors` (`LAB_RAYCAST_SKIP_SENSORS`) | Trigger volumes. A ground probe that finds a trigger zone instead of the floor under it is the failure this exists for |
| `SkipDynamic` (`LAB_RAYCAST_SKIP_DYNAMIC`) | Dynamic bodies. **Kinematic and static bodies are still hit**, so a moving platform carrying the caster is found like any floor |

Skipping happens during the cast, like `ignore`, so a skipped body never hides what is behind
it. The engine cannot do this with a collision layer, because it puts dynamic and kinematic
bodies on the same moving layer; the flags are tested against each body's own motion type.

```cpp
// A grounded test for a rolling ball: the floor under it, ignoring itself, triggers and other balls.
const auto hit = LABNative::Physics::RaycastEx(
	GetWorldPosition(), LAB_Vec3{ 0.0f, 0.0f, -1.0f }, m_Radius + 0.12f, GetEntity(),
	LABNative::Physics::SkipSensors | LABNative::Physics::SkipDynamic);
const bool grounded = hit.Hit && hit.Normal.z >= 0.55f;
```

## What is not here

There is no torque or angular impulse, no gravity getter or setter, and no shape query on the
ABI. A module wanting any of those has three options: derive the effect from what is here, drive
the entity's transform directly (which [notifies the scene](../entity/02-transform.md), so a
physics-backed entity follows), or ask for the call in the ABI.

A windmill, for instance, is a kinematic body whose transform a behaviour rotates, or whose
`SetAngularVelocity` it sets, rather than a body the solver spins.
