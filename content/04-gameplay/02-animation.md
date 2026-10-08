---
title: "Animation"
---

How a skinned character actually moves: one component on the entity, one graph asset that
does everything else.

If you only want a prop with a single looping idle, you do not need any of this — put an
**Animator** on it, leave the mode at Single Clip, type the clip name, done. The rest of
this chapter is about the graph.

---

## The shape of it

There is exactly **one animation component**: **Animator**. (Ragdoll is separate, and is
physics rather than animation.) Everything else — blending, state machines, IK, transitions
and their conditions — lives inside a **`.Lanimgraph`** asset that the Animator points at.

That split is the whole design. Because the behaviour is an asset and not a pile of
components, two characters can share one graph, and editing it updates both.

A graph has three nested levels, and you move between them by double-clicking in and
clicking the breadcrumb to get back out:

```
State Machine          states as nodes, transitions as links
  └─ State             double-click a state → its Pose Graph
       └─ Pose Graph   nodes feeding one mandatory Animation Output
  └─ Transition        double-click a link → its Condition Graph
       └─ Condition    a visual script ending in one Condition Result node
```

A pose graph can contain a **State Machine** node, whose states have pose graphs of their
own, which can contain more state machines. **There is no nesting limit.**

---

## Making one

**Asset Browser → New… → Animation Graph**, or the **New Graph…** button on an Animator.
Either way you get a graph with one `Idle` state already holding an Animation Sequence
wired into its Animation Output, because an empty canvas does not tell you that both of
those are mandatory.

Then point an Animator at it: set **Mode** to Animation Graph and pick the asset.

---

## The state machine

The top level. Each box is a state; each arrow is a transition.

| Action | How |
|---|---|
| Add a state | Right-click the canvas → Add State |
| Make a transition | Drag from one state's **out** pin to another's **in** pin |
| Open a state | Double-click it |
| Edit a transition's condition | Double-click the link, or right-click → Edit Condition |
| Set the entry state | Right-click a state → Set as Entry (it gets an `[ENTRY]` badge) |
| Delete | Select and press Delete |

**Any State** is a reserved node on the left. A transition drawn from it can fire no matter
what is currently playing — a hit reaction, a death. Transitions out of a specific state are
always considered before Any State ones, so a targeted link gets first refusal.

Transitions carry a **Fade (s)** crossfade time. While a crossfade is running, no new
transition is taken — see [Limits](#limits-worth-knowing-before-you-hit-them).

---

## Inside a state: the pose graph

Everything here feeds the one **Animation Output** node. It is pulled backwards from that
node, so anything not connected to it does not run.

| Node | What it does |
|---|---|
| **Animation Sequence** | Plays one clip. Speed, Loop |
| **Blend Space 1D** | Clips along one axis — Idle at 0, Walk at 2, Run at 6 — picked by a Float input |
| **Blend Space 2D** | Clips on a plane, e.g. speed × direction, by barycentric weight |
| **State Machine** | A whole nested machine. Double-click to go into it |
| **Blend** | Two poses, one alpha |
| **Layered Blend** | Two poses blended *per joint* — name the joints that take pose B. This is upper-body/lower-body layering |
| **Add (Additive)** | Adds a difference pose on top of a base |
| **Make Additive** | Builds that difference pose: Source minus Reference |
| **One Shot** | Plays a shot over a base pose with its own lifetime and fades, triggered by a parameter |
| **Time Scale** | Scales time for everything above it. A whole subtree slows together |
| **Time Seek** | Pins everything above it to an absolute time |
| **Two Bone IK** | Solves a limb (root/mid/end joint) onto a target |
| **Look At IK** | Aims a joint chain at a target |
| **Parameter** | Reads one of the graph's named parameters, as a Float |

Right-click the canvas to add nodes. A **Float** input with nothing wired into it gets a
little drag field on the node, so you do not need a Parameter node just to supply a
constant.

### Driving a blend with a parameter

Add a **Parameter** node, pick the name from its dropdown, wire its Value into the blend
space's X input. Then set it from a script:

```lua
entity:set_anim_parameter("Speed", 4.2)
```

---

## Parameters

Declared in the details panel on the right, each with a type (Float, Bool, Trigger) and a
default.

- The **graph** declares them and holds the defaults.
- Each **entity** holds its own live values.

That is why two characters can share one graph and still be in different states with
different speeds.

Parameters are **one flat namespace per entity**, including nested machines: a parameter
declared three machines deep is read and written by its bare name, without any path. The
trade-off is that reusing a name at two depths gives you one shared value.

A **Trigger** is consumed the moment a transition using it fires, so it behaves as an edge
rather than a level — you set it once, it fires once.

---

## Conditions

A transition with no condition graph fires immediately. That is what you want for a one-shot
state with a single way out.

Otherwise, double-click the link. The condition is **an ordinary visual-scripting graph** —
the same nodes and the same editor as anywhere else — with one rule: whatever reaches the
**Condition Result** node's Value pin decides whether the transition fires.

The usual shape is three nodes:

```
Get Anim Parameter ("Moving")  →  A > B  (B = 0.5)  →  Condition Result
```

Because it is a real script graph, a condition can also ask about anything else a script
can: distance to another entity, whether a key is down, a variable.

> **Conditions only evaluate while playing.** They compile to Lua and run through the
> scripting engine, which does not exist in the editor's preview. In the editor a graph sits
> on its entry state rather than walking itself, which is deliberate — you do not want
> states flickering past while you are dragging transitions around.

---

## Root motion

When a walk cycle travels forward in its clip, you usually want the *entity* to move, not
the mesh to slide away from it.

1. In the graph's details panel, set **Root Motion → Joint** to the joint that carries the
   travel (usually the root/hips).
2. On the Animator, set **Root Motion** to Extract Only or Apply To Transform.

Extract Only takes the travel out of the pose and publishes it; Apply To Transform also
moves the entity. Read it from a script with:

```lua
local delta = entity:root_motion_delta()
```

> With a **Character Controller**, use Extract Only and feed the delta to the controller's
> move call. Apply To Transform writes the transform directly, which teleports the
> controller past its collision sweep.

Only the **outermost** graph's root motion joint is used — a nested machine's own setting is
ignored, and the editor says so, because there is one entity to move and the delta has to be
measured once.

---

## Sockets

Named points on the skeleton, authored on the Animator: a name, a joint, and an offset.

Something else attaches to one with a **Socket Attachment** component (target entity +
socket name), or from Lua:

```lua
sword:attach_to_socket(character, "hand_r")
local p = character:socket_position("hand_r")
```

---

## Limits worth knowing before you hit them

These are real and current, not hypothetical:

- **No interrupting transitions.** While a crossfade is running, no new transition is
  considered. A hit reaction landing during a death fade waits for the fade.
- **No "at end" transitions.** You cannot say "take this transition when the clip finishes"
  — a condition has no access to the playing state's remaining time.
- **No transition priority.** When several conditions are true at once, authored order wins
  (targeted before Any State). Deterministic, but you cannot re-rank them.
- **IK is a node, not a post-blend pass.** Two states crossfading each solve their own IK
  and then the *results* are blended, which is wrong for foot locking mid-transition.
- **Blend space clips are always phase-matched** to a weighted-average duration. Usually
  what you want; occasionally surprising when clips differ wildly in length.
- **Skeletal meshes are raster only** and are not frustum culled.

---

## See also

- [Components → Animator](../02-building-worlds/02-components.md#animator) — the field reference
- [Visual Scripting](../06-scripting/01-visual-scripting.md) — the node editor conditions reuse
- [Lua Scripting](../06-scripting/02-lua/api/entity/07-animation.md) — the full animation API
- [`docs/ANIMATION_GRAPH.md`](../ANIMATION_GRAPH.md) — the engine-side design, why it is
  shaped this way, and how it compares to Godot's AnimationTree
