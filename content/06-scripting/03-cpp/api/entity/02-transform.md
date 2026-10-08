---
title: "Transform"
---

Where an entity is, which way it faces, and how big it is.

## The units trap

Two spellings of the same data use different units, and getting them mixed up produces rotation
that is wrong by a factor of about 57 in one direction.

| Access | Unit |
|---|---|
| `GetRotation` / `SetRotation` | **Degrees** |
| `LAB_TransformComponent::Rotation` | **Radians** |

The standalone calls are degrees because that is what a Lua script and the inspector use, and
they convert on the way in and out. The component mirror struct is the engine's own
representation, so it is radians. The engine's own code does the conversion explicitly
(`t->Rotation * kRadToDeg` on a read, `degrees * kDegToRad` on a write), so both are exact and
neither is a rounding of the other.

If you are moving an entity every frame and it spins far faster than you asked, this is why.

## `GetPosition` / `SetPosition`

```c
LAB_Vec3 (*GetPosition)(LAB_EntityId entity);
void (*SetPosition)(LAB_EntityId entity, LAB_Vec3 position);
```

```cpp
LAB_Vec3 LABNative::GetPosition(LAB_EntityId entity);
void LABNative::SetPosition(LAB_EntityId entity, LAB_Vec3 position);

// on a behaviour, for its own entity:
LAB_Vec3 LABNative::Behaviour::GetPosition() const;
void LABNative::Behaviour::SetPosition(LAB_Vec3 position) const;
```

The entity's **local** position: relative to its parent when it has one, and to the world when
it does not. A zero vector for an entity that does not resolve.

**Writing a transform notifies the scene, which is what makes it stick.** A physics-backed
entity has its transform rewritten by the solver every step, so a position written from outside
the simulation would otherwise be discarded on the next step. The engine's own setter calls
`Scene::NotifyTransformChanged` for you, so a script or a behaviour writing a position on a
rigid body is not fighting the physics, and the body is moved to match.

That is not a discontinuity: it is how continuous scripted motion has always moved anything, the
same path `PhysicsWorld::Step` uses for the transforms it writes back.

## `GetRotation` / `SetRotation`

```c
LAB_Vec3 (*GetRotation)(LAB_EntityId entity);            // degrees
void (*SetRotation)(LAB_EntityId entity, LAB_Vec3 degrees);
```

```cpp
LAB_Vec3 LABNative::GetRotation(LAB_EntityId entity);
void LABNative::SetRotation(LAB_EntityId entity, LAB_Vec3 degrees);

LAB_Vec3 LABNative::Behaviour::GetRotation() const;
void LABNative::Behaviour::SetRotation(LAB_Vec3 degrees) const;
```

**In degrees.** Euler angles, X then Y then Z as the engine composes them.

Writing one notifies the scene, exactly as `SetPosition` does.

The rotation is the entity's **local** rotation, so a child's rotation is relative to its
parent's.

### Which way does an entity face?

There is no single answer, because different things in this engine use different axes, and
assuming the wrong one is a common way to spend an afternoon.

- **A directional light shines along its entity's world +X axis.** Rotation Y is how far below
  the horizon it points, Z is its heading, and X only rolls it about its own beam. `[0,0,0]` is
  a horizontal sun. Aiming one is a transform write, not a light write.
- **A generic entity's forward is `-local Z`.** At `Rotation = (0,0,0)` that is straight down
  (the engine is Z-up), and `Rotation.z` alone does nothing to it: a rotation about the axis a
  vector is already collinear with cannot turn it. Y is what tilts forward out of vertical.

If you are aiming something, check which convention it uses before trusting a formula.

## `GetScale` / `SetScale`

```c
LAB_Vec3 (*GetScale)(LAB_EntityId entity);
void (*SetScale)(LAB_EntityId entity, LAB_Vec3 scale);
```

```cpp
LAB_Vec3 LABNative::GetScale(LAB_EntityId entity);
void LABNative::SetScale(LAB_EntityId entity, LAB_Vec3 scale);

LAB_Vec3 LABNative::Behaviour::GetScale() const;    // defaults to (1,1,1)
void LABNative::Behaviour::SetScale(LAB_Vec3 scale) const;
```

The entity's local scale. `GetScale` answers `(1,1,1)` for an entity that does not resolve,
rather than zero, because one is the identity and zero is a degenerate object.

**Scale does not follow a live resize for a collider.** Jolt bakes the size into the shape when
the body is created and does not follow a change afterwards, so scaling an entity that already
has a collider moves the visual and leaves the collision where it was. The same is true of the
Lua facade. If you need a collider at a different size, build the entity that way.

## `GetWorldPosition`

```c
LAB_Vec3 (*GetWorldPosition)(LAB_EntityId entity);
```

```cpp
LAB_Vec3 LABNative::GetWorldPosition(LAB_EntityId entity);
LAB_Vec3 LABNative::Behaviour::GetWorldPosition() const;
```

The entity's position in world space, with every ancestor's transform applied. A zero vector for
an entity that does not resolve.

There is no world-position *setter*, because the local value is the one that is stored. To place
an entity at a world point, spawn it there or account for its parent's transform yourself.

```cpp
// The distance from this entity to a world-space point, parent or no parent.
const LAB_Vec3 here = GetWorldPosition();
const float distance = std::sqrt(
	(here.x - target.x) * (here.x - target.x) +
	(here.y - target.y) * (here.y - target.y) +
	(here.z - target.z) * (here.z - target.z));
```

## Reading a whole transform at once

The four calls above are the convenient ones. For reading or writing everything together,
including the radian form, use the component mirror:

```cpp
LAB_TransformComponent transform{};
if (GetTransform(transform))       // radian rotation, in the engine's own units
{
	transform.Position.y += 1.0f;
	SetTransform(transform);
}
```

See [Components](04-components.md).

## A complete example

A bobber: rise and fall about where it started, on the frame clock.

```cpp
void OnCreate() override
{
	m_Origin = GetPosition();      // local, so a parented entity bobs about its own origin
	m_Phase = m_Origin.x;
}

void OnUpdate(float deltaTime) override
{
	m_Elapsed += deltaTime;
	LAB_Vec3 position = m_Origin;
	position.z += 0.5f * std::sin(m_Elapsed * 2.0f + m_Phase);
	SetPosition(position);
}
```

Note that the behaviour remembers `m_Origin` rather than reading the position each frame off the
entity. Reading it back every frame and adding to it accumulates error, and it fights anything
else that moves the same entity.
