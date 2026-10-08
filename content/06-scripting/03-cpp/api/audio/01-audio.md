---
title: "Audio"
---

Sounds and the bus volumes a game's options screen writes to.

Audio is the one area where a call that does nothing is ordinary rather than a fault: a machine
with no output device, or any moment outside play mode, makes every one of these a silent no-op.
Treat them as fire and forget, and never gate gameplay on whether a sound played.

## `PlaySound` / `StopSound`

```c
void (*PlaySound)(LAB_EntityId entity, const char* clip);
void (*StopSound)(LAB_EntityId entity);
```

```cpp
void LABNative::PlaySound(LAB_EntityId entity, const char* clip = nullptr);
void LABNative::StopSound(LAB_EntityId entity);
```

Plays the entity's own `AudioSourceComponent`, or `clip` when one is given.

**A null or an empty clip mean the same thing**: the component's own clip. A node graph's
unconnected string pin and a Lua call that omitted the argument have to agree, so both arrive
here as empty rather than as a distinct case, and the fallback is the component's.

```cpp
LABNative::PlaySound(GetEntity());                    // whatever the component names
LABNative::PlaySound(GetEntity(), "audio/hit.wav");   // this clip instead
LABNative::StopSound(GetEntity());
```

The play is **spatial**: it comes from the entity's world position through the same audio engine
a scene's `PlayOnStart` uses, so an entity moving away gets quieter and pans. There is no
"play this at the listener" call on the ABI, unlike the Lua facade's `audio.play_2d`. A sound
that has to outlive its emitter is the thing that call exists for, and a module does not have it
yet.

**A destroyed entity's sound is stopped.** The engine tracks at most one sound per entity, keyed
by the entity handle, and destroys that entry with the entity. Leaving one playing would let it
resurface against whatever new entity happens to be created at the same recycled handle next,
which is a wrong-position bug that looks like nothing in the code.

## `SetSoundParams` / `IsPlaying` (ABI minor 13)

```c
typedef struct LAB_AudioAPI
{
	uint32_t StructSize;
	void (*SetSoundParams)(LAB_EntityId entity, float volume, float pitch);
	int (*IsPlaying)(LAB_EntityId entity);
} LAB_AudioAPI;
```

```cpp
void LABNative::Audio::SetSoundParams(LAB_EntityId entity, float volume, float pitch);
bool LABNative::Audio::IsPlaying(LAB_EntityId entity);
```

`SetSoundParams` changes the volume and pitch of the sound the entity is playing, **while it
plays**: an engine note that rises with speed, a rope hum that swells with tension. Call it every
frame for a voice that follows a value; it does not restart the sound. The volume becomes the
sound's new base level, the one raytraced-audio muffling and ambience scale from, and a negative
volume reads as silence and a pitch has a small floor so it can never reach zero. A no-op for an
entity with nothing playing. Lua's `entity:set_sound_params(volume, pitch)` is the same call, and
`entity:get_sound_params()` reads the pair back (a table with `volume` and `pitch`, or nil with
nothing playing).

`IsPlaying` answers whether the entity has a sound the audio engine still holds: playing now, or a
non-looping one that ended since the last frame, which the engine reaps once a frame. With no
audio device nothing ever plays, so it is false, which is why a game must not gate logic on it
beyond starting a voice again.

```cpp
void OnUpdate(float) override
{
	if (!LABNative::Audio::IsPlaying(m_HumVoice))
		LABNative::PlaySound(m_HumVoice);
	LABNative::Audio::SetSoundParams(m_HumVoice, 0.2f + 0.8f * m_Tension, 0.8f + 0.6f * m_Tension);
}
```

### Clips are decoded once

The first time a clip plays, the engine decodes it and keeps the decoded data for the life of the
play session. `PlaySound` stops an entity's current voice before starting its next, so before
this a clip retriggered from one entity (a bumper hit, a footstep) was decoded from disk again on
the main thread every time. The second play is now a lookup. A clip file edited while the game
runs is not reloaded until play stops.

## `SetBusVolume` / `GetBusVolume`

```c
void (*SetBusVolume)(const char* bus, float volume);
float (*GetBusVolume)(const char* bus);
```

```cpp
void LABNative::SetBusVolume(const char* bus, float volume);
float LABNative::GetBusVolume(const char* bus);
```

The three buses, and their names:

| Bus | Name | |
|---|---|---|
| Music | `"Music"` | Background music |
| SFX | `"SFX"` | Everything else |
| UI | `"UI"` | Interface sounds |

**Anything unrecognised becomes SFX.** The lookup is a plain string comparison against `Music`
and `UI`, and every other input, including a typo, falls through to SFX. That is the same
tolerance a mistyped key name gets, and it has the same cost: a settings screen with a typo'd
bus name silently adjusts the wrong bus rather than failing.

Note there is no `"Master"`. The three above are all of them, so a "master volume" slider is
something a game implements itself, by scaling what it writes to the three.

```cpp
void OnCreate() override
{
	// A remembered preference, or a default.
	const float music = LABNative::GetBusVolume("Music");
	LABNative::SetBusVolume("Music", music * 0.5f);
}
```

`GetBusVolume` answers **1.0 with no audio engine**, which is the neutral value rather than a
failure. A behaviour that reads a bus and writes it back is therefore correct whether or not
there is a device.

## A complete example

A footstep that fires on the physics clock, at a volume the game's options screen controls.

```cpp
void OnFixedUpdate(float fixedDeltaTime) override
{
	m_StepTimer -= fixedDeltaTime;
	if (m_StepTimer > 0.0f)
		return;

	m_StepTimer = 0.4f;

	if (const LAB_Vec3 v = GetVelocity();
		(v.x * v.x + v.y * v.y) > 0.25f)     // moving faster than half a metre a second
		LABNative::PlaySound(GetEntity());
}
```
