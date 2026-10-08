---
title: "Steam apps (steam_apps)"
---

Three calls that say which language Steam runs the game in, all of them inert wherever Steam is
not there. See [Lua Scripting](../../index.md).

A player picks a game's language in Steam (the game's **Properties → Language**), and Steam
downloads that language's depots and restarts the game in it. The game reads the choice here,
usually once, on first launch, to preselect its own text language before the player has opened a
settings screen:

```lua
-- Steam's name -> the game's own language code.
local FROM_STEAM = { english = "en", french = "fr", russian = "ru", japanese = "ja",
                     schinese = "zh-Hans", german = "de", latam = "es-419", brazilian = "pt-BR" }

local saved = game.get_state("language", "")
local language = saved ~= "" and saved or FROM_STEAM[steam_apps.game_language() or ""] or "en"
ui.set_text_language(language)
```

## Steam is a capability, never a requirement

The same rule as [Steam Input](01-steam-input.md): the Steam library is resolved at run time,
failure to initialise is logged at info, and **nothing here fails when it is missing**. Every
call answers `nil` (or `false` for `available`), so the `or` chain above needs no guard. That is
the normal case in the editor and in every automated run, where Steam is not initialised at all.

A packaged game built with **Steam build** off is in the same state on every machine: it never
loads the Steam library and never starts Steam, even where a Steam client is running. See
[The build](../game/05-build.md).

Both languages are read once, when Steam initialises. Steam restarts a game to change its
language, so neither answer can change under a running session, and asking every frame costs
nothing.

## `steam_apps.available()`

`true` when Steam initialised for this process.

## `steam_apps.game_language()`

`ISteamApps::GetCurrentGameLanguage`: the language the player chose for this game in Steam, or
Steam's own language when they never chose one. `nil` without Steam.

The answer is one of **Steam's API language names**, not an ISO code: `english`, `french`,
`german`, `italian`, `japanese`, `koreana`, `polish`, `brazilian` (Portuguese, Brazil),
`portuguese`, `russian`, `schinese` (Simplified Chinese), `tchinese` (Traditional Chinese),
`spanish` (Spain), `latam` (Spanish, Latin America), `turkish`, `ukrainian` and so on. Map it to
whatever codes the game's own translation files use; the engine does not.

## `steam_apps.ui_language()`

`ISteamUtils::GetSteamUILanguage`: the language the Steam client itself is displayed in, with
the same names as above. `nil` without Steam. Valve's own advice is to use `game_language` for
anything the player reads in the game; this is for the rare case that wants the client's.

## The buy-the-game card

For a project that sets `Steam: RequireOwnership` (see
[Steam integration](../../../../STEAM.md#requiring-ownership)). A game that wants the card in its
own style draws it itself:

```lua
steam_apps.buy_overlay_claim()                       -- once, when the script loads
-- every frame:
local due, reason = steam_apps.buy_overlay_due()     -- true from five minutes in, once the check failed
if due then
    -- reason is "not_on_steam" (started outside Steam, which an owner may have done) or "not_owned"
    -- draw the card; on a button:
    steam_apps.open_store_page()                     -- the game's Steam store page
    steam_apps.open_url("https://example.com/")      -- Steam: WebsiteUrl, or any address the game's own list allows
end
```

`buy_overlay_claim` turns off the engine's plain fallback card; a script that never reaches it
leaves the fallback in place. `buy_overlay_due` is always `false` in the editor, under `--test`,
and for a game the account owns. `open_store_page` answers whether anything opened.

See [Steam integration](../../../../STEAM.md) for how the library is loaded, and
[HUD](../../../../05-ui/01-hud.md#languages) for what `ui.set_text_language` does with the result.
