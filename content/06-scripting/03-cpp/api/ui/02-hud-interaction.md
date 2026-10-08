---
title: "HUD interaction"
---

Hover, press and click, by entity or by name, plus the virtual pointer a replay drives.

Hit testing is updated **once per frame from the mouse position**. There is no separate event or
signal system: a behaviour asking `IsClicked` is asking about this frame, the same way it asks
about a key.

## `SetInteractive`

```c
void (*SetInteractive)(LAB_EntityId entity, int interactive);
```

```cpp
void LABNative::UI::SetInteractive(LAB_EntityId entity, bool interactive);
```

Whether the element takes part in hit testing at all.

An element that is not interactive is never hovered, pressed or clicked, whatever the pointer
does. That is what makes a decorative panel not swallow clicks meant for a button behind it, and
what makes `PointerOver` a meaningful test.

## `IsHovered` / `IsPressed` / `IsClicked`

```c
int (*IsHovered)(LAB_EntityId entity);
int (*IsPressed)(LAB_EntityId entity);
int (*IsClicked)(LAB_EntityId entity);
```

```cpp
bool LABNative::UI::IsHovered(LAB_EntityId entity);
bool LABNative::UI::IsPressed(LAB_EntityId entity);
bool LABNative::UI::IsClicked(LAB_EntityId entity);
```

| Call | True when |
|---|---|
| `IsHovered` | The pointer is over that element right now |
| `IsPressed` | Hovered, and the mouse button is currently down |
| `IsClicked` | For exactly one frame, when a press and a release both landed on that element |

`IsClicked` is the usual button semantics: pressing on one element and releasing on another is
not a click on either. That is what stops a click being stolen by whatever the pointer happened
to pass over on the way.

```cpp
if (LABNative::UI::IsClicked(GetEntity()))
	StartGame();
```

Where two interactive elements overlap, **only the topmost wins**, and the tie-break is draw
order: a `UITransformComponent`'s `Layer`. So a button on a panel is clickable and the panel
underneath it is not.

## The by-name forms

```c
int (*IsHoveredByName)(const char* name);
int (*IsPressedByName)(const char* name);
int (*IsClickedByName)(const char* name);
```

```cpp
bool LABNative::UI::IsHoveredByName(const char* name);
bool LABNative::UI::IsPressedByName(const char* name);
bool LABNative::UI::IsClickedByName(const char* name);
```

The same three questions by entity **tag** rather than by id, using the same name lookup
`FindEntityByName` and a Lua script's `scene.find` use.

A behaviour usually holds the parent of a HUD, or is on an entity that knows the *name* of a
button from the Properties panel rather than an id it never stored. This is that spelling:

```cpp
if (LABNative::UI::IsClickedByName("PlayButton"))
	LABNative::LogInfo("play pressed");
```

## `GetHoveredEntity` / `GetPressedEntity` / `GetClickedEntity`

```c
LAB_EntityId (*GetHoveredEntity)(void);
LAB_EntityId (*GetPressedEntity)(void);
LAB_EntityId (*GetClickedEntity)(void);
```

```cpp
LAB_EntityId LABNative::UI::GetHoveredEntity();
LAB_EntityId LABNative::UI::GetPressedEntity();
LAB_EntityId LABNative::UI::GetClickedEntity();
```

Whatever the HUD's own hit test last saw the pointer over, pressing or clicking, as an entity id
or **0** for none.

These are the form a dispatcher wants: one behaviour on the player handling every HUD event,
rather than one per button.

```cpp
if (const LAB_EntityId clicked = LABNative::UI::GetClickedEntity())
{
	if (clicked == m_PlayButton)   StartGame();
	else if (clicked == m_QuitButton) Quit();
}
```

**A hovered parent counts for its descendants too.** A pointer over a child of an interactive
element hovers the parent, which is what makes a panel behave as one thing rather than as its
parts.

## `GetRect`

```c
void (*GetRect)(LAB_EntityId entity, float* outX, float* outY, float* outWidth, float* outHeight);
```

```cpp
void LABNative::UI::GetRect(LAB_EntityId entity, float* outX, float* outY, float* outWidth, float* outHeight);
```

The element's final rect on screen, resolved against the current viewport: x, y, width and
height. All four out pointers are required.

**It resolves the whole HUD**, so read it on an event rather than every frame. The rect is where
the element *ended up* after anchors, pivots, offsets, parents and the viewport scale have all
had their say, which is exactly what a behaviour needs for something that is not a HUD element:
drawing a cursor to a slot, moving a 3D highlight onto a label, or aiming a world-space thing at
a screen position.

```cpp
float x = 0, y = 0, w = 0, h = 0;
LABNative::UI::GetRect(selectedSlot, &x, &y, &w, &h);
// ... place a 3D marker at the centre of that slot: (x + w * 0.5, y + h * 0.5)
```

## `PointerOver`

```c
int (*PointerOver)(void);
```

```cpp
bool LABNative::UI::PointerOver();
```

1 while the pointer is over any interactive HUD element.

This is the test a **world click** makes before acting, so that clicking a button does not also
click what is behind it in the 3D scene. A behaviour that acts on a mouse press, on an entity
that is not the HUD, wants this guard:

```cpp
if (LABNative::IsMousePressed(0) && !LABNative::UI::PointerOver())
	SelectWhateverIsUnderTheCursor();
```

Without it, a click that lands on a HUD button also reaches the world, which is the classic
"clicking a menu item also fires the weapon" bug.

## `GetViewportSize`

```c
void (*GetViewportSize)(float* outWidth, float* outHeight, float* outScale);
```

```cpp
void LABNative::UI::GetViewportSize(float* outWidth, float* outHeight, float* outScale);
```

The game viewport's own size **in UI units**, then the scale from UI units to pixels.

So `outWidth * outScale` is the viewport in real pixels. A behaviour doing its own layout maths
wants the UI-unit size, since that is the space every other call here works in.

## The virtual pointer

```c
void (*SetVirtualPointer)(float u, float v, uint32_t buttons, float wheel);
void (*ClearVirtualPointer)(void);
```

```cpp
void LABNative::UI::SetVirtualPointer(float u, float v, uint32_t buttons, float wheel);
void LABNative::UI::ClearVirtualPointer();
```

A scripted pointer standing in for the mouse, so a test or a replay can drive a HUD with no mouse
at all.

| Argument | |
|---|---|
| `u`, `v` | Across the viewport, 0 to 1. Not pixels, and not UI units |
| `buttons` | A bit mask: 1 left, 2 right, 4 middle |
| `wheel` | Wheel steps, in notches |

```cpp
// Hover the middle of the screen and press the left button.
LABNative::UI::SetVirtualPointer(0.5f, 0.5f, 1, 0.0f);
```

Three things to know:

- **It takes effect on the next frame**, like a real mouse move, so a caller that sets it and
  immediately tests a click sees the previous frame's state.
- **It replaces the mouse entirely.** Everything that reads the mouse, in the engine and in Lua,
  reads the virtual pointer while one is installed, not the real device. That is what makes a
  replayed session reproducible.
- **Set it once and clear it when the replay is over.** A virtual pointer left installed is a
  session whose mouse no longer works, with nothing on screen to explain it.

## A complete example

A pause menu, driven from one behaviour on the menu's root.

```cpp
void OnUpdate(float deltaTime) override
{
	if (!m_Paused)
		return;

	if (LABNative::UI::IsClickedByName("ResumeButton"))
	{
		m_Paused = false;
		LABNative::UI::SetVisible(m_MenuRoot, false);
		return;
	}

	if (LABNative::UI::IsClickedByName("QuitButton"))
		Destroy();
}
```

Note that the element names are the only thing this behaviour has to agree with the scene about,
which is the point of the by-name forms: a HUD can be rebuilt in the editor without touching the
code, as long as the tags stay.
