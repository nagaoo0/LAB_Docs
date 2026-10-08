---
title: "Scripting"
---

LAB has three ways to write gameplay, and all three can run in the same scene at once.

| | Use it for |
|---|---|
| [Visual Scripting](01-visual-scripting.md) | Node graphs, no code. They compile to Lua, so you can read what they do. |
| [Lua Scripting](02-lua/index.md) | Most gameplay. Save the file and replay, no rebuild. |
| [C++ Scripting](03-cpp/index.md) | Systems that need speed, a debugger or third-party libraries. |

The usual split is C++ modules for the systems (movement, combat, AI) and Lua for the
one-offs. [Lua or C++?](03-cpp/index.md#lua-or-c) compares them side by side.

## API reference

- [Lua API Reference](02-lua/api/index.md) — a page for every call, by category
- [C++ API Reference](03-cpp/api/index.md) — the native module API, page by page
