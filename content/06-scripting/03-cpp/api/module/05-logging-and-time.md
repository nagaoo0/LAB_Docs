---
title: "Logging and time"
---

The three log calls, and the engine's own clocks.

## Logging

```c
void (*LogInfo)(const char* message);
void (*LogWarn)(const char* message);
void (*LogError)(const char* message);
```

```cpp
void LABNative::LogInfo(const char* message);
void LABNative::LogWarn(const char* message);
void LABNative::LogError(const char* message);
```

```cpp
LABNative::LogInfo("Mover: on_create");
LABNative::LogWarn("the object path did not resolve");
LABNative::LogError("the module's rigid body struct is from an older header");
```

A module's line goes through the same path a Lua script's `log.info` does, and arrives **tagged
`[script]`**, the same as a script's. So a module and a script are indistinguishable in the log
except by what they say, which is fine in practice and worth naming if you are grepping: tag your
own messages with the behaviour's name if you want to tell them apart.

Output goes to `LAB/logs/LAB.log` and `LAB/logs/LABEngine.log`, both flushed per record, so a
crash leaves the last line rather than losing it.

### There is no formatting

These take a finished string. There is no `printf` and no fmt, because the ABI is plain C and a
variadic call across a shared-library boundary is a compiler-specific contract.

Build the string yourself:

```cpp
char message[128];
std::snprintf(message, sizeof(message), "Mover: reached (%.1f, %.1f, %.1f)",
	position.x, position.y, position.z);
LABNative::LogInfo(message);
```

Or with the C++ the sugar layer allows:

```cpp
LABNative::LogInfo(("spawned " + std::to_string(count) + " objects").c_str());
```

```cpp
// Fine when the log line is only evaluated at a rate you control.
if (m_Debug && (m_Tick % 60) == 0)
	LABNative::LogInfo(("state " + std::to_string(m_State)).c_str());
```

A null message is logged as an empty line rather than crashing, but building one costs an
allocation, so a per-frame log line in a hot loop is a real cost. Log on state changes and on a
throttle, not every frame.

### Which level

| | |
|---|---|
| `LogInfo` | Progress and state, for someone reading the log after the fact |
| `LogWarn` | Something was not what the code expected, and it coped. A missing object path, a stale struct |
| `LogError` | Something is wrong. The counterpart to the engine's own error level |

A clean run that narrates itself at error level is noise in the one place anyone looks for a real
failure, which is why the three exist separately rather than one `Log`.

## Time

```c
double  (*ElapsedTime)(void);
uint64_t(*FrameIndex)(void);
float   (*FixedDeltaTime)(void);
```

```cpp
double   LABNative::ElapsedTime();
uint64_t LABNative::FrameIndex();
float    LABNative::FixedDeltaTime();
```

All three are the **engine's own counters**, so a behaviour comparing its own timing against the
log is comparing the same numbers.

| | |
|---|---|
| `ElapsedTime` | Wall-clock seconds since the scene started running |
| `FrameIndex` | How many frames it has rendered |
| `FixedDeltaTime` | The timestep `OnFixedUpdate` is called with |

```cpp
// A one-shot after five seconds of play.
if (!m_Fired && LABNative::ElapsedTime() > 5.0)
{
	m_Fired = true;
	SpawnWave();
}
```

`FrameIndex` is the call for anything that should happen every N frames, and it is exact where
counting your own `OnUpdate` calls is only as exact as the callback being entered every frame.

### Read `FixedDeltaTime`, do not assume 1/60

`FixedDeltaTime` is the engine's physics step, and it is the value `OnFixedUpdate` receives.
A behaviour integrating anything over its own idea of the step drifts against the bodies it is
steering, because the two are then running on different clocks that only look alike.

```cpp
void OnFixedUpdate(float fixedDeltaTime) override
{
	// The parameter is the same number, so use whichever reads better in the code.
	m_Charge += fixedDeltaTime;
}
```

There is **no fixed-rate tick that runs without physics**. `OnFixedUpdate` fires once per physics
step the solver actually took, and a scene with no rigid bodies takes no steps at all. A
behaviour wanting a steady tick independent of the simulation keeps its own accumulator in
`OnUpdate`:

```cpp
void OnUpdate(float deltaTime) override
{
	m_Accumulator += deltaTime;
	while (m_Accumulator >= 0.1f)
	{
		m_Accumulator -= 0.1f;
		Tick();          // ten times a second, whatever the frame rate is doing
	}
}
```

That is the honest way to do it, and its own cost is real: a long frame produces several ticks at
once, so anything expensive belongs behind a check rather than in the loop.

## A complete example

A behaviour that reports what it is doing, on a throttle, with the engine's own clock.

```cpp
void OnCreate() override
{
	char message[96];
	std::snprintf(message, sizeof(message), "%s: ready at frame %llu",
		GetName().c_str(), static_cast<unsigned long long>(LABNative::FrameIndex()));
	LABNative::LogInfo(message);
}

void OnUpdate(float deltaTime) override
{
	m_Timer += deltaTime;
	if (m_Timer < 1.0f)
		return;
	m_Timer = 0.0f;

	char message[96];
	std::snprintf(message, sizeof(message), "%s: t=%.1fs position (%.1f, %.1f, %.1f)",
		GetName().c_str(), LABNative::ElapsedTime(),
		GetWorldPosition().x, GetWorldPosition().y, GetWorldPosition().z);
	LABNative::LogInfo(message);
}
```

Note the throttle: `m_Timer` gates the formatting and the log write to once a second, so the
message is readable in the log and the cost is nil. An unthrottled version of the same thing at
60 fps is a file that nobody can read and a real per-frame cost.
