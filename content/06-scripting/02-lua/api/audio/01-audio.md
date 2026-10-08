---
title: "Audio"
---

Non-spatial playback and the three mix buses, from Lua. See
[Lua Scripting](../../index.md).

The `audio` table is the part of the API that is not tied to an entity: a UI click or a menu cue
has no emitter to hang a component on, and the bus volumes are what a game's options screen
writes to. The per-entity sound calls (`entity:play_sound`, `entity:stop_sound`) live with the
rest of the entity's methods.

Audio is also the one area where a call that does nothing is ordinary rather than a fault: a
machine with no output device, and any moment outside play mode, make everything here a silent
no-op. Treat these as fire and forget, and never gate gameplay on whether a sound played.

[C++ equivalent](../../../03-cpp/api/audio/01-audio.md)

## `play_2d(clip [, volume [, bus]])`

Plays a clip with no position: it is heard at the same level everywhere, with no distance
attenuation and no panning. That is the whole point of the call, and it is what makes it right
for a UI click, a menu cue, a pickup chime.

It is **fire and forget**: the sound belongs to no entity, there is nothing to stop or move, and
it reclaims itself once it ends. Two calls play two copies at once, so a rapid tap sounds like
taps rather than replacing itself, and the sound outlives whatever played it.

```lua
audio.play_2d("audio/ui/click.wav")                        -- SFX, full volume
audio.play_2d("audio/music/theme.wav", 0.5, "Music")
```

`clip` is project-relative, resolved the same way a script component's path is. `volume`
defaults to `1.0` and `bus` to `"SFX"`. The file is decoded when the call loads it rather than
streamed as it plays, which is the right trade for the short clips this engine targets.

The call answers nothing. It is a silent no-op outside play mode and with no output device,
which is also what an empty clip gets. A clip that cannot be decoded writes a warning naming the
file to the log and plays nothing.

## `set_bus_volume(bus, volume)`

Sets the volume of one mix bus. The three buses are `"Music"`, `"SFX"` and `"UI"`, and every
sound played through the engine is routed through one of them, so this scales that whole group
at once, including sounds already playing.

**Anything unrecognised resolves to SFX**, including a typo and including the word `"Master"`,
which is not a bus. The names are matched exactly and are case-sensitive, so `"music"` is not
recognised either: a settings slider wired to the wrong spelling silently moves the wrong bus.

There is no master volume in the engine. A slider that should scale everything is the game
writing scaled values to the three buses itself.

The call answers nothing, and is a silent no-op when there is no audio engine.

```lua
-- An options screen applying remembered preferences.
audio.set_bus_volume("Music", 0.5)
audio.set_bus_volume("SFX", 0.8)
audio.set_bus_volume("UI", 0.8)
```

## `get_bus_volume(bus)`

The bus's volume as it currently stands.

**Answers `1.0` when there is no audio engine**, which is the neutral value rather than a
failure. A script that reads a bus and writes it back is therefore correct whether or not there
is a device.

The bus name resolves exactly as it does for `set_bus_volume`, so an unrecognised name answers
the SFX volume.

```lua
local music = audio.get_bus_volume("Music")
audio.set_bus_volume("Music", music * 0.5)
```

## A complete example

A footstep the SFX bus controls, and the click that changes the bus.

```lua
local stepTimer = 0

function on_update(dt)
    stepTimer = stepTimer - dt
    if stepTimer > 0 then return end
    stepTimer = 0.4

    local v = entity:get_velocity()
    if v.x * v.x + v.y * v.y > 0.25 then     -- moving faster than half a metre a second
        entity:play_sound()
    end
end

-- The options screen's SFX slider.
function on_sfx_slider_changed(value)
    audio.set_bus_volume("SFX", value)
    audio.play_2d("audio/ui/click.wav", 1, "UI")     -- preview on the UI bus, so the
                                                     -- SFX slider does not change it
end
```

The footstep uses `entity:play_sound`, because a footstep has an emitter, follows one and should
attenuate with distance. The `audio.play_2d` calls are the ones with no entity behind them. For
anything spatial, tracking a moving emitter, or needing to be stopped, use the entity's own
calls: `entity:play_sound` and `entity:stop_sound` are documented with the entity's other
methods.
