---
title: "Logging (log and print)"
---

Four ways for a script to write a line to the log. See [Lua Scripting](../../index.md).

## `log.info(message)` / `log.warn(message)` / `log.error(message)`

```lua
log.info("wave " .. wave .. " started")
log.warn("no navmesh loaded yet, holding position")
log.error("objective entity is gone")
```

Each writes one line, tagged **`[script]`**. That is the same tag a native module's own log lines
get, so a module and a script are indistinguishable in the log except by what they say, which is
worth knowing if you are grepping. If you want to tell them apart, say so in the message.

| | When to reach for it |
|---|---|
| `log.info` | Progress and state, for someone reading the log after the fact |
| `log.warn` | Something was not what the script expected, and it coped |
| `log.error` | Something is wrong. A clean run should not narrate itself at this level |

**Output lands in `LAB/logs/LAB.log`**, flushed per record, so the last line survives a crash
rather than being lost with the buffer. The engine's own core lines go to `LAB/logs/LABEngine.log`
beside it; both files are written while the editor or a build runs.

**The argument is a finished string.** There is no formatting: a number or a `vec3` is not
converted for you, so build the line with `..` first, and `tostring(v)` for anything that is not
already a string (`tostring` of a `vec3` reads `vec3(1.0, 2.0, 3.0)`).

```lua
log.info("speed " .. speed .. " at " .. tostring(entity:get_position()))
```

**Nothing is returned, and the call cannot fail.** A script that logs is always heard; there is no
level filter a script can trip, no capacity to overflow, and no return value to check. What does
cost is volume: a line built and written every frame is a real per-frame cost and a file nobody
can read, so log on state changes and on a throttle rather than every tick.

The same three calls are on the module side, and a module's lines arrive tagged `[script]` exactly
the same way: see [Logging and time](../../../03-cpp/api/module/05-logging-and-time.md) for that half.

## `print(...)`

```lua
print("wave", 3, entity:get_position())
```

Lua's own `print`, replaced. In a packaged build there is no console for it to reach, so
**everything it prints goes to the same log as `log.info`**, at info level and with the same
`[script]` tag.

It takes any number of arguments. Each one is turned into text (a string used as it is, anything
else through Lua's `tostring`) and the pieces are joined with a **tab**, so `print(a, b)` puts a
tab between the two rather than a space. It returns nothing and cannot fail.

```lua
print("wave", wave, "spawned")
```

With `wave` set to `3`, that writes a line whose message is `[script] wave` then a tab, `3`, then
another tab and `spawned`.

Two things to prefer over it: `log.warn` and `log.error` when the line is a warning or a problem
(`print` is always info), and `..` with a single string when you want the line spaced the way you
typed it rather than tab separated.

## Where the line ends up

A log record is written as a timestamp, a level and the logger name, then the message:

```
[14:03:12] [info] LAB: [script] wave 3 started
```

So `log.error("objective entity is gone")` arrives as an `[error]` line with the same `[script]`
message tag, which is what to grep for when several systems are logging to the same file.

A script's own failures land in the same place: an error raised in `on_update` is written with its
Lua message and the script is disabled from then on (see
[Lua Scripting](../../index.md)), so a script that stops running always says why in
`LAB/logs/LAB.log`.
