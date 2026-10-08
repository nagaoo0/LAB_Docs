---
title: "Components"
---

Reading and writing the components a behaviour most often needs, without the module having to
know anything about `entt`, `glm` or the engine's own headers.

## `GetComponent` / `SetComponent` / `ComponentSize`

```c
int (*GetComponent)(LAB_EntityId entity, int type, void* out, uint32_t size);
int (*SetComponent)(LAB_EntityId entity, int type, const void* in, uint32_t size);
uint32_t (*ComponentSize)(int type);
```

```cpp
template<typename T> bool LABNative::GetComponent(LAB_EntityId entity, LAB_ComponentType type, T& out);
template<typename T> bool LABNative::SetComponent(LAB_EntityId entity, LAB_ComponentType type, const T& value);
uint32_t LABNative::ComponentSize(LAB_ComponentType type);
```

Both return 1 on success and 0 otherwise, including when the entity simply does not have that
component, which is not an error worth logging.

```cpp
LAB_TransformComponent transform{};
if (GetComponent(entity, LAB_COMPONENT_TRANSFORM, transform))
	transform.Position.y = 1234.0f;
```

## The size contract

`size` is **your own `sizeof`**, and the engine checks it against its own. A module built before
a field was added passes a smaller struct and is **refused** rather than partly filled, because
half a component is worse than none: the tail the engine did not write keeps whatever your
struct's initializer left there, and it is indistinguishable from a real value.

`ComponentSize(type)` is what a module compares against to notice that mismatch for itself:

```cpp
if (LABNative::ComponentSize(LAB_COMPONENT_RIGIDBODY) != sizeof(LAB_RigidBodyComponent))
	LABNative::LogWarn("the module's rigid body struct is from an older header");
```

### A view, not a layout clone

The mirror structs name the fields a module can see. The engine copies them **field by field**,
so this is a view rather than a memory layout the two sides have to agree on exactly. Adding a
field to the engine's own component does not move anything a module reads, which is the point.

The consequence: a field of the engine's component that the mirror struct does not name is
simply not visible, and a write through `SetComponent` does not disturb it. Copying a mirror
struct back is a write of the named fields only.

## The component types

```c
typedef enum LAB_ComponentType
{
	LAB_COMPONENT_TRANSFORM = 0,
	LAB_COMPONENT_CAMERA = 1,
	LAB_COMPONENT_RIGIDBODY = 2,
	LAB_COMPONENT_DIRECTIONALLIGHT = 3,
	LAB_COMPONENT_POINTLIGHT = 4,
	LAB_COMPONENT_SPOTLIGHT = 5,
	LAB_COMPONENT_COUNT
} LAB_ComponentType;
```

An enum in the ABI header rather than one of the engine's own type indices, because these numbers
are stable across builds and compilers in a way a C++ type id is not.

**Six components cross.** Everything else is reached through its own API, or is simply not
available to a module yet. There is no mirror for a script, an audio source, a HUD element, a
particle emitter, a collider, a character controller, a nav agent or anything under a skeleton:
those are reached through their own calls, which is why the sub-tables exist.

## `LAB_TransformComponent`

```c
typedef struct LAB_TransformComponent
{
	LAB_Vec3 Position;
	LAB_Vec3 Rotation;   // radians
	LAB_Vec3 Scale;
} LAB_TransformComponent;
```

| Field | Unit |
|---|---|
| `Position` | Local, metres |
| `Rotation` | **Radians.** See the note below |
| `Scale` | Local, unitless, `(1,1,1)` is identity |

**Rotation here is radians, while `GetRotation`/`SetRotation` are degrees.** That is not an
inconsistency to work around, it is the difference between the engine's own representation and
the convenience pair. Pick one per code path and stay in it.

```cpp
bool LABNative::Behaviour::GetTransform(LAB_TransformComponent& out) const;
bool LABNative::Behaviour::SetTransform(const LAB_TransformComponent& value);
```

## `LAB_CameraComponent`

```c
typedef struct LAB_CameraComponent
{
	float FOV;          // degrees
	float NearClip;
	float FarClip;
	float Exposure;
	int Primary;
} LAB_CameraComponent;
```

`Primary` is an `int` because this header is plain C; treat it as a bool. A primary camera is the
one play mode switches the viewport to.

```cpp
bool LABNative::Behaviour::GetCamera(LAB_CameraComponent& out) const;
bool LABNative::Behaviour::SetCamera(const LAB_CameraComponent& value);
```

A behaviour changing its own camera's FOV every frame is a valid way to do a zoom, and writing
the field back through `SetCamera` is enough; the renderer reads it per frame.

## `LAB_RigidBodyComponent`

```c
typedef struct LAB_RigidBodyComponent
{
	int Type;              // 0 static, 1 dynamic, 2 kinematic
	float Mass;            // kilograms
	int UseGravity;
	float Restitution;
	float Friction;
	float LinearDamping;
	float AngularDamping;
	float GravityScale;
} LAB_RigidBodyComponent;
```

| Field | Meaning |
|---|---|
| `Type` | 0 static, 1 dynamic, 2 kinematic, matching the engine's own body types |
| `Mass` | Kilograms. Only meaningful for a dynamic body |
| `UseGravity` | Whether the solver applies gravity |
| `Restitution` | Bounciness, 0 to 1 |
| `Friction` | Surface friction. Zero on **either** of two touching bodies makes them slide |
| `LinearDamping` | Velocity damping per second |
| `AngularDamping` | Spin damping per second |
| `GravityScale` | A multiplier on gravity, for floatier or heavier-feeling bodies |

Setting a friction of zero on a ball *or* on the floor it rolls on makes it slide rather than
roll. Rolling comes from friction at the contact point, so it takes both sides having some.

```cpp
bool LABNative::Behaviour::GetRigidBody(LAB_RigidBodyComponent& out) const;
bool LABNative::Behaviour::SetRigidBody(const LAB_RigidBodyComponent& value);
```

The mass is settled when the physics body is created, so writing `Mass` on a live body changes
the component and not the simulation. Forces and velocity are the way to change a moving body's
behaviour, through [Physics](../physics/01-physics.md).

## The three lights

Separate types rather than one with a kind field, because that is what they are in the engine: a
behaviour animating a spot's cone has no business carrying a radius it will never use.

```c
typedef struct LAB_DirectionalLightComponent
{
	LAB_Vec3 Color;
	float Intensity;
	int PhysicalUnits;      // when set, Intensity is ignored and Lux is the figure that matters
	float Lux;
	float IndirectIntensity;
	int CastShadows;
	float ShadowDistance;
	int ShadowCascades;
	float ShadowBias;
	float ShadowStrength;
} LAB_DirectionalLightComponent;
```

```c
typedef struct LAB_PointLightComponent
{
	LAB_Vec3 Color;
	float Intensity;
	float Radius;
	float IndirectIntensity;
	int CastShadows;
	float ShadowBias;
	float ShadowStrength;
} LAB_PointLightComponent;
```

```c
typedef struct LAB_SpotLightComponent
{
	LAB_Vec3 Color;
	float Intensity;
	float InnerConeAngle;   // radians
	float OuterConeAngle;   // radians
	float IndirectIntensity;
	int CastShadows;
	float ShadowRange;
	float ShadowBias;
	float ShadowStrength;
} LAB_SpotLightComponent;
```

**A directional light has no direction field**, and that is deliberate: it shines along its
entity's world +X axis, so aiming one is a transform write. A `Direction` vector beside a
rotation would be two sources of truth for one thing, free to disagree. See
[Transform](02-transform.md#which-way-does-an-entity-face).

Spot cone angles are in **radians**, matching the engine's own fields, unlike the degrees the
transform pair uses.

```cpp
bool LABNative::Behaviour::GetDirectionalLight(LAB_EntityId entity, LAB_DirectionalLightComponent& out) const;
bool LABNative::Behaviour::SetDirectionalLight(LAB_EntityId entity, const LAB_DirectionalLightComponent& value);
bool LABNative::Behaviour::GetPointLight(LAB_EntityId entity, LAB_PointLightComponent& out) const;
bool LABNative::Behaviour::SetPointLight(LAB_EntityId entity, const LAB_PointLightComponent& value);
bool LABNative::Behaviour::GetSpotLight(LAB_EntityId entity, LAB_SpotLightComponent& out) const;
bool LABNative::Behaviour::SetSpotLight(LAB_EntityId entity, const LAB_SpotLightComponent& value);
```

These take an entity id rather than assuming the behaviour's own, because a light is usually on
another entity: reaching the sun from a behaviour on a player is the ordinary case.

## A complete example

Dim the sun over time, and read back a body's state.

```cpp
void OnUpdate(float deltaTime) override
{
	const LAB_EntityId sun = LABNative::FindEntityByName("Sun");
	LAB_DirectionalLightComponent light{};
	if (LABNative::GetComponent(sun, LAB_COMPONENT_DIRECTIONALLIGHT, light))
	{
		light.Intensity = std::max(0.0f, light.Intensity - 0.1f * deltaTime);
		LABNative::SetComponent(sun, LAB_COMPONENT_DIRECTIONALLIGHT, light);
	}

	LAB_RigidBodyComponent body{};
	if (GetRigidBody(body) && body.Type == 1)
	{
		// A dynamic body: its motion is the solver's business, so read it through
		// GetVelocity rather than by watching the transform.
	}
}
```
