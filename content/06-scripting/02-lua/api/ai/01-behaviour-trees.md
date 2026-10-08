---
title: "Behaviour trees (ai)"
---

The `ai` table is one call: the way a script-driven behaviour tree task reports whether it
succeeded or failed. See [Lua Scripting](../../index.md).

## What a task script is

An entity with an `AIControllerComponent` runs a behaviour tree. A tree is a set of **states**,
each holding a list of **tasks** and the transitions that leave it. The tree is authored, never
scripted: it lives in the scene file or in its own `.Lbehavior` asset and is edited in the
editor's AI State Tree panel, the same way an animation graph is authored in a `.Lanimgraph`.
There is no Lua API that creates, edits or even reaches into a tree's structure.

A **task** is one step inside a state. Most tasks are built in: move to a point, wait N seconds,
set a blackboard value, play a sound, set an animation parameter. One task type is
**Custom Script**, and it is the reason this table exists: instead of a built-in behaviour, the
task names a `.Lgraph` (or a `.lua` file), and that script decides when the task is finished.

A task script is an ordinary script in everything but its lifecycle:

- it is loaded when its task first ticks, and its `on_create` runs then
- `on_update(dt)` runs every tick while the state is active
- `on_destroy` runs when the tree moves on
- `entity` is bound to the entity the tree belongs to, and every global (`log`, `scene`,
  `physics`, `data`, ...) is the same one any other script gets

The differences are small but they are what the page is about. The script is keyed by an opaque
handle rather than by an entity, so one entity can have several task scripts live at once: two
Custom Script tasks in the same state, or the same state reached again while an earlier copy is
still running. And the script ends when the task reports a result, not when the entity is
destroyed. While a Custom Script task runs, the script **is** the task, and nothing else decides
when it is done.

A task script is usually authored as a visual graph, in the same node editor as any other graph,
because the two calls it needs are nodes there: **Task Succeeded** and **Task Failed**, in the
node editor's **AI** category. A plain `.lua` file works identically, and `ai.set_task_status` is
the same call from either.

## `ai.set_task_status(status)`

```lua
ai.set_task_status("succeeded")
```

Reports this task's result to the tree that is running it. `status` is one of three lowercase
strings, and anything else is refused with a warning naming the three:

| Status | What the tree does |
|---|---|
| `"succeeded"` | The task is done. The script is unloaded (its `on_destroy` runs) and the state is free to transition on. |
| `"failed"` | The task did not succeed. Same unloading, and the state machine is free to move on. |
| `"running"` | Nothing. The task stays active. |

**Running is what a task is by default.** It is also what not calling this means, which is why
there is no `"running"` node in the node editor: a script that has not reported anything yet is
Running. The status the tree reads is whatever the script left behind when its `on_update`
returned, so if a script calls this more than once in a tick, the last call is the one that
counts.

**It is read on the same tick it is set.** A task script that succeeds in its `on_update` has
its state's transitions evaluated in the same frame's tree tick, so nothing has to be deferred.
The same is true from `on_create`: an instant-result task (a check that either passes or does
not) can report from `on_create` and be acted on before `on_update` is ever called.

**An unknown status is a warning, and changes nothing.** `ai.set_task_status("Success")` writes
`ai.set_task_status: unknown status 'Success' (expected running/succeeded/failed)` to the log
(see [Logging](../debug/02-logging.md)) and leaves the task exactly as it was.

**A call with no task script running is silently ignored.** The `ai` table is bound on every
script, not only on task scripts, so an ordinary `ScriptComponent` script calling this from its
own `on_update` gets nothing: no error, no log line, no change to anything. That is the same
"no throw, just does not do anything" contract every other context-dependent call in the engine
follows, and it is worth knowing which way round it fails: a script that thinks it is reporting
a result and is not has no symptom at all, so if a Custom Script task seems to have no effect,
check that the tree actually loaded it (its `on_create` should be running) before suspecting the
status call.

**An error in a task script is a failed task.** If `on_update` raises, the Lua error is logged
with the message and the task is marked Failed, so the tree moves on instead of sitting forever
in a state whose script has stopped working. If `on_create` raises, the script does not load at
all, which the tree also treats as a failed task on that first tick.

**Leaving and re-entering the state starts the script over.** A terminal status drops the
script, so coming back to the same state loads a fresh copy and runs `on_create` again: the
graph is not resumed where it left off. Anything a task has to remember across re-entries
belongs somewhere that outlives it, which in a tree means the blackboard
(`entity:ai_set_blackboard`), not a Lua local.

```lua
-- A Custom Script task: walk to a point, then report whether we got there.
local elapsed = 0

function on_create()
    entity:move_to(entity:get_position() + vec3.new(5, 0, 0))
    elapsed = 0
end

function on_update(dt)
    elapsed = elapsed + dt

    if entity:is_move_complete() then
        ai.set_task_status("succeeded")
    elseif elapsed > 8 then
        ai.set_task_status("failed")
    end
end
```

The `failed` branch is the part worth copying: a task that can never arrive reports a failure
rather than sitting in its state forever, which is also how whatever comes next in the tree gets
its turn.

## What is around this call

Everything else a task touches is the ordinary script API, and the entity's own AI methods in
particular, documented in [the entity page](../entity/14-ai.md):

| | |
|---|---|
| `entity:move_to(target)` / `entity:stop_moving()` | Steering, through the same navmesh the tree's own MoveTo task uses |
| `entity:is_move_complete()` | Whether the agent has arrived |
| `entity:ai_current_state()` | Which state the tree is in, for a script reacting to the AI's decision |
| `entity:ai_get_blackboard(name)` / `entity:ai_set_blackboard(name, value)` | Reading and writing the tree's named parameters |

Movement needs a navmesh, and the navmesh has to exist **before the first tick**: the tree's
entry state starts evaluating on frame one, ahead of any script's `on_update`, so a scene that
bakes from `on_update` finds nothing loaded and the task fails immediately. `scene.bake_navmesh`
is documented in [the scene page](../scene/01-scene.md), and the trap itself in
[Navmesh and behaviour trees](../../../03-cpp/api/ai/01-navmesh-and-behaviour-trees.md) on the C++ side.

C++ equivalent: a module has no `set_task_status`. The module-side sibling of a Custom Script
task is a task naming a function the module exported, called each tick with the entity and the
frame's delta time, which answers its status as a number (0 Running, 1 Succeeded, 2 Failed).
The tree queries a module can make instead are in
[Navmesh and behaviour trees](../../../03-cpp/api/ai/01-navmesh-and-behaviour-trees.md).

## What is not here

There is no way to author a tree from Lua. States, tasks and transitions live in the scene file
or in a `.Lbehavior` asset, and the editor is where they are built. Lua can read a running tree
(`entity:ai_current_state`, `entity:ai_get_blackboard`), inject values into one
(`entity:ai_set_blackboard`), steer an agent (`entity:move_to` and friends), and report a Custom
Script task's result. It cannot add a state, add a task, or ask what a state contains.

See also `docs/NATIVE_API_REFERENCE.md` and `docs/ANIMATION_GRAPH.md` for what a tree and its
tasks are in the engine's own terms, and [AI Agents](../../../../07-projects-and-tools/04-editor-ai-agents.md) for the editor's
MCP surface, which builds and inspects trees the same way the panels do.
