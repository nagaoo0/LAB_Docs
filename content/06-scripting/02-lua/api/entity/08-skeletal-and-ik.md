---
title: "Skeletal animation and IK"
---

Reading a skinned entity's evaluated pose: whether its mesh uploaded, how far the animation has
advanced, and where a joint is.

see [Lua Scripting](../../index.md).

These are readers. The pose they read is rebuilt every frame by the skeletal update from the
entity's Animator (or by its ragdoll) and written to a pose component that is deliberately never
serialized; a scene file has no business carrying a pose. Nothing here authors a clip, a blend or
an IK solve, and actually driving the animation is what the [Animator
calls](07-animation.md) are for. IK is a node inside the graph, so the pose these calls read
already includes whatever solves the graph ran. See [Animation](../../../../04-gameplay/02-animation.md).

Two things have to exist before any of this answers with real numbers:

- **An uploaded skeletal mesh.** The mesh resolves, its buffers go to the GPU, and its cache key
  is set. Before that there is no skeleton to evaluate.
- **Something posing it**: an Animator, or a Ragdoll component (which needs a pose to build its
  bodies from). An entity with neither has no pose, and every reader here answers zero.

A destroyed reference answers the same neutral values rather than raising.

## `skeletal_vertex_count()`

The number of vertices in this entity's uploaded skeletal mesh buffers, and `0` before the mesh
has uploaded, with no Skeletal Mesh component, or on a reference that no longer resolves.

This is a test reader rather than something a game tends to branch on, but it is the direct
answer to "did the ECS actually hand this mesh to the GPU", which CPU-side numbers alone cannot
tell you.

## `skeletal_index_count()`

The same for the index buffer, under the same conditions: an uploaded mesh or `0`.

## `animator_time()`

Seconds into the clip the Animator's own playback is playing, and `0` on an entity with no
Animator.

This is the Animator's clip clock, and only single clip mode advances it. While a graph is
playing, the graph runs on its own clock and this field is left where it was, so a graph-driven
entity reads whatever its last single-clip playback left behind (usually `0`). As a test reader
it answers "is playback actually advancing as the scene steps" for the single clip case.

## `bone_count()`

How many joints the entity's currently evaluated pose has, which is the skeleton's joint count
once a pose has been sampled. `0` when there is no pose to read.

## `bone_position(jointIndex)`

One joint's position out of the evaluated pose. `jointIndex` is 0-based, so `0` is the root joint,
and a zero vector is the answer when there is no pose or the index is out of range.

This reads the pose in the entity's own **mesh space**: the joints composed down the hierarchy,
before the entity's own position, rotation and scale are applied. A joint on an entity that has
been moved across the scene reads exactly the same as it did before the move.

For a world-space position use `ik_joint_position` below. For a point that is authored on the
skeleton rather than a joint (a muzzle, a grip), resolve a socket with `socket_position`.

```lua
-- A limb's own position this frame, for a debug overlay.
local knee = entity:bone_position(2)
```

## `ik_joint_position(jointName)`

The same pose, by joint name rather than by index, converted to **world space**: the entity's
world transform times the joint's mesh-space transform. The name is looked up in the entity's
current rig, so a joint that was renamed by a mesh swap resolves to nothing.

The "ik" in the name is about intent, not a separate source: an IK solve is a node in the graph,
and this reads whatever the last skeletal update wrote, IK included. There is no call here that
authors a solve.

A zero vector when the entity has no Skeletal Mesh component, no pose, no resolved skeleton, or
no joint by that name.

```lua
-- Where the hand ended up this frame, after the graph and any IK nodes in it.
local hand = entity:ik_joint_position("hand_r")
log.info(("hand at %.2f, %.2f, %.2f"):format(hand.x, hand.y, hand.z))
```
