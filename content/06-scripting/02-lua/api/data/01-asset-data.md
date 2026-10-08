---
title: "Reading data files (data)"
---

Two calls that read a project text file into Lua, for a script that has no filesystem of its own.
See [Lua Scripting](../../index.md).

A game's content (item tables, level lists, dialogue lines) usually belongs in a data file rather
than in a script, and a script is deliberately given no `io` and no `os`. `data` is the seam:
both calls take an asset-relative path, the same kind of path a `ScriptComponent` or a
`scene.spawn` call takes, and read through the virtual filesystem, so a file inside a mounted
`.Lpak` resolves exactly like one on disk. With a project open the path is resolved under the
project's asset directory; without one it resolves against the working directory, the way the
rest of the engine's asset references do.

## `data.load(path)`

```lua
local items = data.load("data/items.yaml")
print(items[1].name .. ": " .. items[1].damage)
```

Reads one **YAML or JSON** file (YAML reads both) and returns it as plain Lua tables.

| In the file | In Lua |
|---|---|
| a map | a table keyed by string |
| a sequence | a table indexed from 1, so `#items` works |
| `true` / `false` | a boolean |
| an unquoted number | a number: an **integer** when written without a `.` or exponent (`1` reads as `1`), a float otherwise (`1.0` reads as `1.0`) |
| anything else | a string |

**Quoting wins.** A quoted scalar is a string whatever it looks like: `"1"` reads as the string
`"1"` rather than the number `1`, which is what keeps a version number or a code with a leading
zero intact.

```yaml
name: rusty sword
damage: 12
stackable: false
tags: [melee, starter]
```

```lua
local item = data.load("data/items.yaml")
log.info(item.name)
log.info(item.damage + 1)
if not item.stackable then ... end
for i = 1, #item.tags do log.info(item.tags[i]) end
```

Read against the file above, those lines log `rusty sword` and `13` (a number, not text),
`stackable` is a boolean false, and `tags` is a two-entry sequence.

**It answers `nil` when it cannot deliver a table.** Two of the cases write a line to the log
first, and one is quiet:

- the file is missing or empty: `data.load: <path> is missing or empty`
- the text does not parse as YAML: `data.load: <path> does not parse: <reason>`
- a document that parses but holds nothing (a null node, an empty document) also answers `nil`,
  quietly

A deeply nested document is truncated rather than recursed forever: past 64 levels, a nested map
or sequence arrives as `nil` from that depth down.

So `if items then` is the whole check, and a `nil` answer is worth a look in the log rather than
something to fall back on silently.

The tables are built fresh for the call, so what a script gets is its own copy: editing `items`
changes nothing for anyone else, and the file on disk is never written.

## `data.load_text(path)`

```lua
local csv = data.load_text("data/balance.csv")
```

The whole file as one string, with no parsing at all, or **`nil`** when the file is missing or
empty (the same empty check `data.load` makes, without the error line).

This is the call for anything that is not a data table: a `.csv` a script splits itself, a text
asset, a file whose format the engine has no reader for. A UTF-8 byte order mark at the start of
the file is stripped by both calls, so a file saved by an editor that writes one reads the same as
any other.

```lua
local text = data.load_text("data/level_list.txt")
if text then
    for line in text:gmatch("[^\r\n]+") do
        scene.preload_scene(line)
    end
end
```

## Which one

`data.load` for anything with structure, and `data.load_text` for anything without. A single
scalar in a file is neither: `data.load("data/version.txt")` holding `1.2.3` comes back as the
string `"1.2.3"`, so a file with one value in it does not need a table shape to be read.
