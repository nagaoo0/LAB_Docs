---
title: "Splines"
---

Queries against a `SplineComponent`, in world space.

A spline is a curve an entity carries: a list of points, whether it closes, and a width profile.
The same content a `SplineMeshComponent` gets extruded from. These calls read it, which is what a
behaviour needs to run something along a path, aim it, or ask how far along it is.

Everything here is in **world space**, distances in **metres** along the curve, and frames as
world vectors. That is exactly what a Lua script gets from the same calls, so a path authored
once behaves the same whichever language follows it.

## `HasSpline`

```c
int (*HasSpline)(LAB_EntityId entity);
```

```cpp
bool LABNative::Spline::HasSpline(LAB_EntityId entity);
```

Whether the entity has a spline at all. Every other call in this area answers a zero or a null
frame for an entity that does not, so this is how you tell "this entity has no spline" from "the
spline is somewhere unexpected".

## `GetLength`

```c
float (*GetLength)(LAB_EntityId entity);
```

```cpp
float LABNative::Spline::GetLength(LAB_EntityId entity);
```

The curve's total length in metres.

The length is **arc-length accurate**, not an estimate from the control points: the spline keeps
a table of the real curve so a sample at 30 metres is 30 metres along it, whatever the points do
in between.

## `IsClosed`

```c
int (*IsClosed)(LAB_EntityId entity);
```

```cpp
bool LABNative::Spline::IsClosed(LAB_EntityId entity);
```

Whether the curve joins its last point back to its first. A closed spline's length includes the
closing span, and sampling past the last point wraps rather than clamping.

## `Sample`

```c
void (*Sample)(LAB_EntityId entity, float distance,
	LAB_Vec3* outPosition, LAB_Vec3* outForward, LAB_Vec3* outRight, LAB_Vec3* outUp, float* outWidth);
```

```cpp
void LABNative::Spline::Sample(LAB_EntityId entity, float distance,
	LAB_Vec3* outPosition, LAB_Vec3* outForward, LAB_Vec3* outRight, LAB_Vec3* outUp, float* outWidth);
```

Position, forward, right, up and width at a distance along the curve.

**Every out pointer is optional, and a null one is simply never written.** So a caller that only
wants the position passes one pointer and four nulls:

```cpp
LAB_Vec3 position{};
LABNative::Spline::Sample(rail, m_Distance, &position, nullptr, nullptr, nullptr, nullptr);
SetPosition(position);
```

The frame is an orthonormal set: `Forward` is the direction of travel, `Right` and `Up` are the
other two axes, all in world space. "Forward" is the curve's own direction, not the entity's, so
a thing following a spline gets the right heading without computing one.

`Width` is the per-point width profile the spline carries, which is what a tube or a wall gets
extruded from. A spline with a constant width answers that constant; one that varies answers the
interpolated value at this point.

### Following a curve

The idiomatic shape: keep a distance, advance it by speed times the timestep, and sample.

```cpp
void OnUpdate(float deltaTime) override
{
	const float length = LABNative::Spline::GetLength(m_Rail);
	if (length <= 0.0f)
		return;

	m_Distance += Speed * deltaTime;
	if (LABNative::Spline::IsClosed(m_Rail))
		m_Distance = std::fmod(m_Distance, length);
	else
		m_Distance = std::min(m_Distance, length);

	LAB_Vec3 position{};
	LAB_Vec3 forward{};
	LABNative::Spline::Sample(m_Rail, m_Distance, &position, &forward, nullptr, nullptr, nullptr);
	SetPosition(position);
	// `forward` is the world direction of travel: turn it into a rotation if the thing
	// following the curve should face along it.
}
```

Advancing by distance rather than by a per-segment index is what keeps the speed constant. A
curve with a tight corner has more control points in the same length, so stepping by point would
speed up in the corners and slow down on the straights.

## `Closest`

```c
void (*Closest)(LAB_EntityId entity, LAB_Vec3 point, float hint, float window,
	float* outDistance, float* outLateral, float* outVertical);
```

```cpp
void LABNative::Spline::Closest(LAB_EntityId entity, LAB_Vec3 point, float hint, float window,
	float* outDistance, float* outLateral, float* outVertical);
```

The distance along the curve nearest `point`, then that frame's lateral (right) and vertical (up)
offset from the point. All three out pointers are optional.

**A hint or a window below zero searches the whole curve**, which is what a caller with no
previous answer passes:

```cpp
float along = 0.0f;
LABNative::Spline::Closest(m_Rail, GetWorldPosition(), -1.0f, -1.0f, &along, nullptr, nullptr);
```

A hint and a window together are the cheap form: pass the last answer as the hint and a window of
a few metres, and the search only looks near it. That matters when this is called every frame for
several entities, because the exact search walks the whole curve.

The lateral and vertical outputs are the offset *in the curve's own frame*, not world axes. That
makes them the right values for a question like "how far off the centre line is this thing", and
the wrong ones for "how far east of it is".

## `GetPointCount`

```c
int (*GetPointCount)(LAB_EntityId entity);
```

```cpp
int LABNative::Spline::GetPointCount(LAB_EntityId entity);
```

How many control points the spline was authored with.

This is a structural count, not a measure of the curve's detail. The sampled curve between two
control points is a smooth interpolation, so a spline with four points is still a smooth loop.

## Scaling

**A scaled entity is assumed to be scaled uniformly**, and its local `+X` axis length is taken as
the scale. So a spline on an entity scaled `(2,2,2)` is twice as long and its frames are twice as
wide; one scaled `(2,1,1)` is treated as scale 2, because a non-uniform scale would need a
different curve, not a stretched one.

## What is not here

There is no way to create, edit or re-point a spline from a module, and no way to read the
control points themselves: the curve is queried, never authored, at run time. Splines are built
in the editor's spline tool and saved with the scene.

There is also no query for a spline's own entity from a mesh generated from it, and no per-point
roll read. A per-point roll exists in the data and shapes the frames a `SplineMeshComponent`
extrudes, so it is already baked into `Right` and `Up` where it matters for following the curve.
