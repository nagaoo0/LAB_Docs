---
title: "Profiling"
---

Timing a span inside a callback, and seeing it beside the engine's own passes.

## `ZoneAdd`

```c
void (*ZoneAdd)(const char* name, double milliseconds);
```

```cpp
void LABNative::ZoneAdd(const char* name, double milliseconds);
```

Reports a span the module timed itself, under a name the module chose. It appears in
`cpu_zones()` and in Tracy beside the engine's own passes, at the same level, so a module's work
is visible in exactly the place the renderer's is.

**The engine cannot time inside a callback.** It knows when the callback started and stopped, but
nothing about what happened in between, because that is the module's own code. So the module
measures and this publishes. A module is the only thing that can say where its own interesting
spans are.

```cpp
const auto start = std::chrono::steady_clock::now();
SolveContacts();
const double ms = std::chrono::duration<double, std::milli>(
	std::chrono::steady_clock::now() - start).count();
LABNative::ZoneAdd("Flock.SolveContacts", ms);
```

## `ZoneScope`

The one-line form of it, and the one to reach for.

```cpp
void OnUpdate(float deltaTime) override
{
	LABNative::ZoneScope zone("Mover.Solve");
	// everything in this scope is timed, and reported when it ends
}
```

```cpp
class ZoneScope
{
public:
	explicit ZoneScope(const char* name);
	// not copyable
	~ZoneScope();     // reports the elapsed milliseconds under `name`
};
```

It measures with a steady clock and reports on destruction, so it works through an early return
and through an exception without any cleanup of its own. The name is a `const char*` the module
owns and must outlive the call.

It asks the engine table whether `ZoneAdd` is there before calling it, so a module built against
a newer header loaded by an older engine reports nothing rather than reading past the end of the
table it was handed.

## Cost, and where not to use it

**Registering a name is not free.** A zone registration takes a lock and compares against every
zone that already exists, because the engine is matching this up against the zones it knows about.
That is a fixed cost per call, small, and completely different in kind from the span itself.

So: use it for **spans**, never for a per-entity loop.

```cpp
void OnUpdate(float deltaTime) override
{
	LABNative::ZoneScope zone("Flock.Update");   // one per frame: right

	for (const auto& boid : m_Boids)
	{
		// LABNative::ZoneScope inner("Flock.Boid");  // one per boid: wrong
		UpdateBoid(boid, deltaTime);
	}
}
```

A per-entity zone measures the profiler more than the code: with a thousand boids that is a
thousand lock-and-compare operations per frame, and the number it reports is dominated by the
cost of reporting it.

If you need to know which *part* of a large loop is slow, time it into an accumulator over the
frame and report that once at the end:

```cpp
void OnUpdate(float deltaTime) override
{
	LABNative::ZoneScope zone("Flock.Update");

	const auto start = std::chrono::steady_clock::now();
	for (const auto& boid : m_Boids)
		UpdateBoid(boid, deltaTime);
	const double loopMs = std::chrono::duration<double, std::milli>(
		std::chrono::steady_clock::now() - start).count();

	LABNative::ZoneAdd("Flock.Neighbours", m_NeighbourMs);
	m_NeighbourMs = 0.0;
}
```

## The zone the engine already opens

A per-class zone is opened by the engine around the whole callback, so a module gets a
`<BehaviourName>.OnUpdate` figure without doing anything. That is the one to look at first: if it
dominates a frame, the interesting question is which span inside it does, and that is what
`ZoneScope` answers.

The two together give the shape of a module's frame:

```
Mover.OnUpdate          0.42 ms
  Mover.Solve           0.31 ms
Flock.Update            2.10 ms
  Flock.Neighbours      1.77 ms
```

## Reading the numbers

| Where | What |
|---|---|
| `cpu_zones()` from Lua, in play | Every zone's time for the last frame, including a module's |
| The Performance panel | The same figures beside the engine's own, with a graph over time |
| Tracy | A plot per zone, if the build has Tracy on |

`cpu_zones()` is what a test asserts with, which is how "the behaviour ran and did its work" is
checked without a breakpoint. That matters more for a module than for Lua: the behaviour's own
effects are visible either way, but a zone proves the callback was entered and finished.

Remember that a zone's cost includes whatever the engine does on the way in and out, and that a
first call in a frame can be slower for reasons that have nothing to do with the code inside it.
Compare zones within one run rather than across two, and turn VSync off before believing any
frame figure.

## When a zone does not appear

Two causes, and both are quiet.

- **The module was built against a header with no `ZoneAdd`**, or is being loaded by an engine
  that does not fill it. Both read as absent rather than failing, which is why `ZoneScope` checks
  before it calls. A zone that never appears on an older engine is the version check working.
- **The name is being registered fresh every frame.** The engine matches by name, so a name that
  changes per call (a counter appended to it, an entity id) creates a new zone every time rather
  than accumulating into one. A name should be a constant.
