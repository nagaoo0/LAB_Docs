---
title: "Renderer diagnostics and quality knobs"
---

What a test can read out of the renderer, and the switches it can throw to make a feature
measurable. Every one of these is a test and diagnostics control: a scene authors its own
settings through its components, and nothing here exists to turn a feature on in a shipped
game.

See [Testing](../../../../07-projects-and-tools/02-testing.md) for the harness these are used from, and
[Lua Scripting](../../index.md) for the chapter this reference belongs to.

Three rules hold across the whole page, and they are the ones that catch people out:

- **A capability is never a requirement.** Ray tracing, FSR and surfel GI are all optional.
  Ask first (`ray_tracing_available()`, `fsr_available()`, `surfel_gi_available()`) and
  return early when the answer is no, exactly as `Dev/Tests/assets/tests/restir_gi.lua` does.
- **A tier clamps, it never raises.** `set_renderer_tier` lowers what the device is willing
  to honour. A tier that could turn an effect *on* would rewrite the project the first time a
  scene was opened on a faster machine.
- **Read back what you switched.** A setter writes into the scene's post-process component
  or into the renderer; whether that took effect is a separate question, and the readers on
  this page are how you answer it.

## Capability tiers

A tier is a ceiling, not a setting. The scene still authors what it wants; the tier says what
this device is willing to honour, so a scene authored on a machine with hardware ray tracing
opens on one without and looks like the same scene rendered more cheaply.

| Tier | What it allows |
|---|---|
| 0 | Raster baseline: raster shadow maps, clustered lighting, IBL, GTAO, TAA. No rays. |
| 1 | GPU-driven raster: Hi-Z, GPU culling, indirect draws. Still no rays. |
| 2 | Ray query with a mid-range budget: shadows, occlusion, narrow reflections, GPU probe updates. |
| 3 | Everything: full reflections, ReSTIR GI, the higher ray budgets, the denoiser over its signals. |

## `renderer_tier()`

The tier this device was put on, `0` to `3`, and `0` when there is no renderer. A test checks
it for consistency rather than for a value: on a device with ray query it has to be one of the
two ray tiers, and on one without it has to not be.

## `set_renderer_tier(tier)`

Forces the tier, clamped into `0..3`. This is how a test asserts a ceiling without owning four
GPUs: force each tier in turn and check what each one honours, which is what
`Dev/Tests/assets/tests/capabilitytier.lua` does. Reading a tier's effect in the frame after the change
is what makes each reading describe the tier it names.

## `running_below_authored()`

Whether this device's tier is honouring less than the scene asked for. Worth asserting
whenever a scene asks for more than the tier allows: without it, the only symptom is an
effect that was ticked, did nothing, and said nothing.

## Per-tier budgets

| Call | Answers |
|---|---|
| `max_shadow_views()` | Shadow views a frame may render at this tier. The layered image is allocated for the full budget either way, so this bounds the work rather than the memory. |
| `max_ddgi_probes_per_frame()` | Probes a GPU-traced DDGI volume may update per frame. The probe trace is the one ray workload whose cost does not fall with the screen resolution, so it is capped by the device rather than by the frame. |
| `reflection_scale_percent()` | The reflection trace's width as a percentage of the render width, so what a test asserts is that a lower tier traces at a lower rate rather than what resolution the window happened to be. `0` with no renderer. |
| `device_local_mb()` | The largest device-local heap, in megabytes. The sum across heaps is the wrong number: a discrete card reports its own memory plus host-visible system memory, and budgeting against the total is how a 4 GB card is treated as though it had 20. |

## Resolution and the display path

## `render_resolution()`

The internal render size and the display size the renderer is actually using this frame, as
`{ width, height, display_width, display_height, upscaling, retro_resolution }`. This is the
direct signature that retro resolution (or `render.scale`) took effect, rather than something
inferred from how blocky the picture looks. An empty table with no renderer.

## `set_retro_resolution(enabled, targetHeight)`

Test-only. Toggles retro resolution mid-run, so one scene test can compare on against off
without two fixtures, the same convention `set_motion_blur` and `set_dof` follow. There is no
authored path for it in a scene file. `targetHeight` is clamped to `16..4096`, and the change
lands at the next `BeginFrame`, the same deferred-apply path `render.scale` uses.

## `recreate_render_targets()`

Resizes the render targets one pixel down and straight back, to exercise the image-layout and
history lifecycle without changing the camera: it returns to precisely the same dimensions, so
no projection change can accidentally repair the thing under test. Answers nothing.

## `set_vsync(enabled)`

Test-only. Lets a performance probe isolate the display-interval wait from actual render and
CPU cost, without a hand on the Diagnostics panel's own checkbox. Turn it off before believing
any frame figure, in this page's timings or anyone else's.

## `sky_mode()`

The sky the lighting uniform actually carries, after every fallback has been decided:
`1` for a procedural sky, `2` for an HDRI, `0` for none. Not "the scene has a SkyComponent",
which would be trivially true while the background stayed the flat clear colour.

## `disable_view_effects()`

Turns off everything that filters or darkens the composite after the GI image: TAA, bloom,
motion blur, depth of field, GTAO, RT ambient occlusion and ReSTIR DI, on every post-process
component in the scene. A comparison between the GI reference and a backend then measures the
GI term alone. Answers nothing.

## `set_motion_blur(enabled, intensity, samples)`

## `set_dof(enabled, aperture, focalLength, focusDistance, maxCoC)`

Test-only mid-run toggles for motion blur and depth of field, so a single scene test can
compare on against off rather than needing two scene fixtures. Motion blur samples are clamped
to `2..32`; the DOF parameters are written through as given. See
`Dev/Tests/assets/tests/postprocess_motionblur_dof.lua`.

## Timings and switches

## `frame_timings()`

The frame's timings as the Performance panel reports them, or an empty table with no renderer.
It carries the GPU pass times (`gpu_offscreen_ms`, `gpu_ray_effects_ms`, `gpu_ray_trace_ms`,
`gpu_gi_backend_ms`, `gpu_denoise_ms`, `gpu_culling_ms`, `gpu_hiz_ms`), the two CPU stalls
(`wait_own_fence_ms`, `wait_imgui_fence_ms`), whether VSync is on, the main loop's own segments
(`cpu_poll_ms`, `cpu_layers_ms`, `cpu_ui_ms`, `cpu_render_ms`, `cpu_present_ms`), and
everything published through the frame metrics: the frame period split into CPU busy and
waiting, where the waits happen (acquire, fences, present), recording time, the GPU gap
between frames, and the prep package's counters. Names starting with `_` are internal
accumulators and are left out.

The two CPU stalls are the reason it exists: they were the numbers a test could not otherwise
attribute, which is what made "the ImGui fence wait is 10 ms" impossible to place.

## `gpu_pass_timings()`

GPU milliseconds per named scope of the last retired frame: every GPU debug event, and every
frame-graph stage as `"stage:GBuffer"`. Scopes nest, so a parent includes its children. Empty
with no renderer.

## `cpu_zones()`

Every profiled zone hit during the frame just stepped, by name, in milliseconds. A plain
global rather than `editor.*` because it compiles into both the editor and the runtime, so a
scene test reads the exact same zone names a live editor session would. A native module's own
spans appear here too: see [Profiling](../../../03-cpp/api/profiling/01-profiling.md).

Read it the same way as the other frame figures. Compare zones within one run rather than
across two, and remember that a zone's cost includes whatever the engine does on the way in
and out.

## `gpu_timing_supported()`

Whether this device reports GPU timestamps at all. The dynamic ray budget is driven by one, so
a device that reports no timing never moves it. A test asserting the budget fell has to be
able to tell that apart from the budget failing to fall.

## `set_perf_toggle(name, value)`

Runtime switches for performance A/Bs. A boolean sets `1`/`0`; a number sets a knob. The names
are the ones the subsystem code asks for by string, and `perf_toggles()` is how you find out
what exists:

```lua
set_perf_toggle("shadergraph.compiler", false)
```

## `perf_toggles()`

Every switch the frame has read, as a table of name to value. A switch nobody read this frame
does not appear, so this is a list of what the frame actually consulted rather than of what is
defined.

## Ray tracing

These are a branch, not a requirement: the suite has to pass on a device with no hardware ray
query. The structure readers and `denoised_signal_count` are on the harness page with the rest
of the counters.

## `ray_tracing_available()`

Whether this device has ray query at all. Ask this before anything else on the page and return
early when it is false, which is what `Dev/Tests/assets/tests/restir_gi.lua` does.

## `ray_tracing_pipeline_active()`

## `ray_tracing_pipeline_supported()`

Whether ray tracing pipelines are available on this device, and whether they were used this
frame. Shadows and occlusion can go through either a ray-tracing pipeline or the ray-query
compute path, both write the same image, and nothing downstream can tell which was used, which
is exactly why it needs saying out loud somewhere.

## `ray_traced_effects_recorded()`

Whether any traced effect ran this frame. A cap the tier reports should also be a cap that
holds: on the raster tier this reads `false`, and nothing traces at all.

## `set_ray_tracing(enabled)`

Forces ray tracing off, to compare a frame against the raster path, and back on. It reaches
the renderer directly rather than the scene's components.

## `set_rt_shadows(enabled)`

## `set_rt_reflections(enabled)`

## `set_rt_ambient_occlusion(enabled)`

Toggle one ray-traced effect independently of what the scene authored, so a measurement probe
can hold every ray-traced effect on together without editing the `.Lscene` on disk. Each writes
into the post-process component of every entity in the bound scene.

## `effective_ray_quality()`

The quality tier actually used this frame, `0` (low) to `3` (ultra), after the performance mode
had its say, and `0` with no renderer.

## `ray_budget_scale()`

Where the dynamic ray budget settled, from `0.25` to `1`, or `1` with no renderer. The budget
is cut by the GPU timestamp loop until frames fit their target, so this is the number to read
alongside `gpu_timing_supported()`.

## `checkerboard_active()`

Whether reflections traced at half rate this frame. Adaptive checkerboarding is the last thing
tried: it only comes on once the budget has run out of rays to cut and the frame is still over
its target.

## GI backends

## `set_gi_backend(backend)`

Selects the global-illumination backend for the whole scene: `0` the default (ReSTIR GI), `1`
surfels, `2` radiance cascades, and `3` the GI reference. Any negative value turns GI off.

`3` is a session override on the renderer that is never authored. The scene's component is
left alone, so switching away restores exactly what was authored, and the tier still clamps as
it would anything else. Answers nothing.

## `set_gi_reference(options)`

The options for the path-traced reference (`set_gi_backend(3)`), the ground truth the other
backends are measured against. Options merge into the current settings, so a test changes one
thing at a time. An unknown key is an error rather than ignored, because a misspelt flag would
otherwise silently measure the wrong convention.

| Key | Value |
|---|---|
| `enabled` | Whether the reference is presented at all. |
| `view` | What the frame shows: `"composite"`, `"irradiance"`, `"samples"`, `"error"`, `"fallback"`. |
| `spp` | Samples per frame. |
| `max_samples` | The sample cap the accumulation stops at. |
| `bounces` | Maximum bounces per path. |
| `rr_depth` | The vertex Russian roulette starts at. |
| `stride` | Pixel stride. |
| `indirect_intensity`, `opaque_light_rays`, `hit_metallic`, `ambient_as_sky`, `hdri_ambient_scale`, `clamp_ray_distance` | Booleans, each set or cleared. |
| `backface` | `"flip"` to enable the backface flip. |

```lua
set_gi_reference({ view = "irradiance", spp = 4, max_samples = 64, bounces = 1, rr_depth = 3 })
```

## `reset_gi_reference()`

Forces a fresh accumulation. The reset is named `"manual"` in `gi_reference_stats()`'s
`reset_reason`, which is how a test tells its own reset apart from the one a camera move
causes. Answers nothing.

## `gi_reference_available()`

Whether the reference can run on this device at all, which needs ray query. A scene test that
uses the reference is gated on this and asserts validation only when it is false.

## `gi_reference_stats()`

The accumulation's bookkeeping, as a table: `available`, `present`, `running`, `samples`,
`max_samples`, `spp`, `converged`, `resets`, `reset_reason`, `key_matches`, `stride`,
`frames_since_reset`, `width`, `height`, `bytes`, `flags`, `gpu_ms`, `frame_ms`. An empty table
with no renderer.

Gate on `running` rather than on `present`: a device without ray query never records the pass.
`key_matches` says the accumulation still matches the scene, and `reset_reason` names whichever
of the camera, the scene or a manual reset invalidated it.

## `gi_reference_mean(u0, v0, u1, v1)`

The accumulation's per-pixel means over a rectangle in `0..1` coordinates, averaged, with the
standard error of that average, or `nil` when there is nothing to read. Pixels with no samples
yet are skipped rather than counted as black.

Returns `r`, `g`, `b`, `lum`, `stderr_lum`, `samples_min`, `samples_max`, `fallback_fraction`
and `nonfinite` (a count of non-finite samples), plus `pixels`, how many pixels contributed.
This is the reader the furnace test holds to a half-percent against an analytic answer, and
`stderr_lum` is what a stochastic phase is held to instead.

## `gi_image_mean(u0, v0, u1, v1)`

The same rectangle's mean from the presented GI image rather than from the accumulation, as
`r`, `g`, `b`, `lum`, `pixels`, or `nil`. Comparing it against `gi_reference_mean` is how a
test proves the image presents the accumulation rather than something of its own.

## `set_restir(enabled)`

## `restir_enabled()`

ReSTIR DI is opt-in and has no authored home in a scene file, so a test is the only way to
reach the reservoir pipeline at all. `restir_enabled()` reports whether it is on, and answers
`false` with no renderer.

## `set_radiance_cascades(cascades, spacing, range, debug)`

Writes a cascade layout into every post-process component: the cascade count (clamped `2..6`),
the probe spacing (clamped `2..64`), the range, and a debug view (`0..3`).

## `radiance_cascades_stats()`

The layout that was authored and what the pass actually dispatched last frame, as `available`,
`running`, `cascades`, `probes`, `rays`, `probe_spacing`, `layer_width`, `layer_height`,
`bytes`, `gpu_ms` and `frame_ms`. Empty with no renderer. `running` is the one to gate on: a
device without ray query never records the pass, and the scene keeps its fallback.

## Surfel GI

The surfel backend has the largest diagnostic family of any subsystem here, because its
regression tests inspect cache state directly rather than only the image. See
`docs/SURFEL_GI.md` for what each reading means in the pass itself.

The knobs first. Most of them write the layout into every post-process component, and
`set_surfel_pressure_override` reaches the pass itself. Either way, read the effect back
through `surfel_stats()` rather than assuming the write landed.

| Call | What it sets |
|---|---|
| `set_surfel_rays(rays)` | Rays per surfel, clamped `1..64`. The schedule is what the budget arithmetic keys off, and 64 is its boundary case: the plan packs the count beside the surfel id, so a count that does not fit the field corrupts the id rather than the count. |
| `set_surfel_updates(updates)` | Surfel updates per frame, clamped `64..16384`. |
| `set_surfel_max_bounce_depth(depth)` | How many further bounces a bounce that found no cached surfel may spend, clamped `0..8`. The real cost bound is the scene's own geometry and the ray distance, which is why the ceiling is low. |
| `set_surfel_debug(debug)` | Debug view, clamped `0..4`. |
| `set_surfel_ray_binning(enabled)` | Whether rays are binned by direction. |
| `set_surfel_light_tree(enabled)` | Whether the stochastic light tree is built and uploaded. |
| `set_surfel_pressure_override(pressure)` | Test-only. Forces the pressure term the recycling policy reads, bypassing the occupancy computation, because organic pressure needs on the order of 144,000 live surfels to clear the eviction onset and no test scene reaches that in a useful number of frames. A negative value clears the override. |

## `surfel_gi_available()`

Whether the surfel backend can run on this device. Gate the rest of this section on it, as
`Dev/Tests/assets/tests/surfel_cellcollide.lua` does.

## `surfel_stats()`

The pass's own numbers for the last frame, or an empty table with no renderer: `running`,
`alive`, `updated`, `moved`, `recycled`, `spawned`, `rays`, `shadow_rays`, `gather_rays`,
`requested`, `overlap_recycled`, `overflow`, `fallback_rays`, `pressure`, `bytes`, `gpu_ms`,
`frame_ms`, `local_point_lights`, `local_spot_lights` and `light_tree_nodes`.

Three of them are worth knowing by name. `requested` is what the surfels asked for before the
frame budget was applied, so a gap between it and `rays` is the throttling and equality means
the budget was not binding. `local_point_lights` and `local_spot_lights` are the true,
uncapped counts the light buffers held, not the core engine's clamped ones.
`light_tree_nodes` is 0 when no tree was built this frame.

## `surfel_history_stats()`

What the temporal history did: `valid`, `fresh`, `reused_fresh`, `fresh_with_history`,
`guided`, `depth`, `unused_with_history`, `guide_ready`, `guide_live` and `guide_filled_mean`.

`guided` says a slot has a map; `guide_ready` says the guided bounce is actually drawing from
it, which is the question that decides whether a sparse signal converges at all, and
`guide_filled_mean` is how close the rest are to the threshold.

## `surfel_pass_timings()`

GPU milliseconds per surfel stage (transform, index, spawn, the eviction stages, ray plan, ray
binning, rays, integrate, history commit, smoothing, cache commit, gather), read from the
pass's own timestamps. This is what `frame_timings()`'s `gpu_gi_backend_ms` is made of. Empty
without timestamp queries or before the first frame's readback.

## `surfel_cell_stats(x, y, z)`

The cell-average fallback's collision guard, for the bucket this world position's cell hashes
to: `valid`, `count` (what the accumulator holds), `owned` and `key` (the raw key that bucket
has claimed this frame).

`owned` compares that key against this same position, for convenience, but two positions that
share a bucket are the case worth testing and they have to be compared against *one* snapshot:
the bucket's owner is re-decided every frame, so two separate calls read two different frames
and can each correctly answer "not mine" without the two frames ever having disagreed. Take
one `surfel_cell_stats` call and check its `key` against as many candidate positions as you
need.

## `surfel_cell_key(x, y, z)`

The key a position's cell would claim, computed host side with no GPU access, so a test can
compare it against a single `surfel_cell_stats` snapshot's `key`. `0` with no renderer.

```lua
local snapshot = surfel_cell_stats(2.25, 2.25, 0.05)
local brightKey = surfel_cell_key(2.25, 2.25, 0.05)
expect(snapshot.key == brightKey, "the bucket is claimed by the cell we asked about")
```

## `surfel_nearest(x, y, z, maxDistance)`

The surfel slot closest to a world position within `maxDistance`, as `valid`, `slot` and
`normal`. This is a proximity search, unlike `surfel_attachment` below, which finds a slot by
the entity that owns it in slot-index order.

## `surfel_depth_bins(slot)`

One surfel's radial depth function, by slot: `valid`, then `mean` and `known`, both 1-indexed
tables of one entry per azimuth and elevation bin, azimuth fastest varying. Get the slot from
`surfel_attachment` or `surfel_nearest`.

## `surfel_guide_bins(slot)`

One surfel's ray-guiding map, by slot: `valid`, then `weight`, a 1-indexed table of the bin
weights.

## `surfel_guide_direction(bin, normal)`

The world-space direction a guide bin's centre points at, given the surfel's own world normal
(taken from `surfel_attachment` or `surfel_nearest`). Host side only, `bin` 1-indexed to match
the weight table `surfel_guide_bins` returns, and `vec3(0)` with no renderer.

## `surfel_attachment(entityId [, requestedSlot])`

The surfel a given entity is attached to: `valid`, and when valid `slot`, `transform`,
`generation`, `created`, `position`, `normal`, `radius`, `local_position`, `confidence` and
`irradiance`. When the entity is still in the scene it also carries `expected_position` and
`expected_normal`, recomputed from the entity's own world matrix, which is what a test compares
the surfel's stored values against.

Pass a slot to ask about a specific one; without it, the first slot owning that entity is used.

## `capture_surfel_frame(name)`

Writes the current frame to `<name>.ppm` in the test's output folder (`Dev/Out/tests/<test name>/`) and returns whether it wrote, so a human can look
at a surfel frame rather than reading numbers about it. The name may contain only lower-case
letters, digits, underscores and hyphens; anything else is refused with `false`.

## DDGI probes

| Call | Answers |
|---|---|
| `set_ddgi_visualization(enabled)` | Turns the DDGI-only debug view on or off. It is the one way a test can see the probes' contribution in isolation: everything else in the frame is direct light, and a brighter pixel proves nothing about which term produced it. |
| `ddgi_visualization_enabled()` | Whether that view is on. |
| `ddgi_gpu_tracing()` | Whether probe tracing is running on the GPU for this device and frame. |
| `ddgi_active_probe_count()` | Probes active in the volume. |
| `ddgi_traced_rays()` | Rays traced for the probes. |
| `ddgi_average_irradiance()` | The volume's average irradiance, as a number. |
| `max_ddgi_probes_per_frame()` | The per-frame probe budget this tier allows, listed with the tier budgets above. |

## FSR

| Call | Answers |
|---|---|
| `fsr_available()` | Whether FSR can be used at all on this device. |
| `fsr_library_loaded()` | Whether the FSR library loaded. It may be absent, and the provider may refuse the device. |
| `fsr_version()` | The version string, empty with no renderer. |

The suite branches on `fsr_available()` the same way it branches on ray tracing: a test asserts
what the upscaler does when it is there, and never requires it.

## Raytraced audio

The acoustics simulation reports the same smoothed statistics the audio engine's own muffle and
reverb hooks are driven from, so a test can assert on them instead of listening.

| Call | Answers |
|---|---|
| `acoustic_room_size()` | The room's size as a real distance in world units, and 0 before any ray has bounced. |
| `acoustic_ambience()` | The escape ratio, low in a sealed room and climbing as the space opens up. |
| `acoustic_ambient_direction()` | The world direction the ambient arrives from, as a vec3. |

`Dev/Tests/assets/tests/acoustics.lua` is the fixture: a listener sealed in a small box room, one source
in the same room and one walled off outside it, with generous tolerances throughout because
this is a statistical ray simulation rather than a number to pin.

## Shadows

| Call | Answers |
|---|---|
| `shadow_view_count()` | Shadow views actually rendered. A spot caster costs one, a point caster six (one per cube face), and a directional caster one per cascade. |
| `shadow_cascade_count()` | Cascades the directional light actually uses. |
| `shadow_split(view)` | The outer distance of one shadow view in world units, and 0 for a view that is not a directional cascade. |
| `shadow_casters_drawn()` | Casters drawn last frame, summed across every shadow view. |
| `shadow_casters_culled()` | Casters culled, summed the same way. Culling is per view, which is where it pays: a spot view covers far less than the camera does. |
| `max_shadow_views()` | The per-tier budget these are measured against, listed with the tier budgets above. |

`Dev/Tests/assets/tests/shadows.lua` pins the arithmetic: drawn plus culled has to divide by the view
count, because every mesh is considered exactly once per view. A cull that silently skipped
entities would still leave both counts looking plausible on their own.

## GPU culling

## `gpu_culling_stats()`

The GPU-driven culling results for the last frame whose readback fence has retired, or `nil`
before one has. The table carries `submitted`, `visible`, `frustum_rejected`,
`occlusion_rejected`, `generated_draws`, the three `lod0`/`lod1`/`lod2` counts,
`timing_supported` and `timing_valid` with their `milliseconds` and `hiz_milliseconds`,
`hiz_mips`, and `indirect_count_supported` and `indirect_count_enabled`.

The `nil` is the point: results are only read once their frame slot's fence has retired, so a
test that reads this on the first frame gets nothing rather than a stale number.

## `set_gpu_occlusion(enabled)`

## `set_indirect_count(enabled)`

Toggle the GPU culling features mid-test, so one fixture can compare the cached, occlusion
tested path against the plain one. Read the answer back from `gpu_culling_stats()`.

## None of this is a shipping-game API

A game does not switch these on. The scene's own components author what it wants, the tier
clamps that downward on the machine it is running on, and the players never see a toggle. What
this page is for is making a renderer feature measurable: force the tier, hold an effect on,
read the statistic the subsystem itself would report, and then assert something that a wrong
implementation could not satisfy.
