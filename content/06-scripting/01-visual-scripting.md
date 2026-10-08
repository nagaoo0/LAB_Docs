---
title: "Visual Scripting"
---

A **node graph** is a script you wire together instead of typing. It saves as a `.Lgraph`
file and is assigned to a **Script** component exactly like a `.lua` file — as far as the
rest of the engine is concerned, they are the same thing.

Graphs compile to Lua and run through the same scripting host, so everything in
[Lua Scripting](02-lua/index.md) about when scripts run, per-entity state and error
handling applies here too.

## Creating and opening a graph

- Add a **Script** component and press **New Graph**. It writes a starter graph under
  `scripts/` and assigns it. It never overwrites: if `scripts/<name>.Lgraph` exists it tries
  `<name>_1`, `<name>_2` and so on.
- Or double-click an existing `.Lgraph` in the Asset Browser.
- Or open the panel from **Windows → Node Graph**.

## The editor

| Control | |
|---|---|
| **Save** | Writes the graph. The title bar shows the path and a `*` when unsaved |
| **Show Lua** | The generated Lua source, so you can see exactly what your graph does |
| **Regenerate** | Recompile the preview |
| **Undo / Redo** | `Ctrl+Z` / `Ctrl+Y` — **the graph has its own history**, separate from the scene's |
| **Variables** | Toggle the variables sidebar |
| **Style** | Node editor appearance. Saved with your editor preferences, not with the graph |

Right-click the canvas for the **Add Node** menu, organised by category. Right-click a node
or link to delete it.

> While the graph editor has focus it owns `Ctrl+Z`. That is deliberate — it is a different
> document with its own history, and one stack spanning both would undo a node edit when you
> meant to undo an entity edit.

## How a graph runs

Two kinds of connection:

- **Execution** (the thick white links) — the order things happen in. Execution starts at an
  event node and flows along these links.
- **Data** (the coloured links) — values pulled in as they are needed. A data node runs
  because something downstream asked for its output, not because execution reached it.

So a graph does nothing until an **event** node fires it.

### Events

`On Create`, `On Update` (which outputs `Delta`), `On Destroy`, `On Key Down`, `On Key Up`,
`On Mouse Down`, `On Mouse Up`, `On Collision Begin`, `On Collision End`, `On Overlap
Begin`, `On Overlap End`, plus `Custom Event` / `Call Event` for your own.

## Node categories

Rows follow the order the Add Node menu groups them in.

| Category | What is in it |
|---|---|
| **Events** | On Create, On Update, On Destroy, input and contact events, Custom Event, Call Event |
| **Flow** | Branch, Sequence, For Loop, While Loop, Do Once, Flip Flop, Gate, Every N Seconds, Delay |
| **Actions** | Log, Set Position, Translate, Set Rotation, Rotate, Set Scale, Set Light Intensity, Set Text, Set UI Visible, Destroy Entity, Set Variable (plus the typed Set Vector/Boolean/Text Variable) |
| **Entity** | Get Position/Rotation/Scale/Name, Get Forward/Right/Up, Find Entity, Self, Is Valid |
| **Objects** | Spawn Object |
| **Scene** | Preload Scene, Append Scene, Open Scene, Set Persistent, Is Persistent — see [Scene streaming](#scene-streaming) |
| **Physics** | Add Force / Torque / Impulse (and at-point variants), get/set velocity and angular velocity, acceleration, sleeping, wake up, has rigid body, Raycast, gravity and terminal velocity |
| **Input** | Key down/pressed/released, key axis, mouse down/pressed, position, delta, wheel, capture; gamepad connected, buttons, axes, sticks; Is UI Hovered, Is UI Clicked |
| **Time** | Delta Time, Elapsed Time |
| **Math** | Add, Subtract, Multiply, Divide, Modulo, Power, Negate, Abs, Sign, Sqrt, Floor, Ceil, Round, Min, Max, Clamp, Lerp, Map Range, Sin, Cos, Atan2, DegToRad, RadToDeg, Random Range, Pi |
| **Vector** | Make/Break Vec3, add, subtract, scale, multiply, length, distance, normalize, dot, cross, lerp |
| **Logic** | Greater, Greater Equal, Less, Less Equal, Equal, Not Equal, And, Or, Not, Select |
| **String** | Concat, To String |
| **Values** | Number, Text, Boolean, Vector, Get Variable (plus the typed Get Vector/Boolean/Text Variable), Get Anim Parameter, Condition Result |

Two of those Values nodes exist for **animation transition conditions** rather than for
ordinary gameplay graphs — see [Animation → Conditions](../04-gameplay/02-animation.md#conditions):

- **Get Anim Parameter** reads one of the target entity's animation-graph parameters by
  name, as a Float. It has the usual Target pin, so it defaults to the entity the graph
  belongs to. Nothing stops you using it in an ordinary script graph, and it works there.
- **Condition Result** is the root of a condition graph: whatever reaches its Value pin is
  what the transition asks. It is the only node in the whole library that makes a graph
  evaluate to a *value* rather than run as a callback, and it does nothing in an ordinary
  gameplay graph. The animation graph editor creates one with each condition and refuses to
  delete it, so you will not normally add one by hand.

The **Select** node (Logic) is the `NodeType::SelectFloat` entry in code, but its title in
the editor — and in the Add Node menu — is just "Select".

There is no "Mouse Released" node, even though a script can call
`input.is_mouse_released` — an asymmetry between the two authoring paths, not a
simplification either one made on purpose.

The Add Node menu is generated from the same table the nodes themselves are, so what you
see in the categorized menu is close to exactly what exists, with one standing exception:
**Reroute** (see [Reroutes](#reroutes) below) carries the category "Utility", which is not
one of the categories above and therefore never appears in the categorized tree. It is
still reachable — type "reroute" into the Add Node search box, which matches every node
regardless of category.

## Targets

Most Entity, Action and Physics nodes have a **Target** pin, and it is the **last** input on
the node. Leave it unconnected and the node acts on the entity the script is attached to.
Connect a `Find Entity` or `Self` node to make it act on something else.

## Variables

Open the **Variables** sidebar to declare them. Four types: **Number**, **Vector**,
**Boolean**, **Text**. Each has a name and a default value.

Drag a variable from the sidebar into the canvas to get a correctly typed getter node; there
are `Set Variable` nodes to write them.

### Global variables

Tick **Global** on a variable (marked `[G]` in the list) to share it across every script in
the scene.

- An ordinary graph variable is **per script instance** — two entities running the same graph
  each get their own.
- A global is **one value**, initialised by whichever script loads first.

The scope is shown in the list rather than only in the variable's properties, because which
scope a name has changes what reading it means.

## Scene streaming

The **Scene** category's five nodes (Preload Scene, Append Scene, Open Scene, Set
Persistent, Is Persistent) are the node-graph side of runtime level switching — see
[Scene streaming](02-lua/api/scene/02-streaming.md) in the Lua chapter for the full
behaviour, which applies here unchanged: `Open Scene` destroys every entity that is not
marked persistent, `Append Scene` never destroys anything, and both queue rather than act
immediately, so their effect lands on a later frame rather than the one that fired them.

## Not yet available as nodes

The **Character Controller** API (`move`, `jump`, `is_grounded`, and the rest — see
[Physics](../04-gameplay/01-physics.md#character-controllers)) and the physics shape-cast/overlap family
(`sphere_cast`, `capsule_cast`, `box_cast`, `overlap_sphere`, `overlap_capsule`,
`overlap_box`) exist in Lua but have no corresponding node yet. A graph that needs either
today has no node for it — this is a real gap, not an oversight to work around with a
different node.

## Reroutes

A **Reroute** is a bare dot you can drop on a link to route it around your layout. It has no
type of its own: it takes the type of whatever it is first connected to, and forgets it again
when the last link is removed. An untyped reroute accepts anything — that is how it acquires
a type at all.

## Comments

Right-click the canvas → **Add Comment** to drop a labelled box behind a group of nodes.
Right-click the comment itself → **Rename Comment** to rename it. Comments are pure
annotation — nothing in the generated Lua knows they exist.

## Reading the generated Lua

**Show Lua** is the fastest way to understand what a graph actually does, and the fastest way
to find out why one is not doing what you expected. The output is ordinary Lua using the same
API a hand-written script uses, so anything you can read in
[Lua Scripting](02-lua/index.md) applies.

If a graph is getting large enough that the generated source is easier to read than the
canvas, that is a reasonable signal to write it as a `.lua` file instead. The two are
interchangeable at the Script component.

## Sample graphs

| File | Shows |
|---|---|
| `assets/scripts/pulse.Lgraph` | The simplest useful graph |
| `assets/scripts/nodes_showcase.Lgraph` | Sequence, Do Once, Every N Seconds, variables, Clamp/Lerp, a keyboard axis, and Find Entity feeding a Target pin |
| `assets/scripts/quake_movement.Lgraph` | A full movement controller |
| `assets/scripts/trigger_graph.Lgraph` | Overlap events |
| `assets/scripts/vars_test.Lgraph` | Variables of every type |
| `assets/scripts/events_test.Lgraph` | Custom events |

`Dev/Tests/assets/scenes/script_test.Lscene` runs two of these alongside a Lua script.
