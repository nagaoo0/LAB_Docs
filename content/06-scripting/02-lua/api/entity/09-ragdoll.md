---
title: "Ragdoll"
---

Turning a skeletal entity into a physics ragdoll: which joints get bodies, how heavy and how
limited they are, and the live switch between animated and simulated.

see [Lua Scripting](../../index.md).

A ragdoll is physics rather than animation. Each named joint gets its own Jolt body, linked by
constraints that mirror the skeleton's own hierarchy, and while the ragdoll is **dynamic** the
solver owns those joints outright. See [Animation](../../../../04-gameplay/02-animation.md) and
[`docs/ANIMATION_GRAPH.md`](../../../../ANIMATION_GRAPH.md) for the system's own account.

The calls split in two, and mixing the halves up is the usual confusion:

- **Authoring calls** (`add_ragdoll_bone`, `set_ragdoll_radius`, `set_ragdoll_mass`,
  `set_ragdoll_bone_mass`, `set_ragdoll_bone_limits`, `set_ragdoll_physical_weight`) write this
  entity's Ragdoll component. They work in any mode, they create the component if it is missing,
  and they are read once, when the ragdoll is built. A change made after that build waits for the
  next one.
- **Live control** (`has_ragdoll`, `set_ragdoll_dynamic`, `is_ragdoll_dynamic`) talks to the
  built ragdoll, which exists only while playing.

**The build itself is automatic.** The first time the scene steps a skeletal entity that carries
a Ragdoll component, with a live physics world (during Play only), its bodies are created from
the pose the entity is in at that moment, not from the bind pose, so a character killed
mid-swing does not pop into its rest pose first. Nothing on this page builds it early.

A name that is not a joint on the entity's current skeleton is skipped at the build with a
warning in the log, and an entity whose names all miss is simply left without a ragdoll.

## `add_ragdoll_bone(jointName)`

Names a joint that gets its own body. Not every joint needs one: fingers are rarely worth it, and
a sparse selection still builds correctly, because the constraint tree is derived at build time
by walking each named joint's own parent chain up to the nearest *other* named joint, skipping
the unnamed ones in between. The order calls are made in does not matter.

Joints are named, not indexed, so the list survives a rig whose joint order changes. Creates the
Ragdoll component on an entity that has none.

## `set_ragdoll_radius(radius)`

The radius of every bone's capsule, in metres. One value for the whole ragdoll, defaulting to
`0.1`; values below `0.01` are clamped up at the build, so a zero does not produce degenerate
capsules.

Authoring only: read when the ragdoll builds, and it does not reach around a ragdoll that is
already falling.

## `set_ragdoll_mass(mass)`

The mass of each bone body, in kilograms. One value for the whole ragdoll, defaulting to `5`;
values below `0.001` are clamped up at the build.

`set_ragdoll_bone_mass` overrides it for individual bones. Authoring only.

## `set_ragdoll_dynamic(dynamic)`

The live control. Called with `true`, every bone switches from kinematic to dynamic and the
solver owns the ragdoll: this is the death trigger. Called with `false`, the bones switch back to
kinematic, where the animated pose drives them again.

A body switched to dynamic has its linear and angular velocity zeroed as it goes limp, so a
ragdoll never launches itself from however the kinematic drive last moved it.

Silent no-op before the ragdoll has been built (there is nothing yet to switch), and silent when
it is already in the state being asked for.

```lua
if health <= 0 then
    entity:set_ragdoll_dynamic(true)
end
```

## `is_ragdoll_dynamic()`

Whether the ragdoll is currently simulated. `false` when there is no ragdoll, before it is built,
and outside Play mode.

## `has_ragdoll()`

Whether a live ragdoll exists for this entity, meaning the build has happened and produced
bodies. `false` in the editor (the build needs a running physics world), `false` when the entity
has no Ragdoll component, no uploaded skeletal mesh or no pose to build from, and `false` when
none of its named bones resolved.

This is the call to poll when the question is "is the ragdoll ready", since nothing here can
build it early.

## `set_ragdoll_physical_weight(jointName, weight)`

"Active ragdoll": how strongly this bone's body is pulled toward its own animated and IK'd target
every frame, position and rotation both, while the ragdoll is dynamic. `0`, the value every bone
starts at, is a plain free dynamic body; `1` follows the animation at full weight. The weight is
clamped into 0 to 1 when the ragdoll builds.

The position half matters even for a purely rotational feel: the constraints between ragdoll
bones only share a point in space, so nothing else holds an unweighted root back from free
falling.

Authoring only, and read at the build: a weight set on a bone before the ragdoll exists is
remembered for when it does. It is applied only while dynamic, so it does nothing to a kinematic
ragdoll.

```lua
-- A death that still reaches for the weapon it was holding.
entity:set_ragdoll_physical_weight("spine_01", 0.8)
```

## `set_ragdoll_bone_mass(jointName, mass)`

A mass override for one bone, in kilograms. `-1`, the value an override starts at, means "use the
ragdoll's own mass" from `set_ragdoll_mass`, and any value of zero or less means the same; only a
positive mass is used.

Authoring only, read at the build. An override for a joint that is not one of the ragdoll's named
bones is simply never looked up.

## `set_ragdoll_bone_limits(jointName, swingDegrees, twistDegrees)`

How far this bone's joint may swing and twist, in degrees (rotations are degrees throughout this
API unless a call says otherwise).

The joint is a swing-twist constraint anchored at the shared joint, with its twist axis along the
segment between the bone and its ragdoll parent. Both limits are clamped into 0 to 180 at the
build. The default, for a bone this was never called on, is 180 and 180, which reads as
unconstrained: an unauthored ragdoll behaves like a plain ball joint throughout. The typical use
is tightening one joint, a knee that should not bend backwards, and leaving the rest alone.

Authoring only, read at the build.

```lua
-- A knee that bends one way only.
entity:set_ragdoll_bone_limits("lower_leg_l", 5, 90)
```

## Reading a ragdoll back

There is no separate ragdoll pose reader. Once the ragdoll is dynamic it writes its joints into
the same evaluated pose the [skeletal readers](08-skeletal-and-ik.md) already read, so
`bone_position` and `ik_joint_position` show where the simulation actually put the bones.
