---
title: "HUD elements"
---

An element's look, its text, and where it sits.

An element is an entity carrying a `UITransformComponent` plus either a sprite or a text
component. These calls reach it by entity id, so a behaviour that holds a HUD element's id can
drive it directly.

**Every call here is a silent no-op on an entity missing the component it needs.** That is the
contract the Lua facade already gives a script: a HUD script does not necessarily know what it is
attached to, and asking the wrong entity should not be a failure.

## Units

Positions and sizes are **UI units**: the pixels a HUD is authored in, scaled to the real
viewport by `ProjectConfig::UIReferenceHeight` when that is set. So a layout authored for a
1080p reference height looks the same on a Steam Deck without any per-resolution maths in your
code.

`GetViewportSize` (see [HUD interaction](02-hud-interaction.md#getviewportsize)) is how a
behaviour asks what it is actually being drawn into.

## Text

### `SetText`

```c
void (*SetText)(LAB_EntityId entity, const char* text);
```

```cpp
void LABNative::UI::SetText(LAB_EntityId entity, const char* text);
```

```cpp
LABNative::UI::SetText(scoreLabel, "1240");
```

### `GetText`

```c
int (*GetText)(LAB_EntityId entity, char* buffer, size_t capacity);
```

```cpp
int LABNative::UI::GetText(LAB_EntityId entity, char* buffer, size_t capacity);
```

The element's own text, copied into your buffer and answered as the length written, or -1 with no
bound scene.

A caller that formats its own text every frame usually does not need this, but it is how an
element's authored text is read back, and how a behaviour confirms what it just set.

### `MeasureText`

```c
void (*MeasureText)(LAB_EntityId entity, const char* overrideText, int hasOverride,
	float* outWidth, float* outHeight);
```

```cpp
void LABNative::UI::MeasureText(LAB_EntityId entity, const char* overrideText, int hasOverride,
	float* outWidth, float* outHeight);
```

The laid-out size of the element's text in UI units, wrapped at the element's width when its text
wraps.

A null `overrideText`, or `hasOverride` 0, measures the component's own text. Otherwise it
measures the string you pass, which is how a behaviour sizes a panel around text it is about to
set rather than around what is there now:

```cpp
float width = 0.0f, height = 0.0f;
LABNative::UI::MeasureText(panel, message.c_str(), 1, &width, &height);
LABNative::UI::SetSize(panel, width + 24.0f, height + 16.0f);
LABNative::UI::SetText(panel, message.c_str());
```

## Visibility and colour

### `SetVisible` / `IsVisible`

```c
void (*SetVisible)(LAB_EntityId entity, int visible);
int (*IsVisible)(LAB_EntityId entity);
```

```cpp
void LABNative::UI::SetVisible(LAB_EntityId entity, bool visible);
bool LABNative::UI::IsVisible(LAB_EntityId entity);
```

A hidden element is not drawn and **does not receive interaction**: hover, press and click all
report nothing for it. So showing and hiding is the whole mechanism for a menu.

### `SetAlpha`

```c
void (*SetAlpha)(LAB_EntityId entity, float alpha);
```

```cpp
void LABNative::UI::SetAlpha(LAB_EntityId entity, float alpha);
```

The element's overall opacity, 0 to 1. Unlike `SetColor`, there is no "has" flag here: an alpha
is always an alpha.

This multiplies with whatever the component already has, so fading out and back in from a value
of 1 is the shape to use rather than writing absolute values from two places.

### `SetColor`

```c
void (*SetColor)(LAB_EntityId entity, LAB_Vec3 rgb, float alpha, int hasAlpha);
```

```cpp
void LABNative::UI::SetColor(LAB_EntityId entity, LAB_Vec3 rgb, float alpha, bool hasAlpha);
```

Colours the element's text **and** its sprite, whichever of the two it has.

`alpha` only replaces the component's own alpha when `hasAlpha` says so. That has-flag pattern
runs through every colour call in this area, and the reason is the same each time: a caller that
wants to change one of two things should not have to know the other one's current value in order
to leave it alone.

```cpp
// Recolour, leave the alpha where it is.
LABNative::UI::SetColor(healthBar, LAB_Vec3{ 1.0f, 0.2f, 0.2f }, 0.0f, false);
```

## Sizing and layout

### `SetSize` / `GetSize`

```c
void (*SetSize)(LAB_EntityId entity, float width, float height);
void (*GetSize)(LAB_EntityId entity, float* outWidth, float* outHeight);
```

```cpp
void LABNative::UI::SetSize(LAB_EntityId entity, float width, float height);
void LABNative::UI::GetSize(LAB_EntityId entity, float* outWidth, float* outHeight);
```

The element's size in UI units. Both out pointers on `GetSize` are required; pass two locals.

On an axis anchored to stretch with its parent, the parent decides the size: `SetSize` leaves that
axis alone and `GetSize` reports what it resolves to (see the Lua [HUD](../../../02-lua/api/entity/10-hud.md)
page for `set_ui_anchors`, which has no native counterpart yet).

### `SetOffset` / `GetOffset`

```c
void (*SetOffset)(LAB_EntityId entity, float x, float y);
void (*GetOffset)(LAB_EntityId entity, float* outX, float* outY);
```

```cpp
void LABNative::UI::SetOffset(LAB_EntityId entity, float x, float y);
void LABNative::UI::GetOffset(LAB_EntityId entity, float* outX, float* outY);
```

The element's offset from its anchor. This is the call for animation: nudging a label, sliding a
panel in, shaking a health bar.

### `SetAnchor` / `GetAnchor`

```c
void (*SetAnchor)(LAB_EntityId entity, float x, float y);
void (*GetAnchor)(LAB_EntityId entity, float* outX, float* outY);
```

```cpp
void LABNative::UI::SetAnchor(LAB_EntityId entity, float x, float y);
void LABNative::UI::GetAnchor(LAB_EntityId entity, float* outX, float* outY);
```

Where the element is pinned in the viewport, as fractions.

| | |
|---|---|
| `0` | Top edge |
| `1` | Bottom edge |
| `x` | 0 left, 1 right |
| `y` | 0 top, 1 bottom, and **increasing y goes down** |

So `(0, 0)` is the top-left corner and `(0.5, 1)` is bottom centre, which is where a subtitle
belongs. The y axis runs the opposite way to the world's, because it is screen space.

`SetAnchor` pins a single point (both corners of the element's anchor rectangle), which makes any
stretched axis a point again. `GetAnchor` answers the rectangle's min corner.

### `SetPivot`

```c
void (*SetPivot)(LAB_EntityId entity, float x, float y);
```

```cpp
void LABNative::UI::SetPivot(LAB_EntityId entity, float x, float y);
```

The point about which the element is positioned and scaled, as fractions of its own size.
`(0.5, 0.5)` is its centre, `(0, 0)` its top-left.

An element that should grow from its centre needs a centre pivot, and one that should grow
rightwards needs a left one.

### `SetLayer` / `GetLayer`

```c
void (*SetLayer)(LAB_EntityId entity, int layer);
int (*GetLayer)(LAB_EntityId entity);
```

```cpp
void LABNative::UI::SetLayer(LAB_EntityId entity, int layer);
int LABNative::UI::GetLayer(LAB_EntityId entity);
```

Draw order, and the tie-break for which element a hit lands on where two overlap. Higher is on
top.

## Sprite and text styling

### `SetTexture`

```c
void (*SetTexture)(LAB_EntityId entity, const char* path);
```

```cpp
void LABNative::UI::SetTexture(LAB_EntityId entity, const char* path);
```

Points a sprite element at a texture, project-relative, the same string form a `.Lscene` holds.

A new path loads the texture if it is not already in the cache, so switching an icon every frame
is a disk and upload cost rather than a free operation. A small set of icons swapped between is
fine; a generated path per frame is not.

### `SetRadius`

```c
void (*SetRadius)(LAB_EntityId entity, float radius);
```

```cpp
void LABNative::UI::SetRadius(LAB_EntityId entity, float radius);
```

The corner radius of a sprite element, in UI units. `0` is a plain rectangle.

### `SetBorder`

```c
void (*SetBorder)(LAB_EntityId entity, float width, LAB_Vec3 rgb, int hasColor, float alpha, int hasAlpha);
```

```cpp
void LABNative::UI::SetBorder(LAB_EntityId entity, float width, LAB_Vec3 rgb, bool hasColor, float alpha, bool hasAlpha);
```

The border width, and its colour and alpha where the has flags say so.

The **width always applies**; the colour and alpha only where theirs do. That is the same rule
`SetColor` follows, and it is what lets a caller change the width alone without recolouring the
border by accident:

```cpp
// Thicker, same colour.
LABNative::UI::SetBorder(selectedTile, 4.0f, LAB_Vec3{}, false, 0.0f, false);
```

### `SetFontSize` / `GetFontSize`

```c
void (*SetFontSize)(LAB_EntityId entity, float size);
float (*GetFontSize)(LAB_EntityId entity);
```

```cpp
void LABNative::UI::SetFontSize(LAB_EntityId entity, float size);
float LABNative::UI::GetFontSize(LAB_EntityId entity);
```

The text element's font size in UI units. Useful for a scale effect that should not change the
element's box: animating the font size grows the glyphs in place, while animating the size moves
the layout.

### `SetTextOutline`

```c
void (*SetTextOutline)(LAB_EntityId entity, float width, LAB_Vec3 rgb, int hasColor, float alpha, int hasAlpha);
```

```cpp
void LABNative::UI::SetTextOutline(LAB_EntityId entity, float width, LAB_Vec3 rgb, bool hasColor, float alpha, bool hasAlpha);
```

An outline around the glyphs, for text that has to stay readable over a moving background. Same
has-flag rule as `SetBorder`: the width always applies, the colour and alpha only where theirs do.

## A complete example

A score label that pops when it changes.

```cpp
class ScoreDisplay : public LABNative::Behaviour
{
public:
	void OnUpdate(float deltaTime) override
	{
		if (m_Pop > 0.0f)
		{
			m_Pop = std::max(0.0f, m_Pop - deltaTime * 3.0f);
			LABNative::UI::SetFontSize(GetEntity(), m_BaseSize * (1.0f + 0.5f * m_Pop));
		}
	}

	void OnScoreChanged(int score)
	{
		LABNative::UI::SetText(GetEntity(), std::to_string(score).c_str());
		m_Pop = 1.0f;
	}

private:
	float m_BaseSize = 32.0f;
	float m_Pop = 0.0f;
};
```

Note `GetEntity()` throughout: these are free functions in the `LABNative::UI` namespace rather
than methods, because a HUD element is usually a different entity from the behaviour driving it.
