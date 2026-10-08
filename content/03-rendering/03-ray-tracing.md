---
title: "Hardware Ray Tracing"
---

Everything in this chapter is **optional and off by default**. A device without hardware ray
query renders exactly as it always did, and a scene that switches these on still opens,
renders and saves unchanged on such a device — see [Device tiers](#device-tiers) for the rule
that makes that true.

All of it lives on the **Renderer Settings** component, under a *Ray tracing* section. Add that
component to a scene settings entity ([Components](../02-building-worlds/02-components.md)) and the section appears.
If the device has no ray query the whole section is greyed out with a note saying so; the
values are still saved with the scene and take effect on a machine that does.

## Contents

- [Is it available?](#is-it-available)
- [Device tiers](#device-tiers)
- [Quality, modes and ray budgets](#quality-modes-and-ray-budgets)
- [Ray-traced shadows](#ray-traced-shadows)
- [Ray-traced ambient occlusion](#ray-traced-ambient-occlusion)
- [Ray-traced reflections](#ray-traced-reflections)
- [Transmission](#transmission)
- [Ray-traced global illumination](#ray-traced-global-illumination)
- [Denoising](#denoising)
- [Test scenes](#test-scenes)

## Is it available?

The renderer needs `VK_KHR_ray_query` and the acceleration-structure extensions beneath it.
On startup it logs one line naming what it found:

```
Renderer tier: High-end ray tracing on NVIDIA GeForce RTX 4080
  (16050 MB device-local, ray query yes, RT pipeline yes, indirect count yes)
```

That line is in `LAB/logs/LABEngine.log` and is the fastest answer to "why is this greyed
out". Ray tracing is a capability, never a requirement: nothing in the frame loop depends on
it succeeding.

## Device tiers

A **tier** describes what this machine can afford. It is detected once at startup and shown
in the inspector as **Device Tier**, with the GPU name and its device-local memory beneath it.

| Tier | What it allows |
|---|---|
| Raster baseline | No ray tracing. Shadow maps, clustered lighting, IBL, GTAO, TAA |
| GPU-driven raster | The same, plus Hi-Z, GPU culling and indirect draws |
| Limited ray tracing | Shadows, occlusion and narrow reflections on a medium budget |
| High-end ray tracing | Everything, including global illumination and the full ray budgets |

Ray query on its own is not enough for the top tier. An entry-level RT part traces perfectly
well and produces a slideshow rather than a better picture when asked for ReSTIR GI at full
budgets, so the top tier also wants a discrete GPU with at least 6 GB. Below 6 GB the tier
drops to Limited, and below 3 GB to GPU-driven raster.

**A tier lowers your settings and never raises them.** This is the part worth internalising:
the tier clamps what the *renderer* honours, and never touches what the scene stores. Open a
scene authored on an RTX machine on a laptop, and it renders as the same scene more cheaply
and **saves back unchanged**. Turning an effect on is always your decision. When the tier is
honouring less than the settings ask for, the inspector says so directly — an effect that is
ticked and doing nothing has to explain itself.

You can **force a tier** from the same combo to preview how the scene behaves elsewhere, and
**Reset to detected tier** puts it back. The tier is a property of the machine, so it is
neither authored nor saved.

## Quality, modes and ray budgets

Three controls decide how many rays a frame spends. They are resolved in one place so they
can never disagree.

**Mode** is the preset: *Native*, *Quality*, *Balanced*, *Performance*. A mode is a quality
tier **and** a ray density, because the four quality tiers are a factor of two apart and most
of what "Balanced" wants lives between two of them.

*Native* is not a preset — it means "use the Quality tier below exactly as authored". That is
what lets a preset list and a manual control coexist without one silently winning; while any
other mode is selected the Quality combo is greyed out, so which one is in force is visible
rather than inferred.

**Quality** (*Low*, *Medium*, *High*, *Ultra*, default Medium) scales every ray budget at
once — shadow rays, occlusion rays and reflection rays — and is only editable in Native mode.

**Dynamic Ray Budget** (off by default) moves a continuous factor on top of that, frame by
frame, to hold the **Target GPU Time** you set (1–33 ms, default 8). The inspector shows where
it has settled as a percentage.

It is deliberately asymmetric and slow: cutting rays makes the frame faster, which immediately
argues for adding them back, so a controller that reacts equally hard in both directions
oscillates visibly. It falls at about 6% a frame, rises at about 2%, and floors at a quarter
of the mode's budget — an effect you switched on going dark is worse than a noisy one. Every
individual ray count also keeps a floor of one ray.

It is off by default on purpose: **a budget that moves on its own makes two runs of the same
scene incomparable**, which is the wrong default while you are measuring anything. Turn it on
for shipping, off for tuning.

**Checkerboard** (*Off*, *Always*, *Adaptive*) traces half the reflection pixels each frame,
the other half taking the velocity-reprojected result of the frame that traced them. The phase
flips every frame, so every pixel is traced at half rate rather than half the image never
being traced. It applies to reflections only — they are the only signal with a history image
and a velocity to reproject it with.

*Adaptive* engages only once the dynamic budget has run out of rays to cut and the frame is
still over target. Halving the trace rate is a bigger quality step than any budget reduction,
so it is the last thing tried rather than the first.

> Measure with VSync **off** (the Diagnostics panel's toggle) or every change will look like
> it did nothing. The Performance panel (Windows menu) reports the GPU pass time these
> controls are steering. See [The Editor](../01-getting-started/02-editor.md).

## Ray-traced shadows

**RT Shadows** replaces the shadow map's answer for the **sun only**. A ray per local light
per pixel is not a budget this pass has; the clustered path still shadows those from their
maps.

| Field | Default | Meaning |
|---|---|---|
| Sun Angular Size | 0.5° | The light's angular radius. Zero is a hard shadow; the real sun is about half a degree, and larger values widen the penumbra with occluder distance |
| Ray Distance | 8 m | How far a shadow ray runs. Zero traces the whole scene |

**Ray Distance is the hybrid control, and it is the interesting one.** At zero, rays own every
shadow. At a positive value the rays own the near field — where cascades are weakest and
contact shadows matter most — and the shadow map owns everything beyond.

The two never multiply. The ray pass reports how much of the sun's visibility it is *entitled*
to answer for, and the lighting blends between the map and the rays by that coverage.
Multiplying them instead darkens every pixel twice, so a contact shadow sits inside a
shadow-map shadow and one's penumbra compounds the other's bias.

Alpha-cutout materials cast a real cutout shadow here: the ray tests the material's alpha at
the hit rather than committing on geometry, so a leaf card shadows as leaves rather than as a
rectangle.

## Ray-traced ambient occlusion

**RT Ambient Occlusion** casts cosine-weighted hemisphere rays. **AO Ray Radius** (default
1.5 m) is the world-space reach and **AO Strength** (default 1) the amount.

It **replaces** GTAO wherever it runs rather than multiplying with it. They are two estimates
of the same quantity, and multiplying them darkens every crevice twice. Leaving GTAO enabled
alongside is harmless — the ray-traced result simply takes over.

## Ray-traced reflections

**RT Reflections** traces world-space rays and resolves their radiance out of the previous
frame's lit image. That is a screen-space resolve on a world-space ray: the *geometry* is
correct — reflections of objects behind the camera work, and so does off-screen occlusion —
while the colour comes from a buffer that already exists.

| Field | Default | Meaning |
|---|---|---|
| Max Roughness | 0.35 | Rougher surfaces keep the environment's specular lobe instead |
| Reflection Distance | 60 m | How far a reflection ray runs |

Above **Max Roughness** the reflection lobe is wide enough that image-based lighting already
approximates it, and tracing it spends the whole budget resolving noise a filter then blurs
back into the same result. Raising it costs real time.

Ray count scales with roughness — a mirror needs one ray, a rougher surface needs several
because its lobe is wide — so the cost of raising Max Roughness is worse than linear.

A hit that lands off-screen falls back to the sky and contributes no confidence, which is what
stops the temporal filter from trusting it.

On the **Limited ray tracing** tier, reflections trace at half resolution and are upsampled;
nothing else in the scene changes.

## Transmission

**Transmission** (under Reflections) makes transparent surfaces *tint* a reflection ray rather
than stop it, and refract it about the surface it actually crossed. That is the difference
between a window and a mirror: a reflection can see through glass to the wall behind it.

**Transmission Tint** (default 0.35) controls how strongly the crossing colours the ray.

## Ray-traced global illumination

**RT Global Illumination** is ReSTIR GI: one traced indirect bounce per pixel per frame, kept
in a reservoir so that previous frames and neighbouring pixels all contribute. It **replaces**
the DDGI probe contribution rather than adding to it — they are two answers to the same
question, and summing them double-counts the bounce.

| Field | Default | Meaning |
|---|---|---|
| GI Ray Distance | 40 m | How far a bounce ray searches |
| Sample Cap | 20 | How many past samples a pixel's reservoir may stand for |
| Spatial Radius | 16 px | How far a pixel looks for a neighbour to reuse. Zero leaves only temporal reuse |
| Validation Stride | 8 frames | How often a reservoir re-checks that the radiance it remembers is still true |
| GI Clamp | 8 | Upper bound on a single sample's contribution |

**Sample Cap** is the responsiveness control. Higher converges smoother and reacts to a
lighting change more slowly.

**Validation Stride** is what lets a light be switched off. A reservoir remembers radiance
that was true when it was traced, and nothing about the *geometry* changes when a light goes
out, so temporal reuse cannot notice on its own. Setting this to zero disables the check, and
a switched-off light then lingers for as long as the history does.

**GI Clamp** bounds a single sample. A reservoir that has collapsed onto one very bright
sample produces a firefly no amount of denoising hides, and a firefly in an indirect term is
far more objectionable than a slightly dimmer bounce.

The bounce carries light from *every* source — point lights, spot lights, emissive surfaces —
without the ray shader knowing the light list, because whatever lit a surface last frame is
already in the previous frame's image. A point the previous frame never showed falls back to
an analytic estimate of the sun and sky.

GI needs the **High-end** tier. On Limited it is switched off and the *Ray tracing* section
reports that it is honouring less than the settings ask for. DDGI still answers for indirect
light there, which is why a scene degrades gracefully rather than going flat.

## Denoising

**Denoise** (on by default) runs temporal accumulation and a variance-guided à-trous filter
over the traced signals. At the ray counts a real-time frame can afford, the raw results are
noise with a shape rather than an image, which is why it defaults on.

One pair of shaders covers all three signals — diffuse GI, specular reflections and shadow
visibility — with different parameters each. Diffuse takes the longest history because an
indirect bounce changes slowly; specular the shortest, because a reflection is view-dependent
and a long history is exactly what smears it across a moving camera; shadows get the tightest
edge handling, because a penumbra is a real gradient and blurring it away reads as "shadows
look wrong" rather than "shadows look noisy".

**Turn it off when tuning ray budgets.** It is the only way to see what the rays actually
produced rather than what the filter recovered.

The denoiser writes its result back over the same image the lighting reads, so switching it
off changes nothing else about the frame.

## Test scenes

| Scene | Exercises |
|---|---|
| `Dev/Tests/assets/scenes/rt_effects_test.Lscene` | Shadows, occlusion and reflections together |
| `Dev/Tests/assets/scenes/rt_cutout_test.Lscene` | Alpha-cutout shadows — a card that shadows as leaves, not a rectangle |
| `Dev/Tests/assets/scenes/restir_gi_test.Lscene` | A red wall bouncing colour onto a white floor |
| `Dev/Tests/assets/scenes/denoise_test.Lscene` | Every traced signal on at once |
| `Dev/Tests/assets/scenes/raybudget_test.Lscene` | Performance modes and the dynamic budget |

## See also

- [Lighting & Shadows](02-lighting.md) — shadow maps, ReSTIR DI, DDGI probe volumes, sky
- [Components](../02-building-worlds/02-components.md) — the Renderer Settings component
- [Renderer diagnostics](../06-scripting/02-lua/api/testing/02-renderer-diagnostics.md) — the globals
  a test drives tiers, ray budgets and every backend's debug view with
- [Troubleshooting](../07-projects-and-tools/05-troubleshooting.md) — when a traced effect looks wrong
