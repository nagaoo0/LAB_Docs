---
title: "The window"
---

Leaving the game, the fullscreen setting, and the system cursor. See
[Lua Scripting](../../index.md).

The last few calls on the `game` table are about the application window rather than the game
world. Most of them are requests and questions: a script asks, and whoever is driving the scene
(a standalone build or the editor) decides what the request means. That split is deliberate,
because a game's Quit button and its settings menu must not be able to close or fullscreen the
editor they are being tested in.

## `quit()`

Asks to leave the game. Nothing happens inside the call: the request is recorded and read after
the frame's update, once the scripts have finished, never in the middle of a callback.

What leaving means is the host's decision. A standalone build closes. In the editor it stops
Play mode and leaves the editor running, exactly as pressing Stop would.

The call answers nothing, and calling it again is harmless.

```lua
function on_update(dt)
    if input.is_key_pressed("Escape") then
        game.quit()
    end
end
```

## `is_fullscreen()`

`true` while the application window is fullscreen, `false` otherwise.

It answers the window's live state, so in the editor it reports the editor's own window, which
F11 toggles there. It does not report what a script last asked for through `set_fullscreen`.

```lua
-- A settings checkbox that starts on the truth.
local fullscreen = game.is_fullscreen()
```

## `set_fullscreen(on)`

Asks for the window to be fullscreen (`true`) or windowed (`false`), applied after the frame's
update. The request takes effect only when it differs from the state the window is already in,
so asking for what is already true does nothing.

**A standalone build's settings menu writes this. The editor consumes the request and ignores
it**, so a game cannot fullscreen the editor. In the editor the call is a silent no-op, and it
answers nothing anywhere.

```lua
game.set_fullscreen(true)
```

## `set_system_cursor_visible(visible)`

Hides the system pointer (`false`) or shows it again (`true`), for a game that draws a pointer
of its own. It is a state rather than a one-shot: the cursor stays hidden until it is shown
again.

**A standalone build only.** The editor keeps its cursor so its panels stay usable during Play,
and this call is a silent no-op there. It answers nothing.

It is not the same thing as capturing the mouse. Hiding only stops the system pointer being
drawn: nothing is locked and look input is unaffected, so a hidden cursor can still walk off the
window. For a first-person camera, where the point is that the pointer cannot leave and keeps
reporting movement, use `input.set_mouse_captured(true)` instead. That hides and locks at once,
and releases on Escape or on losing focus.

```lua
-- A game that plays its own HUD pointer hides the system one while Play runs.
game.set_system_cursor_visible(false)
```

## What is not here

There is no getter for the cursor, and no call for the window's size or position. The four calls
on this page are the whole of a game script's window surface; the editor's own viewport controls
are driven from its UI, or from a test through `editor.*`.
