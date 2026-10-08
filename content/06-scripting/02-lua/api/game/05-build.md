---
title: "The build"
---

Whether this is the demo, and what a demo or non-Steam build changes. See
[Lua Scripting](../../index.md).

Build Standalone has two switches that describe what the build is: **Demo build** and
**Steam build** ([Projects and builds](../../../../07-projects-and-tools/01-projects-and-builds.md#build-options-demo-and-steam)).
This page is how a script sees them.

## `is_demo()`

`true` when the game is the demo, `false` otherwise. Always a boolean.

In a packaged game it is the build's **Demo build** setting, written into the shipped `Game.lab`
when the build was made. In the editor's Play mode it is the per-user **Editor Settings → Demo
build** preference ("Run the project as its demo"), so a developer flips between the demo and the
full game without rebuilding. A run of `LABRuntime` straight against a project folder reads the
project's own setting.

```lua
if game.is_demo() then
    -- the demo's chapters, the wishlist button
end
```

This is the call a game should branch on. It means the same thing everywhere.

## `editor_demo_mode()`

The editor preference itself: a boolean in the editor, `nil` in a packaged game and in a runtime
test. It keeps that meaning on purpose. Games use the `nil` to tell the editor from a build
(`if game.editor_demo_mode() ~= nil then -- running in the editor`), and a script that does would
change behaviour in a demo build if it started answering `false` or `true` there. Use
`is_demo()` for "is this the demo", and `editor_demo_mode()` only to ask "am I in the editor".

## Steam off

A build with **Steam build** off never loads the Steamworks library and never initialises Steam,
so `steam_apps.available()` is `false`, the language calls answer `nil`, `steam_lobby` and
`steam_input` are inert, and `net.host_steam` answers that Steam is not running: the same answers
as a machine with no Steam client. A script needs no check for the build kind. See
[Steam apps](../steam/02-steam-apps.md).
