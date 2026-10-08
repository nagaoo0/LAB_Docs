---
title: "Animation"
---

The runtime half of the animation system: which graph an entity plays, the parameters that drive
it, what it is currently doing, and the root motion and sockets that hang off it.

see [Lua Scripting](../../index.md).

There is exactly one animation component, the **Animator** on the entity, and one graph asset,
the **`.Lanimgraph`**. Blending, state machines, transitions, IK and blend spaces are nodes
inside that asset, authored in the node editor and shared by path, so nothing on this page
builds a machine call by call. What a script wants from a graph at runtime is which graph plays,
what its parameters are, and what it is doing. See [Animation](../../../../04-gameplay/02-animation.md) for the
system itself and [Components → Animator](../../../../02-building-worlds/02-components.md#animator) for the fields.

A few things are true of the whole page:

- **Parameters are one flat namespace per entity**, nested machines included: a parameter
  declared three machines deep is still set and read by its bare name.
- **Two entities pointed at one graph share the asset and nothing else.** Each holds its own
  live parameter values, its own current state, its own crossfade.
- **Some of these calls create an Animator on an entity that has none** (`set_animator_mode`,
  `set_anim_graph`, `set_root_motion_mode`, `add_socket`). The parameter and reader calls do
  not, because a parameter on an entity with nothing to animate would only ever be read back by
  whoever set it.
- **A call that cannot do what it says answers a neutral value rather than raising**: a no-op, a
  `0`, or an empty string. The same tolerance a mistyped input action name gets.

## `set_animator_mode(mode)`

Switches the Animator between its two modes. `"graph"` evaluates a `.Lanimgraph`; **anything
else means single clip**, so `"clip"` is the natural spelling for the other mode and a typo is a
silent switch rather than an error.

Switching to single clip plays whatever clip the Animator already names, falling back to the
bind pose when that name matches nothing loaded. Switching to graph mode with no graph set does
the same, and is normally followed by `set_anim_graph`, which sets the mode itself.

A mode switch does not reset the machine: going back to graph mode resumes where the graph left
off. `set_anim_graph` is the call that starts a machine over.

Creates an Animator on an entity that has none.

```lua
entity:set_animator_mode("clip")   -- back to one clip, whatever the graph was doing
```

## `set_anim_graph(path)`

Points the Animator at a `.Lanimgraph` and switches it to graph mode in the same call. Pointing
an entity at a graph is what wanting the graph means, and a script that set the path but left the
mode alone would silently go on playing whatever single clip was authored.

The cached copy of the old graph and the runtime that indexed into it are dropped together, so
the machine starts over: the next evaluation resolves the new asset and enters its entry state.
Calling this again with the same path is therefore a restart, not a no-op.

The path is asset-relative, like every other asset reference. A path that does not resolve, or an
empty one, leaves the entity on its bind pose rather than failing.

```lua
entity:set_anim_graph("anim/character.Lanimgraph")
```

## `get_anim_graph()`

The path that was set, or `""` on an entity with no Animator.

It answers the authored string, not whether that asset actually loaded: a path that resolves to
nothing is not reported here.

## `set_anim_parameter(name, value)`

Sets one of the graph's live parameter values: a number for a Float, `1`/`0` for a Bool or a
Trigger. These are the values a graph's blend spaces and transition conditions read, so this is
the whole interface between gameplay and animation.

Silent no-op on an entity with no Animator.

A name the graph never declared is appended rather than refused, so a misspelt parameter is
inert: it reads back, and no graph looks at it. Values declared by the graph are seeded from the
graph's own defaults the first time it evaluates, and a value a script has already set is never
overwritten by that seeding.

A **Trigger** is consumed the moment a transition using it fires. Set it once, it fires once, and
it reads `0` afterwards, which is how you confirm the transition actually took it.

```lua
entity:set_anim_parameter("Speed", 4.2)
entity:set_anim_parameter("Moving", 1)
entity:set_anim_parameter("Jump", 1)   -- a Trigger
```

A native module reads and writes the same parameter from C++:
[C++ equivalent](../../../03-cpp/api/animation/01-animation-and-particles.md).

## `get_anim_parameter(name)`

The live value. `0` on an entity with no Animator, and `0` for a name that has no value yet.

Bool and Trigger parameters are floats here, so they read back `0` or `1`. A parameter the graph
declares but nothing has touched reads its declared default once the graph has evaluated, not
zero.

## `get_anim_state()`

The name of the state the outermost machine is in. `""` until the graph has resolved and
evaluated at least once, `""` on an entity with no Animator, and `""` when the live index does
not fit the resolved graph. That last case is not padding: an editor edit that removes a state
leaves every live index one past where it belongs until the next evaluation reseats it.

This is the outermost machine's state. A nested machine inside the current state has its own
current state, which this call does not name.

It reads the graph's runtime, so after a switch to single clip it keeps the last answer the graph
left.

## `is_anim_transitioning()`

True while a crossfade between two states is running. `false` on an entity with no Animator, and
`false` unless a graph crossfade is what is running; like `get_anim_state`, it reads the graph's
runtime.

While a crossfade is running, no new transition is considered, so this is also the answer to "is
the machine free to change state right now".

## `set_root_motion_mode(mode)`

`"extract"`, `"apply"`, and anything else, which means `"none"`. Which joint carries the travel
is the graph's business, because that is how the clips were authored; this decides what this
entity does with it.

| Mode | What happens |
|---|---|
| `"none"` | Ignore it. The travel stays in the pose and the mesh slides away from its entity (the default) |
| `"extract"` | Take the travel out of the pose and publish it to `root_motion_delta()` for a script to read |
| `"apply"` | Take it out and move this entity by it every frame |

> **Use `"extract"` with a character controller, never `"apply"`.** Apply writes the entity's
> transform, which teleports a controller past its collision sweep, so it walks through walls.
> Read the delta and feed it to the controller's own `move` call instead. See
> [Animation → Root motion](../../../../04-gameplay/02-animation.md#root-motion).

Creates an Animator on an entity that has none.

## `root_motion_delta()`

Last frame's extracted movement, as a `vec3` in the entity's own local space.

Zero when the graph names no root motion joint, when the mode is `"none"`, on the very first
evaluated frame (there is no previous frame to difference against), and on an entity with no
Animator.

The delta is a distance travelled over one frame, while `move` wants a velocity, so scale by
the frame time before feeding one to the other.

```lua
local delta = entity:root_motion_delta()
if delta:length() > 0 then
    entity:move(delta * (1 / dt))
end
```

## `add_socket(name, jointName [, position [, rotation [, scale]]])`

Adds a named point on this entity's skeleton, offset from a joint: what an attachment anywhere
else resolves against by name. A muzzle, a hand, a hip.

Adding a name that already exists on this entity **replaces that socket in place**, so a socket
is authored once and edited live. The three offsets are optional and default to `(0, 0, 0)`,
`(0, 0, 0)` and `(1, 1, 1)`.

> The rotation is the one rotation argument in this API that is **radians**, not degrees: it goes
> into the socket's own field unconverted, where every transform call converts on the way in.
> The Properties panel shows the same field in degrees.

Creates an Animator on an entity that has none, which is the right side to err on: a socket is a
named point on an animated skeleton, and an idle animator costs a default-constructed component,
while a socket with nowhere to live would silently never resolve.

```lua
entity:add_socket("muzzle", "hand_r", vec3.new(0, 0, 0.5))
```

## `remove_socket(name)`

Removes the named socket from this entity. Silent no-op on an entity with no Animator or no
socket by that name.

Anything attached to that socket stops resolving, and keeps the transform it last had.

## `socket_position(name)`

The socket's world position, resolved through this entity's current pose exactly the way an
attachment resolves it.

A missing socket, a joint name that is not in the current rig, or no evaluated pose yet answers
**this entity's own world position**, deliberately rather than a sentinel: one bad name should
not crash a script, and "at the entity" is also the honest answer for "nothing to attach to".
The consequence is that a typo reads as a position, not as an error, so a socket that refuses to
move is worth checking by name first.

A destroyed reference answers a zero vector.

## `attach_to_socket(target, socketName [, position [, rotation [, scale]]])`

Makes **this** entity follow a named socket on `target`. Every frame the socket is resolved and
this entity's transform is overwritten with it, composed with the offsets, which default to
`(0, 0, 0)`, `(0, 0, 0)` and `(1, 1, 1)` (the rotation is radians, as in `add_socket`).

The resolve runs in the editor as well as while playing, so an attached prop previews on its
socket while authoring.

A `nil` target detaches, the same "no argument means the neutral state" convention `set_parent`
uses, and so does a target that no longer resolves. Attaching to itself is ignored. A target with
no such socket leaves this entity where it is rather than snapping it anywhere. Silent no-op on a
destroyed reference.

Because the transform is rewritten every frame, a script should not also move an attached entity
by hand; that is what `detach_socket` is for.

```lua
sword:attach_to_socket(character, "hand_r")
```

## `detach_socket()`

Removes the attachment, leaving this entity where it currently is and free to be moved normally
again. Nothing restores where it was before it attached. Silent no-op when nothing is attached.
