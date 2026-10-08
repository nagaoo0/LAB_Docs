---
title: "Sound"
---

Playing, stopping and adjusting a sound an entity is making. See [Lua Scripting](../../index.md) for how a script runs and when its callbacks fire.

Audio is the one area where a call that does nothing is ordinary rather than a fault: a machine with no output device, or any moment before play has started, makes every one of these a silent no-op. Treat them as fire and forget, and never gate gameplay on whether a sound played.

[C++ equivalent](../../../03-cpp/api/audio/01-audio.md), which covers the same calls plus the bus volumes.

## `play_sound([clip])`

```lua
entity:play_sound()
entity:play_sound("audio/hit.wav")
```

Plays this entity's own `AudioSourceComponent`: volume, pitch, loop flag, spatial parameters and bus all come from it. A `clip` argument overrides just the path, so a script can vary what an entity says without a component for every line.

**No argument and an empty string are the same thing**: both mean the component's own clip. So a script passing a variable that happens to be empty falls back the same way as one that passed nothing, which matters because a node graph's unconnected clip pin arrives here as an empty string too, and both have to mean "the component's clip" rather than "play nothing".

The clip path is project-relative, and what it names is a raw `.wav`, `.ogg`, `.mp3` or `.flac`: there is no cooked audio form yet, so it is a Source Browser file rather than an Asset Browser one.

**It works with no `AudioSourceComponent` at all.** The play then uses what the component would have defaulted to: the SFX bus, non-looping, spatial, at the default volume, pitch and distance range, positioned at the entity's world position. So a script can make any entity speak once without the Properties panel being involved.

The play is **spatial**: it is heard at the entity's world position, so it gets quieter and pans as the entity moves away from the listener.

**One sound per entity.** A second `play_sound` replaces the first rather than layering a second instance of it, and a sound in flight is cut off the moment the next one starts. Two clips at once from one entity is what `audio.play_2d` is for, along with any sound that has to outlive the entity it came from.

The call is silent, and answers nothing, when there is nothing it can play: an entity that does not exist, no audio engine (outside play mode, or a machine with no output device), no clip on either the call or the component, or a file that fails to load. A load failure leaves a warning in `LAB.log` naming the path it tried.

```lua
function on_collision_begin(other)
    if other:get_name() == "Bullet" then
        entity:play_sound("audio/impact.wav")
    end
end
```

## `stop_sound()`

Stops the sound this entity is playing, wherever it is in the clip. No-op with nothing playing, and no-op on an entity that does not exist.

## `set_sound_params(volume, pitch)`

```lua
entity:set_sound_params(0.4, 0.8 + 0.6 * speed)
```

The volume and pitch of the sound this entity is already playing, changed while it plays: an engine that revs, a hum that swells, a radio tuning past a station. No-op with nothing playing.

Volume is clamped at 0 or more, and pitch at 0.01 or more, so neither can be driven negative into silence or into a stalled playback by a badly scaled formula.

This changes the playing sound, not the component, so the next `play_sound` starts from the component's own `Volume` and `Pitch` again. A script driving both this and the Properties panel values is fighting itself; own one of them.

## `get_sound_params()`

```lua
local params = entity:get_sound_params()
if params then print(params.volume, params.pitch) end
```

The volume and pitch the entity's playing sound has now, as a table with `volume` and `pitch`, or `nil` with nothing playing (which is also what a machine with no audio device answers, since it plays nothing). The volume is the base level `set_sound_params` last set, before raytraced-audio muffling scales it.

## `get_muffle_ratio()`

```lua
local clarity = entity:get_muffle_ratio()
```

The smoothed line-of-sight fraction for this entity's own playing sound, from the raytraced audio system: the share of the acoustics rays between the source and the listener that found a path, smoothed over time so a single blocked frame does not click. `1` is fully audible, lower is more obstructed, and the ratio is what drives the low-pass and the loudness the sound is heard with.

It answers `1.0` with nothing playing, with the acoustics system not running, and for an entity that does not exist, which is the same value a fully clear line of sight gives.

This is a diagnostic and a tuning reader, for a test or a debug overlay that wants to know whether the ray tracing is doing anything. A game rarely needs it, because the effect it measures is already applied to the sound itself.
