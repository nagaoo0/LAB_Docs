---
title: "Text language"
---

Which language HUD text is in, for the fonts that draw it. See
[Lua Scripting](../../index.md), and [HUD → Languages](../../../../05-ui/01-hud.md#languages)
for the manifest side.

A project's HUD font can have **fallback fonts** for the characters it does not have, and the
manifest can give a language its own order of them (`UIFontFallbacksByLanguage`) and its own extra
characters to bake (`UIFontCharsetByLanguage`). That is what the language is for: Chinese and
Japanese share most of their ideographs as the same Unicode codepoints, drawn differently in each
language, so the font that supplies 直 or 骨 has to be the Japanese one for Japanese text and the
Chinese one for Chinese text. The engine cannot tell the two apart from the text; the game says
which one it is showing.

## `ui.set_text_language(language)`

```lua
ui.set_text_language("ja")
ui.set_text_language("zh-Hans")
ui.set_text_language(nil)          -- back to the project-wide fallback order
```

`language` is the game's own code for it, matched against the keys the manifest declares under
`UIFontFallbacksByLanguage` and `UIFontCharsetByLanguage` ignoring case (with `_` read as `-`), the
whole tag first and then with trailing subtags removed, so `"zh-Hans-CN"` finds `zh-Hans`, then
`zh`, and `"ja-JP"` finds `ja`. A language the manifest does not mention simply uses the project-wide
order, which is also what `nil` or `""` selects.

**It takes effect at once.** The next thing that lays text out, a `measure_text` straight after
this call or the next frame's HUD, finds the font rebaked for the new language if that changes
which fonts or characters it needs; the atlas reaches the GPU at the top of the next frame, before
anything is drawn with it. A switch between two languages whose fonts and characters are the same
bakes nothing. Rebaking a few thousand ideographs takes about half a second on a desktop and
roughly a second on a Steam Deck (the table in [HUD → Languages](../../../../05-ui/01-hud.md#languages)),
so this belongs where a player changes a setting, or once at startup, before the first text is
measured: a game that measures its HUD first and sets the language after pays for two bakes.

The language is **one setting for the whole process**, not per scene: it survives a level
change, and in the editor it outlives a Play session, so the Scene view keeps showing the HUD in
the last language a script chose.

## `ui.text_language()`

The language last set, or `""`.

## Choosing the language

What the game supports and what the player picked is the game's business; the engine only
supplies the inputs. A usual first-launch order is the player's saved choice, then the language
they picked in Steam ([`steam_apps.game_language()`](../steam/02-steam-apps.md), Steam's own
names such as `"schinese"`, mapped to the game's codes), then the game's source language.
