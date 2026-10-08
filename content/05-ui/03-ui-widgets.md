---
title: "UI Widgets"
---

The HUD ([HUD](01-hud.md)) draws elements and answers "is the pointer on this one". Widgets are
the layer on top: **components that give an element a behaviour** (a button, a toggle, a slider, a
scroll list and a text field), a **per-frame event queue** that tells a
script what happened to them, and **layout containers** that arrange elements in rows, columns and
grids.

Everything here is opt-in. An element with only a UI Transform and a Sprite or Text is exactly what it
was: it emits no events, nothing touches its colour, and a game that builds its buttons by hand in Lua
keeps working untouched (see [Next to hand-rolled buttons](#next-to-hand-rolled-buttons)).

## The event queue

Once a frame, right after the hit test, the engine turns what the pointer did into events for the
elements that carry a widget component. A script reads them with `ui.events()`:

```lua
function on_update(dt)
    for _, e in ipairs(ui.events()) do
        if e.type == "clicked" and e.name == "PlayButton" then
            scene.open_scene("scenes/level1.Lscene")
        end
    end
end
```

Each entry is `{ entity = <entity>, name = "<tag>", type = "<type>", value = <number> }`, oldest first.
A [terminal](#terminal)'s `submitted` and `complete` also carry `text` (the line), and `complete` the `caret`; its `link` carries the link's payload in `text`. It is a fresh table per call, so a script may keep or modify it.

| `type` | Sent when | `value` |
|---|---|---|
| `hover_enter` | The pointer starts to be over the widget (over the widget **or its relative children**: a button's label is part of the button) | 0 |
| `hover_exit` | It stops | 0 |
| `pressed` | The left button goes down on it | 1 |
| `released` | The button that went down on it comes up, **wherever the pointer is** | 1 when it came up over the widget too, else 0 |
| `clicked` | Pressed and released on the same widget; sent right after `released` | 1 |
| `value_changed` | A widget's value moved: a toggle flipped (by a click, by its radio group, or by `set_value`), a slider's value changed, a scroll list scrolled, a text field's edit was committed | a toggle's 1 or 0, a slider's new value, a scroll list's new normalized position (0 to 1), a text field's new length |
| `submitted` | A focused widget was confirmed with the accept action, or Enter was pressed in a [text field](#text-field) | 0 |
| `text_changed` | A text field's text was edited: a keystroke, a paste, a delete, an Escape that put the old text back | the new length |
| `complete` | Tab in a [terminal](#terminal) | the caret |
| `link` | A [link span](#links) in a terminal's output was clicked | 0 |
| `focus_gained`, `focus_lost` | Keyboard or gamepad focus arrived / left, [when focus navigation is on](#focus-and-navigation) | 0 |
| `cancel`, `page_prev`, `page_next` | The cancel / page action was used; sent to the focus scope that has the focus (or to the focused widget), see [Focus and navigation](#focus-and-navigation) | 0 |

The first five come from the pointer. The others come from the widgets and the focus system, and only when
the project turns focus navigation on (`submitted` and the focus events), so a script written for a game
that does not use it never sees them.

Within one frame the order is always: hover (exit, then enter), `pressed`, `released`, `clicked`, then
the `value_changed` the click or the drag caused (for a radio group, the toggle that was clicked first,
then the ones it turned off), then anything a script raised itself.

`ui.changed(name)` is true when the element with that tag had a `value_changed` this frame, and
`entity:ui_changed()` is the same for an entity you hold.

### Frame order

Both hosts, the editor's Play mode and the standalone runtime, run a frame in this order:

1. **Scripts** (`on_update`) and physics: `Scene::OnUpdateRuntime`.
2. The frame is rendered. Its HUD is gathered (`GatherUIElements`) with the widgets' current tints.
3. **The hit test**, `Scene::UpdateUIInteraction`, reads the pointer (or the scripted one).
4. **The widgets**, `Scene::UpdateUIWidgets`, clears the last queue, builds this one from the hit
   test, and moves the tints along.

Scroll lists ride the same two steps: the wheel and a press that becomes a drag are taken up in step 3 (a
drag cancels the press there, before the click is decided), the offset moves in step 4, and a script sees the new
offset (`get_scroll`) and its `value_changed` in the next frame, like any event.

So a script always runs *before* the hit test of its own frame, and everything it reads (`ui.events()`,
`ui.is_clicked`, `ui.hovered`) is what steps 3 and 4 of the **previous** frame left. The queue stays
untouched until step 4 of the next frame, which is what makes three things true:

- **A click is seen exactly once**: by the script's next `on_update`, never twice and never missed,
  however many scripts look, and in whichever order they run.
- Every script in a frame sees the same queue.
- A pointer you set with `ui.set_virtual_pointer` in frame *n* is taken up by step 3 of frame *n* and
  its events arrive in frame *n + 1*.

A tint is applied when the HUD is gathered, which is before step 4, so what is on screen trails the
events by one frame. Widgets are only updated while the game is playing (Play or Pause in the editor,
always in the runtime): authoring a HUD in Edit mode does not light up its buttons.

## Button

Add **UI Button** (Add Component, next to Sprite and Text) to an element that has a UI Transform and
a Sprite, a Text, or both.

| Field | Default | |
|---|---|---|
| Disabled | off | Drawn with the Disabled tint; emits no events; is not hovered or clicked. It **still blocks the pointer**: what is underneath does not react to a click meant for the button, and `ui.pointer_over()` stays true over it (a disabled button is not a hole) |
| Normal Tint, Hover Tint, Pressed Tint, Disabled Tint, Fade Time, Click Sound, Hover Sound | | The shared [widget style](#the-widget-style) |

### The widget style

Button, Toggle and Slider carry the same set of look-and-sound fields, one `UIWidgetStyle` each, so a
tint means the same thing on all of them. The fields are saved flat in the widget's own block (a
Button's saved keys are the same as before the style was a struct).

| Field | Default | |
|---|---|---|
| Normal Tint | (1, 1, 1, 1) | Multiplied into the element's colour at rest |
| Hover Tint | (1.15, 1.15, 1.15, 1) | While the pointer is over it |
| Pressed Tint | (0.85, 0.85, 0.85, 1) | While it is down and the pointer is still over it |
| Disabled Tint | (0.6, 0.6, 0.6, 0.6) | While Disabled; the alpha fades it out |
| Fade Time | 0.08 | Seconds a change of state takes to cross-fade. 0 switches at once |
| Click Sound | empty | An audio clip, relative to the asset directory, played once on the **UI** audio bus when a click lands (a slider: when it is pressed) |
| Hover Sound | empty | The same, when the pointer enters |

**What a tint reaches.** The tint is multiplied into the colour of the entity's **own** Sprite (its
colour and its border colour) and its **own** Text (colour, outline, shadow), the way Unity and Godot
tint only a button's target graphic. Relative children, a label entity for one, are **not** tinted: put
the Text on the button entity itself when it should follow the tint, or colour the child from a script
(`set_ui_color`, or react to `hover_enter` / `hover_exit`). The tint is *multiplied*, so a tint above 1
brightens a sprite that is not already at full value and a tint on a black sprite does nothing.

**It is never written into the sprite.** The authored Sprite colour is untouched; the tint in effect is
run-time state that lives on the scene, is not saved, and is gone with it.

**A hit on a child is a hit on the button.** When the pointer is over a label or an icon inside a
button, the hit resolves to the nearest relative ancestor that is a widget. The events, `ui.hovered()`,
`ui.clicked()` and `ui.is_clicked("PlayButton")` therefore all name the button.

**Disabled in detail.** A press never starts on a disabled button, so a press dragged onto one is not a
click on it. Enabling and disabling can be done from a script, `entity:set_ui_disabled(true)`, and the
next frame's hit test takes it in. A button that is hovered or held when it becomes disabled still gets
the closing `hover_exit` / `released` (never a `clicked`), so a script tracking hover state cannot be
left stuck; no other event comes from a disabled button.

**Sounds.** `ClickSound` and `HoverSound` play through `AudioEngine::Play2D` on the UI bus, so
`audio.set_bus_volume("UI", v)` governs them. A missing clip or no audio device is silent, as everywhere
else in the engine. The two paths are discovered as dependencies by Build Standalone.

## Toggle

**UI Toggle** is a two-state widget: a checkbox, a switch or a radio button. A click flips it.

| Field | Default | |
|---|---|---|
| Is On | off | The state. What a click changes while playing |
| Disabled | off | As a Button's: drawn with the Disabled tint, deaf to the pointer, still blocks it |
| Group | empty | Non-empty makes a **radio group**: toggles with the same Group, anywhere in the scene, are exclusive |
| Allow Switch Off | off | Groups only. Off: clicking the toggle that is on keeps it on (something in the group always is). On: it turns off and none is left on |
| Graphic | none | An element, usually a child such as a check mark, drawn only while Is On |
| Widget style | | See [above](#the-widget-style) |

**Events.** A click sends the Button's `hover_enter`, `pressed`, `released` and `clicked`, then
`value_changed` (1 or 0). Turning a radio toggle on turns the others in its group off, and **each of those
sends its own `value_changed` (0)** in the same frame, after the clicked one's.

**Graphic.** The element is hidden while the toggle is off and shown while it is on. Its own `Visible`
is never written: `Scene::ResolveUIRect` applies "hidden while off" when the layout is resolved, the
same way a parent's visibility is inherited, so its Relative children go with it, `entity:is_ui_visible()`
still answers what the scene file says, and the Designer preview and the editor viewport follow Is On
too.

**Scripts.** `get_value()` answers 1 or 0, and `set_value(v [, notify])` takes a number or a boolean
(`true` is 1; any other number than 0 is on). It notifies by default, and a script can set a disabled
toggle (Disabled is for the pointer). Setting a group member on turns the rest of the group off
(with events, unless `notify` is `false`). Setting the last one on in a group that may not be switched
off, off, does nothing.

## Slider

**UI Slider** is a value between Min and Max that a press sets and a drag follows.

| Field | Default | |
|---|---|---|
| Min, Max | 0, 1 | The range |
| Value | 0 | Where it is now (what a press or drag changes while playing) |
| Step | 0 | 0 is continuous; otherwise the value snaps to `Min + n * Step` |
| Direction | Left To Right | Left To Right, Right To Left, Bottom To Top or Top To Bottom: the direction the value grows on screen |
| Disabled | off | As a Button's |
| Fill | none | The element stretched along the slider, from its start to the handle |
| Handle | none | The element that slides along the track |
| Widget style | | See [above](#the-widget-style); it tints the slider's own sprite (the track), not the Fill or the Handle |

**The track** is the slider's own rect, less the Handle's own length on the slider axis, so the handle
stays inside (as in Unity): the handle's centre moves from half its length in from the start to half its
length in from the end. With no Handle, the track is the whole rect.

**Press and drag.** A press on the slider (or its Fill or Handle, relative children resolve to it) jumps to
the pointer's value; dragging follows the pointer **wherever it is**, outside the rect and outside the
window too, because the press is captured for as long as the button stays down. Releasing ends it.
`value_changed` (value = the new value) is sent for each frame the value *changes*: holding still, or
moving inside one step, sends nothing. A disabled slider blocks the pointer and does nothing else.

**Fill and Handle are driven.** Like a toggle's Graphic, their placement is worked out when the layout is
resolved (`Scene::ResolveUIRect`), as the anchors would be, and is never written into their UI Transform:
on the slider axis their own anchors, offsets and pivot are ignored; the Handle keeps its Size, and the
Fill runs from the track's start (its far edge when the direction is reversed) to the handle's centre. The
other axis is theirs, so a Fill normally stretches across the slider's short side. They should be
Relative children of the slider (or of an element the size of its track): a hit on them then counts as a
hit on the slider. Their position is also right in the Edit-mode viewport and the Designer preview.
Author them on the slider's own rect: the handle with a Size on the slider axis and stretched anchors on
the other, the fill with stretched anchors on both.

**Scripts.** `get_value()` is the value, `set_value(v [, notify])` clamps to [Min, Max] and snaps to Step
(`notify` as for the toggle), and `entity:ui_adjust(steps)` steps it by `steps` notches (a notch is Step,
or a tenth of the range when Step is 0) and answers whether it changed: what a keyboard or gamepad
does to a focused slider, in C++ `Scene::AdjustUIWidget(entity, steps)`.

### Referring to other entities

A toggle's Graphic and a slider's Fill and Handle name their entity by **stable id** (the target's
`IDComponent` ID, `0` = none), like a constraint's `ConnectedEntity`, resolved through the scene's id
lookup; they survive a reload, a rename and a reorder. The inspector shows a combo of the HUD elements and
accepts an entity dragged from the hierarchy. When a **prefab instance** or a **paste** has to give a copy
a fresh id (a second instance of an object whose ids are already in the scene), the references inside the
copied set are rewritten to the new ids, so a slider prefab dropped twice drives its own Fill and Handle.
An id that finds nothing drives nothing, and the inspector says "(missing entity)". A paste or duplicate
copies only the entities that were selected, so duplicating a slider alone leaves its copy driving the
original's Fill and Handle: duplicate its children with it.

### The UI Designer

The Fill, the Handle and a toggle's Graphic are **driven elements**: their rect on the driven axis (and a
Graphic's visibility) come from the widget, not from their own fields, which a Designer edit of those
fields cannot change. The Designer preview draws them as they will be at the widget's current Value or Is
On, marks them with a padlock ("Driven by <widget>") and offers no handles on them: a drag selects them and
moves nothing. The Designer's Create palette makes a Button, Toggle, Slider, HBox, VBox and Grid already
wired; see [UI Designer](02-ui-designer.md#creating).

### From a script

| Call | |
|---|---|
| `ui.events()` | This frame's events, see above |
| `ui.changed(name)` | A `value_changed` for the element tagged `name` this frame |
| `entity:set_ui_disabled(disabled)` / `entity:is_ui_disabled()` | A widget's Disabled flag. No-op / `false` on an entity that is not a widget |
| `entity:ui_changed()` | `value_changed` this frame |
| `entity:get_value()` | The widget's value (a toggle's 1 or 0, a slider's value, a [text field](#text-field)'s text as a string), or `nil` for one with none (a Button) and for an entity that is not a widget |
| `entity:set_value(v [, notify])` | Sets a widget's value (a number or boolean; a string for a text field) and, unless `notify` is `false`, raises `value_changed` for the next frame. Does nothing for a Button |
| `entity:ui_adjust(steps)` | Steps a slider by notches; `true` when its value changed |

`ui.is_hovered`, `is_pressed`, `is_clicked`, `hovered`, `pressed`, `clicked` and `pointer_over` are
unchanged. For a widget they name the widget itself rather than a child of it.

## Text field

**UI Text Field** is a single-line text input: a name, a password, a number, a chat line. It is a widget like the Button
(hover, pressed look, `Disabled`, the shared [style](#the-widget-style)) whose value is a string, and which, while it is
being **edited**, takes the typed characters and the editing keys. Multi-line text is not supported: Enter submits.

### Setting one up

```
NameField       UI Transform, Sprite (the background), UI Text Field (Text Entity = NameText, Placeholder Entity = NamePh)
  NameText      UI Transform (Relative; anchors stretch across, 8 in from each side), Text            <- draws what is typed
  NamePh        UI Transform (Relative; the same rect), Text (grey)                                    <- draws the hint
```

| Field | Default | |
|---|---|---|
| Text | empty | What the field holds: the starting text, and what a script reads with `get_value()`. UTF-8 |
| Placeholder | empty | The hint, drawn by the Placeholder Entity while the field is empty **and not being edited** |
| Max Length | 0 | The most **characters** (not bytes) it holds, enforced as it is typed and pasted. 0 is unlimited |
| Filter | Any | What may be typed or pasted: **Any**, **Integer** (digits, a leading `-`), **Decimal** (digits, one `.`, a leading `-`), **Alphanumeric** (letters and digits of the scripts the HUD font covers: no spaces or punctuation). Control characters (a pasted newline) are never text |
| Password, Password Char | off, U+2022 | A password draws one Password Char per character instead of the text, and **refuses to copy or cut**. The HUD font has to have the character |
| Select All On Focus | on | Editing that starts from an accept (a gamepad's A, Enter, `ui_begin_edit`) selects all the text, so typing replaces it. A click places the caret instead |
| Submit Clears Focus | on | Enter ends the edit. Off keeps it going: a chat box that sends on Enter and stays ready |
| Disabled | off | As a Button's: the Disabled tint, deaf to the pointer and the keys, still blocks the pointer |
| Selection Colour | blue, 45% | The highlight behind selected text |
| Text Entity, Placeholder Entity | none | The elements that draw the text and the hint |
| Widget style | | See [above](#the-widget-style). While a field is being edited it uses its **Focus Tint** |

**The Text and Placeholder entities are driven, never written.** The Text Entity draws the field's Text (masked for a
password), shifted left so the caret stays in view when the text is longer than the element, and **cut to its own rect**;
its own `Text` is ignored. The Placeholder Entity draws the field's Placeholder and is hidden while the field has text or
is being edited. Their `Text` components are never changed (so `entity:get_text()` on them is what the scene file says,
not what is shown), and the Designer preview and the Edit-mode viewport draw the same. A field with no Text Entity draws
its **own** Text component the same way, which makes a one-element field. Put the Text and Placeholder entities in the
field as Relative children with a UI Transform; a click on either is a click on the field.

### Editing

A click on a field starts editing it and puts the caret where it landed; a **click anywhere else** (over the HUD or not)
ends it, keeping the text. With [focus navigation](#focus-and-navigation) on, moving focus to a field only focuses it:
`ui_accept` (Enter, Space, a gamepad's A) starts editing, and while it is being edited nothing navigates (below).

| Input | Does |
|---|---|
| Characters | Typed at the caret, replacing the selection. UTF-8: `héllo` and CJK work, and the caret and Backspace count characters |
| Left, Right | Move the caret one character; with Shift they extend the selection; with Ctrl they move by word. With a selection and no Shift they go to its start / end |
| Home, End | To the start / end (with Shift: select to it) |
| Backspace, Delete | Delete the character before / after the caret, or the selection; with Ctrl, a word |
| Ctrl+A | Select all |
| Ctrl+C, Ctrl+X, Ctrl+V | Copy, cut, paste through the system clipboard. Paste goes through the filter and Max Length. Nothing is copied or cut from a password |
| Mouse | Click: caret. Drag: select. **Double click**: select the word (all of a password). **Triple click**: select all. Shift+click extends |
| Enter | **Submit**: raises `value_changed` if the text differs from where the edit began (or from the last submit) and `submitted`, then ends the edit unless Submit Clears Focus is off |
| Escape | **Cancel**: the text goes back to where the edit began and the edit ends. It raises `text_changed` for the restore and nothing else, in particular **no `cancel` event**, so a dialog around the field is not also closed |
| Tab | Nothing |
| Gamepad (focus navigation on) | d-pad / left stick left and right move the caret, **A submits, B cancels** |

The caret blinks (about a second) and the view scrolls horizontally to keep it visible; the selection is a coloured
highlight behind the text. When the edit ends the field shows the start of its text again.

An edit also ends, committed, when the field is **disabled or hidden**, when **focus moves off it** (focus navigation on), when a
script or a click moves focus elsewhere, when one of the editor's own ImGui fields takes the keyboard, and when Steam's
on-screen keyboard is closed ([below](#steam-deck-keyboard)).

### The keyboard belongs to the field

While a field is being edited, **the game does not get the keyboard**: `input.is_key_down`, `is_key_pressed`,
`get_key_axis` and every key binding of the project's [actions](../06-scripting/02-lua/api/input/03-actions.md) answer as if nothing were
held, so a W typed into a name does not walk the player, and Space is not an accept, and the `ui_*` navigation keys stay out
of it. Gamepad bindings are not keyboard and stay live. This is one engine-wide flag, `UITextInput::WantsTextInput()`: ImGui's
own flag (one of the editor's panels, or the console, has the keyboard) **or** a HUD field being edited. It holds in the editor's Play
mode and in the standalone runtime alike, and in the editor it also silences the editor's own shortcuts (Ctrl+Z, the gizmo keys)
while you type into a field in the Play viewport. A C++ game module that reads keys itself should ask the same function.
The flag is raised the frame an edit starts and dropped the frame it ends; a key that was down when the edit ended is the game's
again at once.

Where the text comes from: both hosts run Dear ImGui over GLFW, and ImGui's own text callback feeds its input queue, which is
what the field reads (it only *reads* it, so ImGui's fields see every character as before; while one of them has the keyboard,
a HUD field does not edit). ImGui keeps characters as 16-bit values, so a character above U+FFFF (an emoji) is not typed.

### Events

| `type` | Sent when | `value` |
|---|---|---|
| `text_changed` | The text changed because it was **edited**: once per typed character, once for a paste, a cut, a delete, or an Escape that put the old text back | The new length, in characters |
| `value_changed` | An edit was **committed** with a different text: Enter, a click elsewhere, focus leaving, `ui_end_edit(true)`, the Steam keyboard closing. Also a script's `set_value` (unless `notify` is `false`), which is not a keystroke and raises no `text_changed` | The new length |
| `submitted` | Enter (or a gamepad's A) in the field | 0 |

So a script that wants every keystroke (a live search) listens for `text_changed`, and one that wants the finished entry
(a name) listens for `submitted`, or for `value_changed` when leaving the field should count too. The usual hover / pressed /
clicked events are the Button's, and `focus_gained` / `focus_lost` are focus navigation's.

### From a script

| Call | |
|---|---|
| `field:get_value()` | The text, a string (the other widgets answer a number). **It is the text, not what is drawn**: a password is the real characters |
| `field:set_value(text [, notify])` | Sets the text (a number becomes its text). It is held to the Filter and Max Length like typing, keeps the caret in range, and raises `value_changed` unless `notify` is `false`. What a script sets is the new baseline an Escape restores. Works on a disabled field |
| `field:ui_begin_edit()` | Starts editing it (selecting all when Select All On Focus is on). `false` when it is not a field, is disabled or hidden |
| `field:ui_end_edit([commit])` | Ends *its* edit; `commit` defaults to `true` (keep the text, `value_changed` if it changed) and `false` puts the old text back. `false` when it was not being edited |
| `field:ui_is_editing()` | Whether it is being edited |

```lua
function on_update(dt)
    for _, e in ipairs(ui.events()) do
        if e.name == "NameField" and e.type == "submitted" then
            player_name = scene.find("NameField"):get_value()
        end
    end
end
```

### Steam Deck keyboard

On a Steam Deck (Steam's `SteamDeck=1`, or the library's `IsSteamRunningOnSteamDeck`), starting an edit asks Steam for its
**floating keyboard** (`ISteamUtils::ShowFloatingGamepadTextInput`) over the field: a numeric layout for Integer and Decimal
fields, a single-line one otherwise. The rectangle is the field's, in window pixels (the HUD viewport scaled to the window: the
runtime's game fills its window). What is typed arrives as ordinary key and character input, so the field needs nothing more.
Ending the edit asks Steam to close it, and **Steam closing it ends the edit** (committed, no `submitted`): that callback
(`FloatingGamepadTextInputDismissed_t`) is read by the Steam callback pump the runtime already runs for stats, so where that
is not running (the editor) the keyboard closing is not seen and the field ends its edit by its own keys. Off a Deck, without
Steam, or with a Steam library that predates the call, it does nothing at all: the runtime never depends on Steam.

### The UI Designer

The Text and Placeholder entities are **driven**: their text comes from the field. The Designer preview draws the field's Text
(and the Placeholder while it is empty); the entities' own Text is ignored there too. The Create palette's text field is described in
[UI Designer](02-ui-designer.md).

## Terminal

**UI Terminal** is a scrolling console: output lines above a prompt line you type on. Built for in-game shells, hacking and chat
logs. It draws its own text inside the element's rect (give the element a Sprite for the background), keeps the scrollback, the
typed line, the caret, a history and the scroll position itself, and talks to a script through `entity:term_*` calls and two events.
It is a widget (hover, press, `Disabled`) but takes no tint: its Sprite stays as authored.

| Field | Default | |
|---|---|---|
| Prompt | `> ` | Drawn before what is typed, in the Prompt Colour. A script changes it with `term_set_prompt` |
| Initial Text | empty | Put in the scrollback the first time the terminal is used (a banner). Newlines make lines; colour tags work. The UI Designer shows it too |
| Max Lines | 2000 | Scrollback lines kept, the oldest go first. 0 keeps all |
| History Size | 100 | Typed lines Up and Down step through. 0 keeps none |
| Font Size, Line Spacing | 18, 1 | UI units, and a multiplier on the font's own line height |
| Text, Prompt, Caret, Selection Colour | green-grey, green, near white, blue | |
| Link Colour, Link Hover Colour | light blue, paler blue | Link text with no colour tag of its own, and the link under the pointer (also underlined). See [Links](#links) |
| Padding | 10 | UI units between the rect's edge and the text. The scrollbar sits in it |
| Rich | on | Output lines read `<c=#rrggbb>..</c>` colour tags and `<l=payload>..</l>` links (see [HUD](01-hud.md)); `<<` is a literal `<`. Off prints lines as they are |
| Caret Blink | on | |
| Read Only | off | Output only: no prompt line, no typing, no focus. The wheel still scrolls it |
| Disabled | off | Drawn dimmed, deaf to the pointer and the keys |

Only this setup is saved. The lines, the typed text, the history and the scroll are run-time state: never saved, never copied into Play.

### Output

`term:term_print(text)` adds lines: a newline starts the next one, and one at the very end is a terminator, not a blank line.
A tab is four spaces. Lines wrap to the width of the rect (less its padding) at the last space that fits, and **cut anywhere** when
there is no space to break at, so a path or a hash never runs out of the box. Wrapped rows count when scrolling.

The view **follows the end**: new output scrolls into view while you are at the bottom. Scroll up (wheel, PageUp) and it stays where you
are, however much arrives. Typing, Enter, Up / Down, `term_set_input` and `term_scroll_to_end` go back to the end.

### Keys

A click focuses the terminal (a click elsewhere, or Escape, lets go). While it is focused it owns the keyboard like a
[text field](#the-keyboard-belongs-to-the-field): the game's key reads and bindings are silent, and focusing a terminal ends a text field's edit.
**A few keys still reach the game**, because they are never text (see [Keys the game still gets](#keys-the-game-still-gets)).

| Input | Does |
|---|---|
| Characters | Typed at the caret in the prompt line, replacing the selection (UTF-8). A long line wraps onto more rows |
| Left, Right, Home, End, Backspace, Delete | As in a text field: Shift selects, Ctrl moves or deletes by word |
| Ctrl+A, Ctrl+C, Ctrl+X, Ctrl+V | Select all, copy, cut, paste. A pasted newline cuts the paste: only its first line goes in |
| Enter | Echoes `prompt + line` into the scrollback (the prompt in the Prompt Colour), adds the line to the history unless it is empty or the same as the last, clears the input, goes to the end and raises `submitted` with the line. Empty lines are submitted too |
| Up, Down | Browse the history. What was typed before Up is kept and comes back when Down goes past the newest |
| Tab | Raises `complete` with the line and the caret. The script answers with `term_set_input` or `term_print`. Tab never moves focus. **Ctrl+Tab does nothing here**: no `complete`, and the game can read it (see below) to move focus itself |
| PageUp, PageDown | Scroll a page less a row. Shift+PageUp goes to the top, Shift+PageDown to the end |
| Ctrl+L | Clears the scrollback |
| Escape | Lets go of the keyboard, from a real key and from `input.inject_key` / the MCP `send_input` tool alike (`Esc` and `Escape` both name it). The key is not also a `cancel` for whatever has focus next |
| Mouse wheel | Scrolls three rows a notch while the pointer is over it |
| Mouse | A click on the prompt line puts the caret there, a drag selects |

#### Keys the game still gets

While a terminal is focused the game's key reads (`input.is_key_down`, `is_key_pressed`, `is_key_released`, `get_key_axis`) and the key
bindings of the project's input actions see these keys as normal, and the terminal ignores them:

- **F1 to F12**, with or without modifiers.
- **Any key held with Alt** (Alt+1 to Alt+5, Alt+Q, ...), and the Alt keys themselves. With Alt held the terminal also ignores typed
  characters and its own editing keys, so a hotkey is never typed into the prompt. Alt together with Ctrl (AltGr, which types `@` and the
  like on many layouts) does not count: those stay the terminal's.
- **Tab with Ctrl held** (Ctrl+Tab).

Every other key stays silent for the game: letters, digits, arrows, Enter, Backspace, Escape, a bare Tab. A [text field](#text-field)
that is being edited silences all of them, with no exceptions. `ui.features().key_passthrough` is true on an engine that does this.

### Links

A line of output can carry clickable spans:

```lua
term:term_print("see <l=open docs>the docs</l> or <c=#ffcc00><l=cd /logs>logs</l></c>")
```

- `<l=PAYLOAD>text</l>`. The payload is any text up to the first `>`. Write `\>` for a `>` and `\\` for a backslash inside it (a backslash
  before anything else is itself). In a Lua string literal that is `"<l=a\\>b>"` for the payload `a>b`.
- Links nest with colour tags either way round. The link text is drawn in **Link Colour** unless a colour tag around it sets its own; under
  the pointer it turns **Link Hover Colour** and is underlined.
- A payload cannot be empty, a link inside a link is not a link, and a tag that does not fit these rules stays in the text as written. A
  link left open runs to the end of its line; a link with nothing between the tags is dropped. Links are per line: a `\n` inside one ends it.
- A link that wraps is clickable on every row it covers. Only the output is clickable, not the prompt line, and links only exist while **Rich** is on.
- A click is a press and a release on the same link, with the pointer not dragged away. It raises a `link` event with the payload in `text`.
  It **does not focus the terminal, move the caret or touch the typed text**, whether or not the terminal is focused, and works on a Read Only
  terminal. A drag that started on the prompt line (selecting) never clicks a link.
- `ui.hand_cursor()` is true while the pointer is over a link. The engine draws no cursor of its own, so a game that shows one follows this.
- `entity:term_link_at(x, y)` answers the payload of the link under a point (HUD pixels, as `get_ui_rect` reports them), or `nil`. Handy in tests.
- `ui.features().rich_links` is true on an engine that has all this, so a game can fall back to typed commands on an older build.
- A **Text** element with Rich Text on reads the tags too but does not make links of them: the markup is stripped and the text is plain.

A key that opens the terminal from a script is still in that frame's text input when `term_focus()` runs, so its character can land in the
prompt line. Call `term_set_input("")` a frame later if that matters.

### Events

Through `ui.events()` like every widget; the line is in `text`.

| `type` | Sent when | `value`, extra |
|---|---|---|
| `submitted` | Enter | 0; `text` is the line |
| `complete` | Tab (not Ctrl+Tab) | the caret (characters); `text` is the line, `caret` the same number |
| `link` | A link was clicked | 0; `text` is the payload |

```lua
local term = scene.find("Shell")
term:term_print("<c=#4fd8ff>10.0.0.1</c> connected")
term:term_focus()
function on_update(dt)
    for _, e in ipairs(ui.events()) do
        if e.name == "Shell" and e.type == "submitted" then
            if e.text == "help" then term:term_print("commands: help, ls, quit")
            else term:term_print("<c=#ff5050>unknown command</c>: " .. e.text) end
        elseif e.name == "Shell" and e.type == "complete" and e.text == "he" then
            term:term_set_input("help")
        end
    end
end
```

### From a script

All of these do nothing, and log a warning, on an entity that has no terminal. See [HUD (entity)](../06-scripting/02-lua/api/entity/10-hud.md#terminal).

| Call | |
|---|---|
| `term_print(text)`, `term_clear()` | Add lines, drop all of them |
| `term_set_prompt(s)`, `term_get_prompt()` | |
| `term_set_input(s [, caret])`, `term_get_input()` | The typed line; the caret defaults to the end |
| `term_focus()`, `term_unfocus()`, `term_is_focused()` | Focus from a script (`false` for a read only or disabled terminal) |
| `term_scroll_to_end()` | Follow the end again |
| `term_line_count()`, `term_get_line(i)` | Scrollback lines, and line `i` (1 based) without its colour or link tags |
| `term_link_at(x, y)` | The payload of the link under a HUD pixel, or `nil` |
| `term_set_history(table)`, `term_get_history()` | The remembered lines, oldest first |

### Cost

Wrapping is cached per line and redone only when the width, the font size or the font changes; only the rows on screen are shaped and
drawn, plus the prompt line, so a long scrollback costs nothing per frame. The HUD draws one quad per glyph, so a full screen of text is a few
thousand draw items (`ui_terminal.lua` prints the measured numbers).

### The UI Designer

The palette has a **Terminal**: a dark sprite with a terminal on it and a few lines of Initial Text, which is what the preview shows.

## Layout containers

A **UI Layout** (Godot's HBox, VBox and GridContainer in one component) puts its children in a row, a
column or a grid, so a menu, a toolbar or an inventory is not a pile of hand-computed offsets. Add **UI
Layout** to an element with a UI Transform; its children are the elements parented to it that are
**Relative** (the usual kind of child, [HUD: relative elements](01-hud.md#relative-elements)).

```
Menu          UI Transform, Sprite, UI Layout (Vertical Box, Padding 16, Spacing 0 8)
  Title       UI Transform (Relative), Text      <- Size is its preferred size
  PlayButton  UI Transform (Relative), Sprite, UI Button
  QuitButton  ...
```

A child's rect is worked out when the layout is resolved, in the same place as anchors, and **never
written** into its UI Transform: a laid-out child's own anchors, pivot and offset do not place it, and
its **Size is its preferred size**. Change a child's Size and the container re-lays out; remove the
container component and the child is where its own anchors put it again. The scripted `set_ui_size` and
the Designer still edit Size, which is the preferred size.

| Field | Default | |
|---|---|---|
| Type | Horizontal Box | Horizontal Box (a row), Vertical Box (a column) or Grid |
| Padding | 0 | Left, top, right, bottom, in UI units: space kept free inside the container's rect |
| Spacing | 0, 0 | Between children: x between the children of a row or the columns of a grid, y between the children of a column or the rows of a grid |
| Main Align | Start | Box only. Where the children sit along the main axis (x for a row, y for a column) when there is room to spare: Start (left/top), Center, End. There is none to spare when a child expands |
| Cross Align | Stretch | Box: how a child sits across the main axis: **Stretch** fills it (as a Godot container's children do), or Start / Center / End keep the child's preferred size there. Grid: the same inside each cell, on both axes (labelled Cell Align). A [Toggle](#toggle) child is never stretched (see below) |
| Columns | 2 | Grid only |
| Cell Size | 0, 0 | Grid only. A fixed cell; a 0 on an axis sizes every column (row) to the widest (tallest) child in it |
| Reverse Order | off | The children are placed in the opposite order. Main Align is not reversed |
| Fit Content | off | See below |

### Which children, and in which order

The children that are laid out are the **direct** children that have a UI Transform, are **Relative**,
are **visible** and are not **Ignore Layout**. A hidden child (its Visible off, a toggle's Graphic while
the toggle is off, the Designer's eye) **takes no space**, as in Godot; showing it again takes its space
back. A slider's Fill and Handle are placed by the slider and are not laid out. A child that is not
Relative is not a child for layout purposes; one with **Ignore Layout** (on a UI Layout Element) keeps its
own anchors and Size, inside the container's rect, and does not move the others. The container
**overflows rather than shrinks**: children bigger than the container simply run past it, and
**Clip Children** on the container (UI Transform) cuts them off.

They are placed in ascending **Order** (UI Layout Element, 0 without one), ties by the child's **ID**.
A hierarchy keeps no stored sibling order (what a scene file writes is not a promise about it), so this is
the only rule that is the same on every load: **set Order when the sequence matters**, in particular for
elements created at run time, whose ids are random (`tile:set_ui_layout_order(n)` before or after parenting
it). (Reordering in the UI Designer is done by rewriting Order.)

**A Toggle is not stretched.** A toggle's rect is its box, and its label usually sits outside it (the
Designer's Toggle puts the label to the right of the box), so widening the box would push the label out of
the container. Under Stretch a toggle keeps its own size across the main axis, at the start, as Start would
(in a Grid, inside its cell); Center and End still place it. Along the main axis it is sized like any other
child, so Expand still works on it.

### UI Layout Element

Optional, on a child. Without it a child uses its Size as its preferred size, does not expand and has
Order 0.

| Field | Default | |
|---|---|---|
| Min Size | 0, 0 | The least the child gets. Its preferred size is the larger of its Size and this |
| Expand X, Expand Y | off | Along the container's main axis (X in a row, Y in a column): share the space the other children leave. Across it Cross Align decides, and in a Grid neither is used |
| Stretch Ratio | 1 | An expanding child's share relative to the other expanding children |
| Order | 0 | See above |
| Ignore Layout | off | Not laid out |

**Expand.** The children that do not expand take their preferred size. What is left of the main axis (minus
spacing) is shared by the expanding ones in proportion to Stretch Ratio, **instead of** their own size, as in
Godot; a share smaller than a child's preferred size (its Size, or Min Size if larger) fixes that child at that
size and the rest is shared again among the others. So Size and Min Size are the floor an expanding child
never drops below. For example, in a 400-unit row with 10 of spacing between four children, a fixed 50,
two that expand with ratios 1 and 2, and one that expands with Min Size 150: the 320 that are left would
give that last one only 80, so it is held at 150 and the other two share the remaining 170 as 1 : 2.
With no room left they just keep their preferred size and overflow.

### Fit Content

With **Fit Content** the container's own size becomes its children's extent plus padding (the row's total
width and its tallest child, and so on), on each axis **where its anchors do not stretch it**: a container
that stretches to its parent stays stretched on that axis. The size is computed at layout time; the
container's Size is not written. It nests: a VBox of Fit Content HBoxes is as tall as its rows and as wide
as the widest, because sizes are computed from the leaves up (each container's preferred size is its
content's size), and rects are then handed down from the top. A container that is **itself laid out by a
parent container** is sized by that parent, which may stretch it (the content size is then only its
preferred size); its own Fit Content applies when it stands on its own.

### Grids

A Grid fills **row by row** into Columns columns. Each column is as wide as its widest child and each row
as tall as its tallest (auto cells) unless Cell Size fixes an axis; children fill their cell (Cell Align
Stretch) or keep their preferred size at its start, centre or end. A child bigger than a fixed cell
overflows it.

### From a script

| Call | |
|---|---|
| `entity:set_ui_layout_order(n)` / `entity:get_ui_layout_order()` | A child's Order (creates its UI Layout Element on first set); 0 when it has none |
| `entity:set_ui_layout_spacing(x, y)` | A container's Spacing. No-op on anything that is not a container |
| `entity:set_ui_layout_padding(left, top, right, bottom)` | A container's Padding |
| `entity:set_ui_layout_columns(n)` | A Grid's Columns (at least 1) |
| `entity:get_ui_rect()` | Reads the laid-out rect like any element's, and so does `entity:get_ui_rect_at(width, height)`: the rect as if the HUD were laid out at a render target of that many pixels (its UI scale included), which is how to see a container at another resolution |

A tile or row created at run time (`scene.spawn(...)` then `entity:set_parent(container)`) is laid out
from the next resolve of the layout, which is the same frame: there is nothing to invalidate.

### Cost

The layout is resolved from scratch once a frame for drawing and once for the hit test (and once for each
`get_ui_rect` call). A container is arranged once per resolve, in O(children) plus sorting them by
Order; the only per-child allocations are the result entries. `ui_containers.lua` times it: resolving a
HUD of about 280 elements with a 200-child grid takes about 0.08 ms in a Release build (and about 9 ms in an
unoptimised Debug one, where the per-element cost of the whole HUD is the same as without containers), so
a frame's few resolves are well under one percent of a frame.

### The UI Designer

A child that a container lays out is **driven** like a slider's Fill: its rect comes from the container,
so its own anchors and offsets cannot be edited meaningfully (the inspector says so), and moving it in the
Designer means changing its Order or its container's settings. `Scene::GetUILayoutContainer(entity)` names
the container that places an element (null if none), which is what the Designer uses to lock it: a padlock
badge ("Placed by <container>"), no handles, and a drag that selects and moves nothing. An element created
inside a container from the palette takes the next Order (a UI Layout Element is added where missing), so
it goes last. Dragging a container's child in the Designer reorders it (Order is rewritten 0..n-1 over its
siblings) or, pulled out of the container, re-parents it, and a selected container has padding bars, spacing
grips and (a Grid) a columns chip on the canvas, and no resize handle on a Fit Content axis. The Designer
preview draws containers laid out, and Fit Content as sized.

## Scroll lists

A **UI Scroll List** (Unity's ScrollRect, Godot's ScrollContainer) turns an element into a **window onto
taller content**: the element's own rect is the window, its content is shifted by a scroll offset and cut to
the window, and the wheel, a drag and a scrollbar move it. The common recipe is a list whose one child is a
container that fits its items:

The [UI Designer](02-ui-designer.md#creating)'s **Scroll List** palette entry makes exactly this, wired, and
[its scroll preview](02-ui-designer.md#scroll-lists) scrolls it on the canvas without touching the saved scene.

```
Levels          UI Transform, Sprite, UI Scroll List (Vertical; Content = LevelList, V Scrollbar = Bar)
  LevelList     UI Transform (Relative; anchors stretch across, top-aligned), UI Layout (Vertical Box, Fit Content)
    Level1      UI Transform (Relative), Sprite, Text, UI Button      <- Size is its preferred size
    Level2      ...
  Bar           UI Transform (Relative; anchors stretch down the right edge), Sprite, UI Scrollbar (Handle = BarThumb)
    BarThumb    UI Transform (Relative), Sprite
```

`LevelList` is as wide as `Levels` (it stretches across) and, with **Fit Content**, as tall as its items, so
that is the content's height. Add or remove items (`scene.spawn(...)`, `set_parent`, `scene.destroy`) and the
scrollable distance follows at once.

| Field | Default | |
|---|---|---|
| Vertical, Horizontal | on, off | The axes that scroll. The plain wheel scrolls the vertical one; a list that is only Horizontal takes the plain wheel sideways; **Shift + wheel** (and a horizontal wheel notch) scrolls the horizontal axis of a list that has both |
| Scroll Speed | 60 | UI units per wheel notch |
| Drag To Scroll | on | A press and drag on the content scrolls it; see [Drag and click](#drag-and-click) |
| Inertia | 0 | 0 stops the content when the drag is let go of. Above 0 it coasts on, and is about 95% slower after this many seconds |
| Smoothing | 0 | 0 moves the content at once. Above 0 the wheel (and `scroll_to`) only move a **target** and the content glides to it, covering half of the remaining distance every this-many seconds, whatever the frame rate: `0.06` is the soft glide of Tiny Lab's level list. Notches in a row add up from the target, not from where the content is. A drag, a scrollbar, `set_scroll` and scroll-into-view are direct and end a glide |
| Clamp To Content | on | The offset stays between 0 and (content - window). Off allows overscroll (a script's elastic effect) |
| Content | none | The child that scrolls. None scrolls **every direct Relative child** that is not a scrollbar and not Ignore Layout |
| V Scrollbar, H Scrollbar | none | Elements with a [UI Scrollbar](#scrollbars) that show and set the position |

**The window clips its content**, whether or not Clip Children is ticked: the content is cut at the
list's rect in what is drawn and in what the pointer can hit. Children that do not scroll (the scrollbars,
and a child with Ignore Layout, such as a pinned header) are not clipped by it, so a bar may sit beside the window.

**The content's extent is its rect.** With one Content child that is that child's rect, which for a Fit Content
container is the size of its items. Without Content it is the union of the rects of the children that scroll
(the list may itself carry a UI Layout, which places them first). The list can move the content by `content far
edge - window far edge` on an axis, so padding between the window's edge and the content is part of what
scrolls, and it is 0 when the content's far edge is inside the window.

**How the offset is applied.** The offset is run-time state kept on the scene, like a widget's tint: it is
never saved, never copied into Play, and is 0 in Edit mode and in the UI Designer, so a list always opens at its
authored positions. At layout time `Scene::ResolveUIRect` subtracts it from the content's rect; everything
Relative to the content resolves against that rect, so the whole subtree moves with it without any
UI Transform being written. Drawing and the hit test read the same rects and clips, so a button is clickable
exactly where it is drawn, and a part of a button outside the window cannot be hit.

### Wheel, drag and the pointer

- **The wheel** scrolls the innermost list under the pointer that has something to scroll. A popup (an
  element in front of the list that is not part of it) keeps the wheel. A list with nothing to scroll leaves the
  wheel to a list around it.
- **Drag and click.** A press on the content, a button included, may become a drag: once the pointer has moved
  **6 pixels** along an axis the list scrolls on, the content follows it (from where the press began, so
  it does not lag by the 6), and the press is **cancelled**: whatever was pressed gets `released` with
  value 0 and **no `clicked`**, as in Unity. A press that stays inside 6 pixels is still a click. A press on a
  **slider or a scrollbar** never becomes a scroll drag (they have their own), and a list with nothing to scroll does not
  take drags at all, so its buttons click as usual. Dragging goes on outside the window and the viewport until the
  button is let go.

### Scrollbars

**UI Scrollbar** goes on the bar's track (the element you see as the groove). Name it in the list's V Scrollbar
(it then scrolls vertically) or H Scrollbar.

| Field | Default | |
|---|---|---|
| Handle | none | The thumb, normally a child of the track. **Driven** like a slider's Handle: its position along the bar comes from the offset and its length from the window's share of the content, `window / (window + max offset)` of the track, never below Min Handle Size; its own anchors on that axis are ignored (the other axis is its own) |
| Min Handle Size | 16 | UI units |
| Hide When Fits | on | The track, and the thumb with it, are hidden while there is nothing to scroll |
| Jump On Track Click | off | A press on the track puts the thumb's centre at the pointer and drags on from there. Off: it **pages** one window toward the pointer |
| Disabled, style | | As a Button's: the [widget style](#the-widget-style) tints the track |

A press on the thumb grabs it and the offset follows the pointer along the track, outside it too, until the
button is let go. The scrollbar is a widget (it has hover, pressed and disabled looks and sends the
pointer events), but a list is not: a list has no tints or states, and a hit on its content stays a hit on the
content.

**Why a component of its own and not a Slider.** A slider is a *value* control: a press jumps to the pointer, its handle
has the length it was authored with, it is a focus target for a gamepad, and it sends its own `value_changed`.
A scrollbar needs a proportional thumb, pages on a track click and must not be a focus stop, and reusing a
slider would have made each of those a special case inside it. The two share the machinery that matters (driven
handle, press capture, widget style).

### From a script

| Call | |
|---|---|
| `list:get_scroll()` | The offset in UI units, `x, y`: how far the content has moved left and up. Zeros on an entity that is not a list |
| `list:set_scroll(x, y [, notify])` | Moves it, clamped as the list says (and ignoring an axis the list does not scroll). Returns whether it moved. Notifies by default |
| `list:scroll_to(x, y)` | Sends it there the way the wheel does: with Smoothing above 0 only the target moves and the content glides to it (`get_scroll` follows it frame by frame); with 0 it is `set_scroll`. Returns whether it has somewhere to go |
| `list:get_scroll_max()` | `x, y`: the most it can move (0 when the content fits). Follows the content as it grows |
| `list:get_scroll_normalized()` | `x, y`, each 0 to 1 of that distance (0 when there is none) |
| `list:set_scroll_normalized(x, y [, notify])` | The same, from a position |
| `list:get_value()` / `list:set_value(v [, notify])` | The normalized position on the main axis: vertical, or horizontal for a list that does not scroll vertically |
| `list:ui_scroll_into_view(child)` | Scrolls the least that makes `child` (a descendant of the list) fully visible; a child bigger than the window lines its start up. Returns whether it moved. A child already inside, or not inside the list, answers `false` |

`value_changed` (value = the normalized position of the main axis) is sent for each frame the offset
changes, by whatever changed it (wheel, drag, bar, coasting, a script's call with `notify`). A script's
`set_scroll(..., false)` is silent. The scrollable distance changing because the content did is not a change of
the offset, unless the offset had to be pulled back inside.

In C++, `Scene::ScrollUIIntoView(scroll, child)` is the same call, `Scene::FindUIScrollOf(element)` names the
list an element is in, and `Scene::GetUIScrollInfo` / `SetUIScroll` read and write the offset. Keyboard and
gamepad focus moving to an element in a list calls `ScrollUIIntoView` for it.

`ui.set_virtual_pointer(u, v, buttons, wheel, wheel_h, shift)` takes a horizontal notch and the Shift state
as its last two arguments, so a test can drive every wheel case.

### The UI Designer

The scroll offset is 0 in Edit mode, so the Designer shows the content where it is authored and a Designer
edit of it is an edit of its place at offset 0. A scrollbar's **thumb** is driven (padlock, "Driven by
<scrollbar>"); the content, the scrollbars' tracks and everything inside it are ordinary elements. The window clips
its content in the Designer too, so content that overflows the window is not drawn or selectable beyond it
(select it in the hierarchy, or enlarge the window while you edit).

### Cost

A frame resolves the HUD layout one more time while any list exists (the interaction update, in Play); measuring a
list is O(its scrolled children), and the per-frame state is a few vectors per list.

## Focus and navigation

**Opt-in, and off by default.** With **UI focus navigation** off (Project Settings, `UIFocusNavigation` in
the `.lab`), nothing in this section exists: no focus, no ring, no input polled, every call below answers
`false` or `nil`. A game that draws and drives its own cursor, as Tiny Lab does from a gamepad script, is
untouched. On, the HUD can be driven without a mouse: the arrows, the d-pad and the left stick move a focus
between widgets, Enter / Space / A activates, Escape / B cancels, and the mouse and focus agree.

**What can be focused.** A Button, Toggle, Slider or Text Field that is visible, enabled and has no UI Focus component
saying `Focusable` off. Scrollbars and scroll lists never are. A widget clipped out of a Clip Children
container is skipped; one in a scroll list is not (see below), so a long list can be walked.

### Moving

| Input | Does |
|---|---|
| `ui_up` / `ui_down` / `ui_left` / `ui_right` | Move focus to the widget in that direction. Held, they repeat (0.35 s, then every 0.08 s) |
| `ui_accept` | A Button is **clicked** (`clicked`, its click sound, a short pressed look), a Toggle flips (`clicked`, `value_changed`), and every accept sends `submitted`. On a [Text Field](#text-field) it **starts editing** (and sends nothing) |
| `ui_cancel` | Sends a `cancel` event to the focus scope |
| `ui_page_prev` / `ui_page_next` | Send `page_prev` / `page_next` to the focus scope (tabs, pages) |

- **Nothing focused:** the first direction (or accept) focuses the widget marked **Default Focus**, else the
  top-left-most one, and does nothing else.
- **Which neighbour.** The candidates are the widgets whose centre is past the focused one's in the direction and
  whose near edge is not far behind its far edge. The best is the one with the smallest `gap along the direction + 2 x gap
  across it (0 when they overlap there) + 0.25 x offset between centres across it`: the nearest widget **in line**,
  not a nearer one a row away. An irregular layout works the same way (`UIFocusMath::Pick`). There is **no wrap**:
  at the edge focus stays, unless Project Settings has **Wrap**, which continues from the far end.
- **Explicit neighbours.** UI Focus on a widget names an Up, Down, Left and Right neighbour by id; one that can
  take focus wins over the search, a hidden or disabled one falls back to it. The ids are remapped when a prefab is
  instantiated or pasted, like a slider's Handle.
- **A focused Slider** takes the direction along it (Left / Right, or Up / Down for a vertical one, as the value
  grows on screen) as an adjustment of one notch, like `ui_adjust`, and keeps focus; the other directions move on.
- **The pointer.** Moving the mouse onto a focusable widget focuses it; a mouse that is not moving never takes
  focus back from the keys, and a click on nothing leaves it where it is.
- **Scroll lists.** Moving focus to a widget inside a [scroll list](#scroll-lists) scrolls the least that
  shows it, in every list around it. A row just outside the window is still a candidate for that reason.
- **When the focused widget goes away** (hidden, disabled, destroyed, taken out of the scope), focus moves to
  the nearest valid widget, or clears when there is none, with the events.

**Scopes.** UI Focus Scope on a panel (or a root) with **Modal** on confines navigation to what is inside it
while it is visible: a pause menu or a dialog. With several visible, the one with the highest layer wins. When it
appears, focus goes to its Default Focus widget (else the nearest); when it goes, focus returns to the widget that
had it. `cancel` goes to the modal scope, or to the nearest scope around the focused widget, or to the focused
widget itself.

**Looking focused.** The focused widget is drawn with its **Focus Tint** (in the [widget style](#the-widget-style),
default the hover brightness, not used when pressed or disabled) and has a **ring** around it: an outline-only
rounded quad, painted over it and cut to the same clip. The ring is a project setting (width, gap, radius, colour;
width 0 draws none); the tint is per widget, so a game can have either or both. `FocusGained` / `FocusLost` carry
the change for scripts that want their own look.

### The input actions

The input is the project's own [actions](../06-scripting/02-lua/api/input/03-actions.md), so a player can remap it in the same
panel. Turning focus navigation on (in Project Settings, or by loading a project that has it on) **adds the actions that are
missing** and leaves any that exist alone, so a project's own `ui_accept` wins:

| Action | Bindings |
|---|---|
| `ui_up` | Up arrow, d-pad up, left stick up |
| `ui_down` | Down arrow, d-pad down, left stick down |
| `ui_left` | Left arrow, d-pad left, left stick left |
| `ui_right` | Right arrow, d-pad right, left stick right |
| `ui_accept` | Enter, keypad Enter, Space, gamepad A |
| `ui_cancel` | Escape, gamepad B |
| `ui_page_prev`, `ui_page_next` | Gamepad left / right bumper |

The direction actions are read as **axes** (above 0.5 counts), which is why the stick bindings carry a sign: up is a
negative `left_y`. Letters (WASD) are not bound, so they stay free for movement. The keys are not read while a text field
has the keyboard, and **while a [Text Field](#text-field) is being edited** nothing here navigates: the arrows move its caret, Space and Enter
are text and submit, Escape cancels the edit (with no `cancel` event), and a gamepad's d-pad / stick moves the caret, A submits and B cancels.
The pointer does not take focus from a field being edited (a click does); Enter or Escape that ended an edit does not also act as an
accept or a cancel while it is still held.

### From a script

| Call | |
|---|---|
| `ui.focused()` | The focused entity, or `nil` |
| `ui.set_focus(name \| entity \| nil)` | Focuses that widget (a tag name or an entity), or clears focus for `nil`. `false` when it cannot take focus now: flag off, not a focusable widget, hidden, disabled, outside the modal scope |
| `ui.navigate("up" \| "down" \| "left" \| "right")` | What the direction actions do. Answers whether anything happened |
| `ui.accept()` / `ui.cancel()` / `ui.page_prev()` / `ui.page_next()` | The same for the other actions: for a test, or for a game with its own input scheme |
| `ui.cancelled()` | Whether a `cancel` event was raised this frame (a script's call, or the action): the usual way to close a menu |
| `entity:ui_focus()` / `entity:ui_focused()` | Focus this widget / whether it has focus |

Focus itself changes at once; the events a call raises join the next frame's queue, like `set_value`.

### Recipe: a pause menu

```
HUD                    UI Transform (full rect)
  PauseMenu            UI Transform (Layer 10, Visible off), Sprite, UI Focus Scope (Modal)
    Resume             UI Transform (Relative), Sprite, Text, UI Button, UI Focus (Default Focus)
    Options            ... UI Button
    Quit               ... UI Button
```

```lua
function on_update(dt)
    local menu = scene.find("PauseMenu")
    if input.action_pressed("Pause") then menu:set_ui_visible(not menu:is_ui_visible()) end
    if menu:is_ui_visible() then
        if ui.cancelled() then menu:set_ui_visible(false) end        -- B / Escape closes it
        for _, e in ipairs(ui.events()) do
            if e.type == "clicked" and e.name == "Resume" then menu:set_ui_visible(false) end
        end
    end
end
```

Showing the menu takes focus to Resume and keeps the arrows inside it; hiding it gives focus back to whatever had it.
Nothing here needs a script to move focus.

## Next to hand-rolled buttons

A button written in Lua (`ui.is_clicked("Name")` plus `set_ui_color` every frame, as a HUD library
does) needs nothing from this chapter and is untouched by it: the widget code only looks at entities
that carry a widget component, and never writes a colour into the Sprite. The two can share a screen.

Two things to know when mixing them:

- Do not put a UI Button on an element whose colour a script already drives from hover state: both
  would tint it, the script by writing the Sprite colour and the button by multiplying on top.
- A script that drives a button's look by hand can still use the events. `hover_enter` / `hover_exit`
  replace polling `ui.is_hovered` every frame, and a disabled button drawn by the script is one
  `set_ui_disabled(true)` away from also being inert.

## Saved data

A scene stores the component with only the fields that differ from the defaults:

```yaml
UIButtonComponent:
  HoverTint: [1.5, 1.25, 1, 1]
  FadeTime: 0.2
  ClickSound: audio/click.wav
```

A default button is `UIButtonComponent: {}`. Toggles, sliders, scroll lists and text fields save the same way:

```yaml
UIToggleComponent:
  IsOn: true
  Group: difficulty
  Graphic: 5400720647465396
UISliderComponent:
  Min: 10
  Max: 20
  Step: 5
  Direction: 2
  Fill: 5400720647465397
  Handle: 5400720647465398
```

```yaml
UIScrollComponent:
  Horizontal: true
  Vertical: false
  ScrollSpeed: 45
  Content: 5400720647465400
  HScrollbar: 5400720647465401
UIScrollbarComponent:
  Handle: 5400720647465402
  MinHandleSize: 24
```

```yaml
UIFocusComponent:
  Down: 5400720647465410
  DefaultFocus: true
UIFocusScopeComponent: {}
```

```yaml
UITextFieldComponent:
  Text: "Ada"
  Placeholder: Your name
  MaxLength: 24
  Filter: 3
  TextEntity: 5400720647465420
  PlaceholderEntity: 5400720647465421
```

`Direction` is the enum's integer (0 Left To Right, 1 Right To Left, 2 Bottom To Top, 3 Top To Bottom); a text field's `Filter` is
0 Any, 1 Integer, 2 Decimal, 3 Alphanumeric (Password, PasswordChar, SelectAllOnFocus, SubmitClearsFocus, SelectionColor and Disabled are saved
when off their defaults). The editing state (caret, selection, scroll) is not saved.
A widget's `FocusTint` is saved only when it is not the default. The scroll offset is not saved. All of them are part of copy/paste, undo, prefab instances (the ids a scroll list and its
scrollbar name are rewritten like a slider's Fill and Handle) and the MCP tools like any component.

## Test

| Scene | Test | What it proves |
|---|---|---|
| `ui_terminal_test.Lscene` | `ui_terminal.lua` | Typing and the editing keys; Enter raising `submitted` with the line, the echo and the history (no repeats, no empty lines); Up / Down with the draft kept; Tab raising `complete` and `term_set_input` answering; game keys silent while focused; Escape and a click elsewhere letting go; Max Lines; wrapping a line with no break; PageUp / PageDown / the wheel and following the end; a colour tag and the prompt colour read from pixels; the caret drawn and placed by a click; Ctrl+L; Ctrl+Tab, Alt+key and F-keys reaching the game while focused; the cost of a gather over 2000 lines |
| `ui_terminal_test.Lscene` | `ui_terminal_links.lua` | Link spans: the hit test, hover colour and underline from pixels, the `link` event with its payload, an escaped payload, a link nested in a colour tag, a link wrapped over rows, focus and caret left alone, no event on a drag, a Read Only terminal |
| `ui_button_test.Lscene` | `ui_button.lua` | The event sequence for a click, a press dragged off and released elsewhere, hover in and out; tints read back from the final image against reference sprites (hover, pressed, resting, disabled, a slow fade); a disabled button emitting nothing and blocking the button beneath it; a plain sprite unaffected; a click on a label; every event seen once |
| `ui_toggle_test.Lscene` | `ui_toggle.lua` | A click toggles; the Graphic's visibility read from pixels without writing its Visible; a radio group's exclusivity with `value_changed` for both toggles; Allow Switch Off; `set_value` with and without notify, and on a group; a disabled toggle |
| `ui_slider_test.Lscene` | `ui_slider.lua` | Press-to-jump, drag inside and far outside the rect, release, all four directions, steps, Min and Max other than 0 and 1, Fill and Handle pixels at several values with their components untouched, `set_value` and `ui_adjust`, one event per change, a disabled slider |
| `ui_slider_test.Lscene` | `ui_widget_prefab.lua` | A slider object instantiated twice: the second copy's Fill and Handle references are remapped, so each slider drives its own |
| `ui_containers_test.Lscene` | `ui_containers.lua` | HBox, VBox and Grid rects with padding and spacing; main and cross alignment; Expand with ratios and Min Size; hidden and Ignore Layout children; Order, the id tie-break and Reverse Order; auto and fixed grid cells; Fit Content including a VBox of HBoxes; a slider inside an HBox; a toggle kept at its own size in a VBox, a centring VBox and an HBox; Clip Children; a container on a Full-Rect root at other viewport sizes; tiles spawned into a grid; the timing of a 200-child grid |
| `ui_scroll_smooth_test.Lscene` | `ui_scroll_smoothing.lua` | Smoothing 0.06 on the wheel: one notch only moves the target and the offset glides, halving the distance every 0.06 s whatever the frame time; notches in a row add up from the target, clamped at both ends; a drag, the scrollbar and `set_scroll` are direct and end a glide; `scroll_to` glides, and is `set_scroll` on a list with no Smoothing; one `value_changed` per frame the offset moves |
| `ui_scroll_test.Lscene` | `ui_scroll.lua` | Wheel steps and clamping at both ends with one event per change; no scroll, no drag-capture and no scrollbar when the content fits; a popup keeping the wheel; a plain click and a 4 px wobble still clicking a child; a drag scrolling and not clicking the button it started on (`released` 0); hit-testing following the offset and cut at the window; the thumb's size and drag, paging and jumping track clicks; `set_scroll`, normalized and `set_value` with and without `notify`; `ui_scroll_into_view`; a horizontal list, Shift + wheel and a horizontal notch, a diagonal drag; a list inside a list; content spawned and destroyed at run time; coasting that stops; a prefab with scroll references instantiated twice |
| `ui_focus_nav_test.Lscene` | `ui_focus_nav.lua` | The pure picking math (grid, irregular layout, in-line beats nearer, tolerance, wrap) through `ui_focus_pick`; nothing focuses with the project flag off; DefaultFocus; moving through a grid and an irregular layout with no wrap; an explicit neighbour; a slider adjusting with the direction along it and a vertical one swapped; a disabled button skipped; accept on a button, toggle and slider; cancel; the pointer taking focus and a still pointer not; the Focus Tint and the ring read from pixels; hidden and disabled focused widgets handing focus on; a modal scope confining, taking and returning focus; a scroll list scrolling to the focused row; wrap; a prefab's neighbour ids remapped on a second instance |
| `ui_textfield_test.Lscene` | `ui_textfield.lua` | A click starting an edit (caret, placeholder hidden, caret and selection read from pixels); typing through `input.inject_text`, UTF-8 (a two- and a three-byte character), one `text_changed` per character; Left / Right / Home / End / Backspace / Delete, with Ctrl by word and Shift selecting; Ctrl+A, replace, drag select, double and triple click; Ctrl+C / X / V; Enter (`value_changed`, `submitted`, the edit over) and a field that keeps the edit; Escape restoring the text without `submitted` or `cancel`; a click elsewhere committing and still clicking what it hit; Integer / Decimal / Alphanumeric filters and Max Length, typed and pasted; `set_value` filtered; a password's mask and its refused copy; a long text scrolling the caret into view and cut at the text area from pixels; the game's key bindings silent while editing and live again after; a disabled field; `ui_begin_edit` / `ui_end_edit`; focus navigation (arriving does not edit, accept does, the keys are the field's, a held Enter is not an accept again); the Steam Deck keyboard through a stand-in; a prefab with a field instantiated twice |
| `allcomponents_test.Lscene` | `serializer.lua` | A button, a toggle, a slider, a scroll list and a scrollbar with every field off its default round-trip, and a default one saves an empty block |

[Testing](../07-projects-and-tools/02-testing.md) explains how to run them.

## See also

- [HUD](01-hud.md): UI Transform, Sprite, Text and the hit test this sits on
- [HUD interaction](../06-scripting/02-lua/api/ui/01-hud-interaction.md): the `ui` table, including `events` and `changed`
- [HUD (entity)](../06-scripting/02-lua/api/entity/10-hud.md): `set_ui_disabled`, `get_value`, `ui_begin_edit` and the rest of the entity calls
- [UI Designer](02-ui-designer.md): laying widgets out on a canvas, the Create palette that makes them wired up, and how the Designer treats what a container or a widget places
