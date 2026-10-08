---
title: "Wall-clock time"
---

The real date and time, for stamping save slots. See [Lua Scripting](../../index.md).

The engine's Lua has no `os` library, so the `game` table has two small calls for the clock.

## `unix_time()`

The current time in whole seconds since 1970-01-01 UTC, as an integer.

## `local_time_text(unix)`

That moment as local time text in the machine's time zone, `"YYYY-MM-DD HH:MM"`, 16 characters. A time the C library cannot convert answers an empty string.

```lua
local now = game.unix_time()
game.save_data("slot1", { saved_at = now, label = game.local_time_text(now) })
```
