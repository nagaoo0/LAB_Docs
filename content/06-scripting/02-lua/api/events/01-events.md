---
title: "events"
---

A small bus for scripts that don't hold a reference to each other.

```lua
-- a pickup script
events.emit("coin_collected", 5)

-- the HUD script
local handle = events.on("coin_collected", function(amount)
    score = score + amount
end)
events.off(handle)
```

Names are plain strings, matched exactly. Calls are synchronous: `emit` returns after every
handler has run.

## `events.on(name, fn)`

Registers `fn` for `name` and answers a handle for `events.off`. A script can register as many
handlers as it likes, for the same name too.

## `events.off(handle)`

True when the handler was registered, false when it was already removed. A handler may remove
itself, or another one, while an emit is running.

## `events.emit(name, ...)`

Calls every handler of `name` with the extra arguments, in the order they were registered, and
answers how many were called (0 when nobody listens).

- A handler added during an emit does not run in that emit.
- A handler removed during an emit, before its turn, does not run.
- A handler may emit again. That runs the inner emit to the end first.

## Lifetime and errors

A handler registered by a script belongs to that script's entity and is removed when the entity
is destroyed or the script reloads. All handlers go when the game stops.

A handler that throws is logged and counted as a script failure, and is removed. The other
handlers of the emit still run.
