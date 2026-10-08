---
title: "Lua Scripting"
---

Attach a **Script** component to an entity and point it at a `.lua` file under your project's
asset folder. The script runs while the scene is playing, and each entity gets its own
private copy of the script's state. Two entities running `spin.Lscript` do not share variables.

Scripts are Lua 5.4 via sol2.

This chapter is the how. Every call the engine offers a script is documented in
[`LUA/`](api/index.md), one folder per area, and the [table at the end](#the-api-page-by-page)
lists all of it.

## The shape of a script

```lua
local elapsed = 0
local start

function on_create()
    start = entity:get_position()
    log.info("attached to " .. entity:get_name())
end

function on_update(dt)
    elapsed = elapsed + dt
    entity:set_position(start + vec3.new(0, 0, math.sin(elapsed * 2) * 0.5))
end

function on_destroy()
end
```

### Callbacks

| Function | Called |
|---|---|
| `on_create()` | Once, when the scene starts playing |
| `on_update(dt)` | Every frame. `dt` is seconds since the last frame |
| `on_destroy()` | When the entity is destroyed or play stops |
| `on_collision_begin(other)` | A solid contact started |
| `on_collision_end(other)` | A solid contact ended |
| `on_overlap_begin(other)` | Something entered a trigger |
| `on_overlap_end(other)` | Something left a trigger |

All are optional. A script with none of them still runs its body once at load, which is
occasionally what you want.

**An error disables the script rather than spamming the log.** The first failure in
`on_update` (or a contact callback) is reported with the file name and the Lua error, and
that instance stops running. An error in `on_create` stops the script from loading at all.

`script_failures()` and `script_failure_details()` are how a test, or an agent, reads what
went wrong without watching the log.

## What is available

Lua's `base`, `math`, `string` and `table` libraries are open. `io`, `os` and `package` are
deliberately **not**: a gameplay script has no business touching the filesystem or spawning
processes.

These globals are set up for every script:

| | What it is |
|---|---|
| `entity` | The entity this script is attached to |
| `scene`, `globals` | The open scene, and a table shared by every running script |
| `vec3`, `quat` | The two value types |
| `physics`, `input`, `ui`, `audio` | Systems |
| `game`, `render`, `ai`, `debug`, `data` | Saving, post-process parameters, behaviour trees, debug draw, reading asset files |
| `native`, `steam_input`, `steam_apps` | Calling into a C++ module; Steam Input; the game language Steam runs in |
| `log`, `print` | Logging |

`io`, `os` and `package` are gone but the same runtime also holds the test and editor APIs
(under [`testing/`](api/testing/01-test-harness.md) and [`editor/`](api/editor/01-editor-api.md)), which is why a
script can reach `fs_read` or `editor.open_scene` in a test and should not in a shipped game.

### Three rules that run through all of it

**A call that cannot do what it says answers a neutral value rather than raising.** A method
on an entity with no such component is a no-op, an unknown animator parameter reads `0`, an
unknown key is never down, an unknown input action never fires. This is deliberate: a script
generally does not know what it is attached to, and an error every frame would be worse than
nothing happening. When something silently does nothing, check spelling first and the
component second.

**Rotations are degrees.** `get_rotation`, `set_rotation`, `rotate`, `quat.from_euler` and
`q:to_euler()` all speak degrees. Radians appear only inside `quat`'s own maths.

**LAB is Z-up, and an entity's local forward is -Z.** So an entity's `get_forward()` is its
world `-Z`, `get_right()` is `+X` and `get_up()` is `+Y`, while *world* up is `+Z` because
gravity is `-Z`. A directional light is the exception that proves the rule: its direction is
its own world `+X`. See [Numbers to get right](#numbers-to-get-right).

---

## `vec3` and `quat`

[`math/01-vec3.md`](api/math/01-vec3.md) and [`math/02-quat.md`](api/math/02-quat.md) are the
reference. The short version:

```lua
local v = vec3.new(1, 2, 3)
local a = vec3.new(5)          -- (5, 5, 5)
local z = vec3.new()           -- (0, 0, 0)
```

`vec3` has `x`, `y`, `z`, `length()`, `normalized()`, `dot()`, `cross()`, and the usual
operators. A zero-length `normalized()` returns zero rather than NaN.

Euler angles stay the authoring representation. Reach for a quaternion when you are
**composing, interpolating, or aiming**, which are the three things Euler angles cannot do:
lerping from 350 to 10 degrees sweeps 340 degrees the wrong way, and `quat.slerp` takes the
short way round.

```lua
local aim = quat.between(entity:get_forward(), to - entity:get_position())
entity:set_rotation_quat(aim)
```

There is no component-wise quaternion constructor; every way to make one is named
(`quat.identity`, `quat.from_euler`, `quat.from_axis_angle`, `quat.look_rotation`, `quat.between`).

> `q:forward()` returns the rotation's own forward, which follows the same `-Z` convention as
> `entity:get_forward()`. Use the entity's when you mean "where is this thing pointing",
> since that one has the parent chain applied.

---

## `entity`

The entity this script is attached to. Other entities come back from `scene.find`,
`get_parent`, `get_children`, raycast hits and `ui.hovered()`, and have the same methods.

[`entity/`](api/entity/) documents all 149 of them. The ones a first script needs:

| | |
|---|---|
| `entity:get_position()` / `:set_position(v)` | Local. `translate(v)` and `get_world_position()` are the other two |
| `entity:get_rotation()` / `:set_rotation(v)` | Euler degrees. `rotate(v)` is additive |
| `entity:get_scale()` / `:set_scale(v)` | |
| `entity:get_forward()` `:get_right()` `:get_up()` | World basis |
| `entity:get_name()` / `:set_name(s)` | Prefer `get_id()`, which survives rename, save and undo |
| `entity:get_parent()` / `:set_parent(e)` | No argument unparents |
| `entity:get_children()` / `:get_child_count()` | |
| `entity:has_body()` `:has_character()` `:has_light()` | The `has_*` checks, worth using before a call that would be a no-op |
| `entity:valid()` | False once destroyed. A held reference outlives the entity |

The rest is grouped by what it drives: [transform](api/entity/02-transform.md),
[hierarchy](api/entity/03-hierarchy.md), [material](api/entity/04-material.md),
[physics](api/entity/05-physics.md), [character controller](api/entity/06-character-controller.md),
[animation](api/entity/07-animation.md),
[skeletal and IK](api/entity/08-skeletal-and-ik.md), [ragdoll](api/entity/09-ragdoll.md),
[HUD](api/entity/10-hud.md), [particles](api/entity/11-particles.md), [sound](api/entity/12-sound.md),
[splines](api/entity/13-spline.md), [AI](api/entity/14-ai.md),
[lights and meshes](api/entity/15-lights-and-meshes.md).

> **Setting scale does not resize the collider.** Jolt bakes scale into the shape, so it
> would need rebuilding. The mesh grows; the physics shape does not.

> **A resting body sleeps.** A force applied to a sleeping body does nothing until
> `wake_up()`. This is the most common reason a script's push "does not work".

### Animation

Driving an Animator: mode, graph, parameters, state, root motion and sockets. All of it is in
[`entity/07-animation.md`](api/entity/07-animation.md), and the system it talks to is in
[Animation](../../04-gameplay/02-animation.md).

```lua
function on_update(dt)
    local speed = entity:get_velocity():length()
    entity:set_anim_parameter("Speed", speed)
    entity:set_anim_parameter("Moving", speed > 0.1 and 1 or 0)

    if input.is_key_pressed("Space") then
        entity:set_anim_parameter("Jump", 1)   -- a Trigger; consumed when it fires
    end
end
```

Parameter names are flat across nested state machines, a Trigger reads back `0` once a
transition has taken it, and a nested machine reports its *outer* state's name.

### Scene streaming

`entity:is_persistent()` and `entity:set_persistent(bool)` decide whether this entity
survives `scene.open_scene()`. Full rules under [`scene/02-streaming.md`](api/scene/02-streaming.md).

---

## `scene`

```lua
local other = scene.find("Door")
if other then other:set_name("OpenedDoor") end

local crate = scene.spawn("objects/crate.Lobj", vec3.new(2, 0, 1))
```

[`scene/`](api/scene/) has the rest: [the table itself](api/scene/01-scene.md),
[streaming](api/scene/02-streaming.md), [picking and screen rays](api/scene/03-picking.md), and
[`globals`](api/scene/04-globals.md), the one table shared by every running script rather than
per entity:

```lua
function on_create()
    globals.count = (globals.count or 0) + 1
end
```

Two entities running the same script see `1` and `2` here, not `1` and `1` each. It starts
empty again the next time play starts.

### The queuing rule

Paths for the streaming calls are project-relative, the same as `scene.spawn`'s.
`preload_scene` runs immediately, because it only parses and caches, so there is nothing for
it to corrupt.

`append_scene` and `open_scene` are different. Both can create or destroy entities, and a
script calling one is running from inside the very loop the engine is stepping every script
instance through, so calling one from `on_update` **queues** it and applies it at a safe
point after every script has had its turn that frame. Check the result a frame later, the way
`streaming.lua` does.

**`open_scene` destroys every entity without a Persistent flag first.** Call
`entity:set_persistent(true)` on anything that has to keep running across the switch (a game
manager, a player controller, a HUD script) before calling it. A persistent entity whose
parent is about to be destroyed is detached to the root rather than taken down with it,
keeping its world placement. `append_scene` never destroys anything, so persistence has no
effect on it.

---

## `physics`

```lua
local h = physics.raycast(origin, dir, 100, entity)
if h.hit then
    log.info("hit " .. h.entity:get_name() .. " at " .. h.distance)
end
```

A raycast returns `{ hit = false }` on a miss, and on a hit also carries `entity`, `position`,
`normal` and `distance`. **Pass the caster as the fourth argument** or the ray immediately
hits the thing it started inside.

A ray is a shape of zero width: it slips through gaps nothing could fit through and misses
ledges a foot would land on. [`physics/01-physics.md`](api/physics/01-physics.md) has the three
shape casts (which sweep a shape and stop when its *surface* touches something, so a sphere
cast always reports a shorter distance than a ray by exactly its radius) and the three
overlaps, which have no direction at all and return an array of entities.

---

## `input`

Input is routed through the same layer the editor uses, so a script reads the keyboard only
when no UI widget has captured it: typing in a text field does not drive the game.

```lua
local x = input.get_key_axis("A", "D")
local y = input.get_key_axis("S", "W")
entity:translate(vec3.new(x, y, 0) * speed * dt)
```

Key names are case-insensitive and match the usual spellings: `"W"`, `"Space"`,
`"LeftShift"`, `"Escape"`, `"UpArrow"`, `"F1"`, `"1"`. An unrecognised name is simply never
down, so check your spelling if a key seems dead.

[`input/`](api/input/) covers the rest: [keyboard and mouse](api/input/01-keyboard-and-mouse.md)
(including capturing the mouse for a first-person camera, which Escape and losing focus both
release), [gamepad](api/input/02-gamepad.md) (Xbox layout with PlayStation aliases, triggers
remapped to 0..1, a 0.15 deadzone with rescaling), and
[project actions](api/input/03-actions.md).

**Actions are the ones worth using in a game.** `input.action_down("Interact")` resolves the
name against the project's own binding table, live, every call, so remapping a key in the
project takes effect immediately and a typo is inert rather than an error:

```lua
if input.action_pressed("Jump") then
    entity:jump(6.5)
end
```

---

## `ui`

Hover, press and click for HUD elements, updated once per frame from the mouse position.
There is no separate event system.

```lua
function on_update(dt)
    if ui.is_clicked("PlayButton") then
        scene.open_scene("scenes/level1.Lscene")
    end
end
```

| | |
|---|---|
| `ui.is_hovered(name)` `is_pressed(name)` `is_clicked(name)` | By the element's name |
| `ui.hovered()` `pressed()` `clicked()` | The topmost element itself, or `nil` |
| `ui.pointer_over()` | Whether the pointer is over any interactive element |
| `ui.size()` | The viewport's width, height and scale |
| `ui.events()` `ui.changed(name)` | This frame's widget events (hover, press, click of a button), see [UI Widgets](../../05-ui/03-ui-widgets.md) |

`is_clicked` is the usual button semantics: pressing on one element and releasing on another
is not a click on either. Only the topmost element wins where two overlap, the same tie-break
as draw order, a `UITransformComponent`'s `Layer`. [`ui/01-hud-interaction.md`](api/ui/01-hud-interaction.md)
has the rest, including the virtual pointer a recorded session replays through.

An element's own look and placement (text, anchor, pivot, offset, size, colour, alpha,
radius, border, texture) is edited through the entity's own `ui_*` methods, documented in
[`entity/10-hud.md`](api/entity/10-hud.md).

---

## The rest of the globals

| Global | What it is | Page |
|---|---|---|
| `audio` | Non-spatial playback, and the Music / SFX / UI mix buses | [audio](api/audio/01-audio.md) |
| `game` | Saves, saved tables, the save folder, wall-clock time, the cross-scene blackboard, window mode, quitting | [game](api/game/01-saves.md) |
| `render` | Post-process material parameters, and the `render_target_*` globals for `.Lrt` assets | [render](api/render/01-post-process-parameters.md), [render targets](api/render/02-render-targets.md) |
| `ai` | Reporting a Custom Script task's status | [ai](api/ai/01-behaviour-trees.md) |
| `debug` | Drawing a line, box, sphere, capsule, arrow or diamond for a frame | [debug](api/debug/01-debug-draw.md) |
| `data` | Reading a file out of the project's assets | [data](api/data/01-asset-data.md) |
| `log`, `print` | Lines tagged `[script]` in `LAB/logs/LAB.log` | [debug](api/debug/02-logging.md) |
| `native` | Calling a function a loaded C++ module registered | [native](api/native/01-native-modules.md) |
| `steam_input` | Steam Input, inert without the Steam library | [steam](api/steam/01-steam-input.md) |
| `steam_apps` | The game language and the Steam client's language, `nil` without the Steam library | [steam](api/steam/02-steam-apps.md) |

Two of these are worth a sentence more.

**`game` is two different things wearing one table.** `game.save`/`load` are checkpoints: the
running scene and the blackboard, together. `game.save_state`/`load_state` are the blackboard
alone, no scene and no entities, which is a progression profile rather than a save. A load is
a fixed point in time, so it deliberately undoes the streaming rule above: an entity marked
persistent that has moved since the save comes back where the file put it.

**`native.call` is the other half of the C++ chapter.** A module registers functions with
`RegisterFunction`, and a script reaches them by name:

```lua
if native.call("set_difficulty", 3) then
    log.info("difficulty set")
end
```

---

## Numbers to get right

These are the ones that produce a scene that looks almost right.

| | |
|---|---|
| Rotations | Degrees everywhere in Lua. `set_rotation(vec3.new(0, 0, 90))` is a quarter turn about Z |
| Up | LAB is Z-up and gravity is `-Z`. World up is `+Z`; an *entity's* `get_up()` is `+Y` |
| Forward | An entity's local forward is `-Z`. A directional light's is its own `+X` |
| `dt` | Seconds since the last frame, so multiply every per-frame rate by it. `entity:move()` takes metres per second and is horizontal only |
| UI units | HUD positions and sizes are in UI units, scaled by the project's reference height. An anchor of `0` is the top, `1` the bottom, and `+y` is down |
| Character jump | `entity:jump(speed)` returns whether it actually jumped, and refuses in mid-air |
| Sleep | A resting body stops being simulated. `wake_up()` before pushing it |

## Example: a complete movement script

```lua
local speed = 5
local turn = 120

function on_update(dt)
    -- Turn with A/D
    local yaw = input.get_key_axis("A", "D") * turn * dt
    entity:rotate(vec3.new(0, 0, -yaw))

    -- Walk along whatever direction we now face
    local drive = input.get_key_axis("S", "W")
    if drive ~= 0 then
        entity:translate(entity:get_forward() * drive * speed * dt)
    end

    -- Jump, if we are standing on something
    if input.is_key_pressed("Space") and entity:has_body() then
        local down = physics.raycast(entity:get_position(), vec3.new(0, 0, -1), 1.1, entity)
        if down.hit then
            entity:add_impulse(vec3.new(0, 0, 5))
        end
    end
end
```

## Sample scripts

Everything in `LAB/assets/scripts/` is a working example. `spin.Lscript` is the smallest
useful one; `player_controller.Lscript` is a complete first-person character controller.

Scripting changes are worth checking against `Dev/Tests/assets/scenes/script_test.Lscene`, which drives
one cube from Lua and two from visual graphs.

## The API, page by page

Every call a script can make, grouped the way the engine registers it. The folder index is
[`LUA/README.md`](api/index.md).

| Area | Page | What is there |
|---|---|---|
| Values | [vec3](api/math/01-vec3.md) | Constructors, components, operators, `length`/`normalized`/`dot`/`cross` |
| Values | [quat](api/math/02-quat.md) | The named constructors, composition, `slerp`, the basis it defines |
| Entity | [Identity](api/entity/01-identity.md) | `valid`, ids and names, the `has_*` checks, persistence |
| Entity | [Transform](api/entity/02-transform.md) | Position, rotation, scale, world space, direction vectors, camera FOV |
| Entity | [Hierarchy](api/entity/03-hierarchy.md) | Parent and children |
| Entity | [Material](api/entity/04-material.md) | Colour, alpha, emission, shader graph and its overrides |
| Entity | [Physics](api/entity/05-physics.md) | Forces, impulses, velocity, sleep, collision layer |
| Entity | [Character controller](api/entity/06-character-controller.md) | `move`, `jump`, grounded state |
| Entity | [Animation](api/entity/07-animation.md) | Mode, graph, parameters, state, root motion, sockets |
| Entity | [Skeletal and IK](api/entity/08-skeletal-and-ik.md) | Bone counts and positions, joint lookups |
| Entity | [Ragdoll](api/entity/09-ragdoll.md) | Building one, and tuning its bones live |
| Entity | [HUD](api/entity/10-hud.md) | Text, fonts, outline, placement, colour, hit tests |
| Entity | [Particles](api/entity/11-particles.md) | Emitting on demand, counting |
| Entity | [Sound](api/entity/12-sound.md) | Playing, stopping, parameters, muffling |
| Entity | [Splines](api/entity/13-spline.md) | Sampling, closest point, length |
| Entity | [AI](api/entity/14-ai.md) | Moving along a navmesh, reading a behaviour tree |
| Entity | [Lights and meshes](api/entity/15-lights-and-meshes.md) | Intensity, indirect intensity, the `has_*` checks |
| Entity | [Components](api/entity/16-components.md) | `add_component`, `has_component`, `remove_component` |
| Scene | [The scene table](api/scene/01-scene.md) | `find`, `spawn`, `create_entity`, `destroy`, `get_name` |
| Scene | [Streaming](api/scene/02-streaming.md) | Preload, append, open, persistence, the queuing rule |
| Scene | [Picking](api/scene/03-picking.md) | Screen rays, mesh rays, viewport projection |
| Scene | [globals](api/scene/04-globals.md) | The one table shared by every script |
| Scene | [timer](api/timer/01-timer.md) | `after`, `every`, `cancel` |
| Scene | [events](api/events/01-events.md) | `on`, `off`, `emit` between scripts |
| World | [physics](api/physics/01-physics.md) | Rays, casts, overlaps, gravity |
| Input | [Keyboard and mouse](api/input/01-keyboard-and-mouse.md) | Keys, axes, pointer, capture |
| Input | [Gamepad](api/input/02-gamepad.md) | Buttons, axes, sticks, deadzones |
| Input | [Actions](api/input/03-actions.md) | The project's binding table |
| UI | [HUD interaction](api/ui/01-hud-interaction.md) | Hover, press, click, the virtual pointer, the event queue, `ui.features` |
| UI | [Text language](api/ui/02-text-language.md) | Which fallback fonts and charset HUD text uses |
| Audio | [audio](api/audio/01-audio.md) | 2D playback and the mix buses |
| Game | [Saves](api/game/01-saves.md) | Checkpoints, `save_data` tables, `save_directory` |
| Game | [Blackboard](api/game/02-blackboard.md) | Values that outlive a scene |
| Game | [Window](api/game/03-window.md) | Fullscreen, the system cursor, quitting |
| Game | [Time](api/game/04-time.md) | Wall-clock time for save slots |
| Game | [The build](api/game/05-build.md) | `is_demo`, and what a demo or non-Steam build changes |
| Render | [Post-process parameters](api/render/01-post-process-parameters.md) | The values a post-process material reads |
| Render | [Render targets](api/render/02-render-targets.md) | `render_target_info`, `_set`, `_resize`, `_save`, `_reload`, `_create` |
| AI | [Behaviour trees](api/ai/01-behaviour-trees.md) | Task status, and what a script may not author |
| Debug | [Debug draw](api/debug/01-debug-draw.md) | The six shapes |
| Debug | [Logging](api/debug/02-logging.md) | `log` and `print` |
| Data | [Asset data](api/data/01-asset-data.md) | Reading a project file |
| Native | [Native modules](api/native/01-native-modules.md) | `native.call` and the module reflection |
| Steam | [Steam Input](api/steam/01-steam-input.md) | Action sets and controller input through Steam |
| Steam | [Steam apps](api/steam/02-steam-apps.md) | The language Steam runs the game in |
| Tooling | [Asset tooling](api/assets/01-asset-tooling.md) | Cooking, importing, thumbnails (editor only) |
| Tooling | [Test harness](api/testing/01-test-harness.md) | `expect`, pixel readback, the counters |
| Tooling | [Renderer diagnostics](api/testing/02-renderer-diagnostics.md) | Tier, timings, and every GI backend's debug view |
| Tooling | [Editor API](api/editor/01-editor-api.md) | Driving panels and documents from a test |

## Where to go next

- [Visual Scripting](../01-visual-scripting.md) for the node graphs that run the same engine
  from a `.Lgraph`, including the nodes that call into a script and back.
- [C++ Scripting](../03-cpp/index.md) for gameplay as a compiled module, which is the same
  scene seen from the other side. Its [Lua interop page](../03-cpp/api/lua/01-lua-interop.md) is the
  bridge between the two.
- [Testing](../../07-projects-and-tools/02-testing.md) for how a `.lua` file in `Dev/Tests/assets/tests/` is run against a
  real scene, which is also the best place to see these calls used for real.
