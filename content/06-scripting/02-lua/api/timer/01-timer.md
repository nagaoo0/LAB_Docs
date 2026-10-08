---
title: "timer"
---

Run a function later, or on an interval, without counting frames in `on_update`.

```lua
timer.after(2.0, function() log.info("two seconds later") end)

local id = timer.every(0.5, function() spawn_spark() end)
timer.cancel(id)
```

Time is game time, the same `dt` your `on_update` gets. A timer made by a script belongs to that
script's entity: when the entity is destroyed, or the script is reloaded, its timers are dropped
with it. Everything is dropped when the game stops.

## `timer.after(seconds, fn)`

Calls `fn()` once, on the first frame at least `seconds` have gone by, and answers an id. The
timer is gone once it has run. `timer.after(0, fn)` runs on the next frame, not inside the call.

## `timer.every(seconds, fn)`

Calls `fn()` over and over and answers an id. It runs at most once a frame, so an interval
shorter than a frame runs every frame and does not catch up on the ones it missed.

## `timer.cancel(id)`

True when the timer was still pending, false for an id that already ran, was cancelled or never
existed. It is safe to cancel a timer from inside its own callback.

## Errors

A callback that throws is logged, counted as a script failure, and its timer is removed so it
doesn't repeat the error every frame. The script itself keeps running.

A timer created from `on_create` or later fires from the frame after it is made. A timer created
while timers are firing waits for the next frame.
