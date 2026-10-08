---
title: "Troubleshooting"
---

## Read the logs first

```
LAB/logs/LAB.log
LAB/logs/LABEngine.log
```

Both are flushed per record, so they survive a crash. **Vulkan validation messages go through
the same log**, which is why reading the file beats watching the console — the console shows
you about half of it.

Validation is enabled by default in Debug builds.

---

## Nothing renders / everything is black

| Check | |
|---|---|
| Is there a light? | A scene with no lights renders black. Add a Directional Light and an Ambient Light |
| Is the camera inside geometry? | Fly out and look back |
| Is the mesh path right? | The log says so when a model fails to load |
| Is ambient at zero? | An Ambient Light at 0 intensity with no other light gives you nothing |

## The scene is visible but very dark

Ambient defaults to 0.1, which is deliberately dim. Either raise it, or add a **Sky** with
*Contribute Ambient* on — an environment-derived ambient looks far better than a flat grey.

## A light stops working when I add more

There are hard limits: **4 directional, 16 point, 8 spot** per scene. Lights past the limit
are simply not lit.

## Shadows are not appearing

- **Cast Shadows** is off by default on every light. Turn it on.
- Only the **first** directional light with Cast Shadows on casts. There is one sun.
- Check the shadow view budget: **16 views total**, and a shadowed point light costs six, a
  directional costs one per cascade. Exceeding it is reported in the log.
- Is the caster inside **Shadow Distance**? The directional shadow only covers a box centred
  on the camera.

## Shadows have stripes, or float away from objects

That is bias.

- **Stripes on flat surfaces** ("acne") — bias is too low.
- **Shadow detached from its caster** ("peter-panning") — bias is too high.

The defaults suit roughly human-scale scenes. A much larger or smaller world needs
adjustment.

## A material looks wrong

Open **Windows → G-Buffer Inspector**. It shows the renderer's intermediate targets, which
tells you which stage is at fault:

| Buffer looks wrong | Likely cause |
|---|---|
| Albedo | Wrong texture, or a tint on the material colour |
| Normal | Normal map not tangent-space, or channels swapped |
| AORM | The metallic-roughness map is packed wrong — it is **R = AO, G = roughness, B = metallic** |
| World position | A transform problem, not a material one |

A metal that looks black usually just has nothing to reflect — give the scene a Sky or HDRI.

If the image is too dark or washed out, adjust Camera **Exposure** (or the editor viewport
Camera popup when not playing). AgX is fixed as the tonemapper; use Renderer Settings contrast
and saturation for artistic changes.

## My HDR monitor still looks SDR

Turn on **HDR10 output** in the **Diagnostics** panel. It is enabled only when the current monitor/driver
advertises Vulkan's HDR10 PQ swapchain format. Toggling it recreates both the swapchain and
the 10-bit viewport target on the next frame; a log entry confirms `HDR10/PQ` presentation.
If the checkbox is disabled, enable HDR in Windows for that display, make the editor window
live on that display, then restart the editor. The renderer intentionally falls back to SDR
when HDR10 is not advertised instead of sending PQ pixels to an SDR swapchain.

## Nothing happens when I press Play

- Is there an entity with a **Camera** component with **Primary** ticked? Without one there
  is nothing to render through.
- Did the scripts load? The log says `Scripts started: N running, M failed`, and names any
  that failed.

## A script does nothing

- **Scripts only run while playing.** Nothing executes in edit mode.
- Check the log. A script that errors is **disabled after its first failure** rather than
  spamming — so the message appears once, near the start.
- An error in `on_create` stops the script from loading at all.
- `scene.find("Name")` returns `nil` if nothing matches, and calling a method on `nil` is
  where the error will actually surface. Prefer `scene.find_by_id` with a stable id.

## A key does nothing in a script

An unrecognised key name is silently never down — it does not raise an error. Check the
spelling: `"Space"`, `"LeftShift"`, `"UpArrow"`, `"Escape"`.

Also: input is routed through the same layer the editor uses, so a script does not see the
keyboard while an ImGui text field has it.

## Something with a rigid body will not fall

- Body **Type** defaults to **Static**. Set it to Dynamic.
- Is there a **Collider**? A rigid body with no collider is ignored entirely.
- Is the body asleep? A resting body stops being simulated. `entity:wake_up()` from a script.
- Is **Use Gravity** off, or **Gravity Scale** zero?

## A ball will not roll

Friction or a rotation lock. Rolling comes from friction at the contact point, so `Friction:
0` on *either* the ball or the ground makes it slide; and locking any rotation axis stops the
spin. `Dev/Tests/assets/scenes/roll_test.Lscene` demonstrates all three cases.

## A model or texture I dropped in does not show up in the Asset Browser

You dropped a **source** file (`.obj`, `.gltf`, `.png`, `.hdr`) straight into your project
folder. The Asset Browser lists only native assets (`.Lmesh`/`.Ltex` and the like) — find
the file in the **Source Browser** instead and import it. See
[Importing Assets](../02-building-worlds/04-importing-assets.md).

## A Character Controller entity does not move

- `entity:move(velocity)` only sets the *horizontal* component — a controller falls under
  gravity on its own, so do not expect `move` to hold it up.
- Check `entity:has_character()` before calling controller methods on an entity that might
  not have one; calling them on an entity with a Rigid Body/Collider pair instead does
  nothing, since a Character Controller replaces that pair rather than joining it.
- `entity:jump(speed)` returns `false` and does nothing when the controller is not grounded
  — check `entity:is_grounded()` if a jump seems to be silently refused.

## A trigger never fires

Triggers only notice **moving** bodies. A static object sitting inside one reports nothing,
because nothing ever happens there.

## Editing an object appears to do nothing

Save the object — that is what refreshes the instances placed in the scene. Also note that
**local edits below an instance's root are overwritten** by a refresh; only the placement
(transform, parent, name) survives.

## Undo does not undo what I expected

- **Undo is disabled while playing.**
- **The node graph has its own undo stack.** While the graph editor has focus, `Ctrl+Z`
  undoes a node edit.
- The Edit menu names what will actually be undone. Read the label if in doubt.

## Everything runs at exactly 60 FPS and changes make no difference

VSync. Turn it off in the **Diagnostics** panel before believing anything in the
**Performance** panel — with it on, every renderer change looks free because the frame is
waiting on the display either way.

## Reload Shaders does nothing / says a shader failed

**Reload Shaders reads `.spv` files and does not invoke a compiler.** Compile first:

```bash
powershell -ExecutionPolicy Bypass -File scripts/Build-Shaders.ps1
```

A file that fails to load leaves the running pipelines alone, so a typo does not black-screen
the editor — but it also means nothing changed.

## A ray-traced effect is ticked and nothing changed

Check the *Ray tracing* section of the Renderer Settings component for the note saying the device
tier is honouring less than the settings ask for. A tier lowers what the renderer honours and
never what the scene stores, so the tick stays where you put it.

If the whole section is greyed out, the device has no hardware ray query. `LABEngine.log` names
what was found on the `Renderer tier:` line at startup.

Global illumination in particular needs the top tier; on *Limited ray tracing* it is switched
off and DDGI answers for indirect light instead. See
[Hardware Ray Tracing](../03-rendering/03-ray-tracing.md).

## Ray-traced shadows and the shadow map both seem to be darkening things

They should not: the ray pass reports how much of the sun's visibility it is entitled to
answer for and lighting blends between the two by that coverage. If a contact shadow looks
like it is sitting *inside* a second, softer shadow, that is the symptom of the two being
multiplied instead — worth reporting with the scene attached.

Check **Ray Distance** first. It is the hybrid hand-off: rays own everything inside it and the
shadow map owns everything beyond, so a value that is too small puts the transition somewhere
visible.

## Traced results look noisy, or smear when the camera moves

**Denoise** (Renderer Settings → Ray tracing) is on by default. If it is off, what you are seeing is
what the rays produced — which is the point of the switch, but not what you want in a shipping
build.

If it is on and the image *smears* rather than sparkles, that is temporal accumulation holding
a history too long. Reflections are the most sensitive because they are view-dependent.

If a raised **Target GPU Time** or a *Performance* mode made it worse, the ray budget has been
cut: fewer rays is more noise. Turn **Dynamic Ray Budget** off while tuning so two runs of the
same scene are comparable.

## Global illumination looks too bright, or a wall is brighter than the floor lighting it

A bounce cannot deliver more light than the surface it came from. If a surface the sun cannot
reach directly ends up brighter than the surface bouncing onto it, the estimator is
double-counting. `Dev/Tests/assets/scenes/restir_gi_test.Lscene` and `tests/restir_gi.lua` are the fixture
for exactly this — run it and see whether it still passes.

Check **GI Clamp** for single bright samples, and **Sample Cap** if the bounce is slow to
respond to a lighting change rather than wrong.

## The editor layout is wrecked

Delete `LAB/imgui.ini` and restart. The layout rebuilds from defaults.

## Assets do not resolve / paths break after moving the project

Content paths are project-relative on purpose. If something breaks after a move, check that
**Project Settings → Asset directory** still points at a folder that exists — the panel says
so in red if it does not.

Also: start the editor with `LAB/` as the working directory when you open the sample project, whose
files are relative to it. The engine's own shaders and content do not depend on it.

## Capturing a frame in RenderDoc

Launch LAB from RenderDoc, then press `F12` (or use **Capture frame** in the **Diagnostics**
panel). The same panel shows whether RenderDoc is attached; without it, `F12` takes an
ordinary screenshot instead — see [The Editor](../01-getting-started/02-editor.md).

## Reporting something that still looks wrong

Include: the log files, whether validation reported anything, what the G-buffer viewer shows,
and the smallest scene that reproduces it. Known defects are catalogued in
[`docs/REVIEW.md`](../REVIEW.md).
