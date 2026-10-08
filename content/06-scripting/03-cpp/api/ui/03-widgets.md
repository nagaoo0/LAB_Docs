---
title: "HUD widgets"
---

Buttons, toggles, sliders, scroll lists and text fields from a native module: the widget events,
widget values, a text field's text, the disabled flag and keyboard / gamepad focus. These are the C++
form of the widget calls in [UI Widgets](../../../../05-ui/03-ui-widgets.md), added at ABI minor 12, and every
one answers the way the Lua call of the same name does. A module built against an older minor still
loads and simply does not have them: ask `LAB_UI_API_HAS_ABI(api->UI, GetFieldText)` first when that
matters, which the `LABNative::UI` wrappers already do.

## `GetEventCount` / `GetEvent` / `HasEvent`

```c
int (*GetEventCount)(void);
int (*GetEvent)(int index, LAB_EntityId* outEntity, int* outType, float* outValue);
```

```cpp
int  LABNative::UI::GetEventCount();
bool LABNative::UI::GetEvent(int index, LAB_EntityId& entity, int& type, float& value);
bool LABNative::UI::HasEvent(LAB_EntityId entity, int type, float* outValue = nullptr);
```

The events the HUD raised in the last frame, oldest first: what `ui.events()` returns to a Lua
script. `type` is a `LAB_UIEventType` (`LAB_UI_EVENT_CLICKED`, `LAB_UI_EVENT_SUBMITTED`,
`LAB_UI_EVENT_VALUE_CHANGED`, ...), and `value` is the event's own number: a slider's new value, a text
field's new length. An event whose entity has since been destroyed answers false.

An event is seen exactly once, by the first `OnUpdate` after the frame that raised it, so a click is
never missed and never handled twice. The events only exist for entities that carry a widget component;
a plain sprite with a UI Transform raises none, which is what `IsClicked` is for.

```cpp
void OnUpdate(float) override
{
    // Enter inside the address field joins, the same as pressing the button.
    if (UI::HasEvent(m_AddressField, LAB_UI_EVENT_SUBMITTED))
        Join(UI::GetFieldText(m_AddressField));
}
```

## `GetValue` / `SetValue`

```c
int  (*GetValue)(LAB_EntityId entity, float* outValue);
void (*SetValue)(LAB_EntityId entity, float value, int notify);
```

```cpp
bool LABNative::UI::GetValue(LAB_EntityId entity, float& value);
void LABNative::UI::SetValue(LAB_EntityId entity, float value, bool notify = true);
```

A toggle's 1 or 0, a slider's value, a scroll list's normalised position. `GetValue` answers false for
anything without such a number (a button, a text field, an entity that is not a widget).
`SetValue` clamps and snaps like the widget does, and raises `value_changed` for the next frame unless
`notify` is false.

## Text fields

```c
int  (*GetFieldText)(LAB_EntityId entity, char* buffer, size_t capacity);
void (*SetFieldText)(LAB_EntityId entity, const char* text, int notify);
int  (*BeginEdit)(LAB_EntityId entity);
int  (*EndEdit)(LAB_EntityId entity, int commit);
int  (*IsEditing)(LAB_EntityId entity);
```

```cpp
std::string LABNative::UI::GetFieldText(LAB_EntityId entity, size_t capacity = 1024);
bool        LABNative::UI::IsTextField(LAB_EntityId entity);
void        LABNative::UI::SetFieldText(LAB_EntityId entity, const char* text, bool notify = true);
bool        LABNative::UI::BeginEdit(LAB_EntityId entity);
bool        LABNative::UI::EndEdit(LAB_EntityId entity, bool commit = true);
bool        LABNative::UI::IsEditing(LAB_EntityId entity);
```

`GetFieldText` is the text the field holds, not what is drawn: a password field answers the real
characters. The C ABI form answers the length written, or -1 for an entity that is not a text field,
and copies into the caller's buffer like every string on this boundary. The C++ wrapper owns its
buffer and answers an empty string for a non-field, which `IsTextField` tells apart from an empty field.

`SetFieldText` is held to the field's filter and maximum length, as typing is. While a field is being
edited the game does not get the keyboard (`input` answers as if nothing were held); the editing state is
the field's.

## `SetDisabled` / `IsDisabled`

```cpp
void LABNative::UI::SetDisabled(LAB_EntityId entity, bool disabled);
bool LABNative::UI::IsDisabled(LAB_EntityId entity);
```

A widget's Disabled flag: the disabled tint, no events, not hovered or clicked, still blocking the
pointer. A no-op, and false, on an entity that is not a widget.

## `GetFocused` / `SetFocus`

```cpp
LAB_EntityId LABNative::UI::GetFocused();
bool         LABNative::UI::SetFocus(LAB_EntityId entity);
```

Keyboard and gamepad focus, when the project turns focus navigation on (`UIFocusNavigation` in the
`.lab`); with it off, `GetFocused` answers 0 and `SetFocus` false. `SetFocus(0)` clears focus. It
answers false when the widget cannot take focus now (hidden, disabled, outside the modal scope).
