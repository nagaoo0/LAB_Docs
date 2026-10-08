---
title: "Input"
---

Keyboard, project actions and the mouse.

Everything here reads the same state the engine's own gameplay does, which is worth knowing for
one reason: input reaches gameplay through ImGui's event queue, not through the platform layer.
A key injected into that queue (by the MCP server's `send_input`, for instance) is therefore
seen here exactly as a real key press is, and edge detection comes free with it.

## Keyboard

```c
int (*IsKeyDown)(const char* key);      // held this frame
int (*IsKeyPressed)(const char* key);   // went down this frame
```

```cpp
bool LABNative::IsKeyDown(const char* key);
bool LABNative::IsKeyPressed(const char* key);
```

```cpp
if (LABNative::IsKeyDown("W") || LABNative::IsKeyDown("Up"))
	MoveForward();
```

**Key names resolve through one shared table**, the same one a Lua script's `input.is_key_down`
and the MCP server's `send_input` use. That sharing is the point: an agent pressing `"W"` and a
behaviour reading `"W"` cannot disagree about spelling, because both go through the same lookup.

Names are the ones `ImGui::GetKeyName` produces. The common ones:

| | |
|---|---|
| Letters and digits | `"A"` to `"Z"`, `"0"` to `"9"` |
| Arrows | `"Up"`, `"Down"`, `"Left"`, `"Right"` |
| Modifiers | `"LeftShift"`, `"RightShift"`, `"LeftCtrl"`, `"LeftAlt"` |
| Common keys | `"Space"`, `"Enter"`, `"Escape"`, `"Tab"`, `"Backspace"` |
| Function keys | `"F1"` to `"F12"` |

**An unknown name is inert, not an error.** It never fires, and nothing is logged, which is the
same tolerance a typo has always had. If a key never triggers, check its spelling against the
Keyboard Reference rather than looking for an error.

Prefer the project's **actions** over raw keys where you have the choice, so the game stays
remappable.

## Project actions

```c
int (*ActionDown)(const char* action);      // held
int (*ActionPressed)(const char* action);   // went down this frame
float (*ActionAxis)(const char* action);    // -1 to 1
```

```cpp
bool LABNative::ActionDown(const char* action);
bool LABNative::ActionPressed(const char* action);
float LABNative::ActionAxis(const char* action);
```

An action is a named binding from the project's own table, edited in the editor's **Input
Actions** panel and stored in the `.lab` file. `"Jump"`, `"Move Right"`, `"Fire"`: whatever the
project defines.

```cpp
const float forward = LABNative::ActionAxis("Move Forward");
SetPosition(GetPosition() + LAB_Vec3{ 0.0f, forward * 4.0f * deltaTime, 0.0f });

if (LABNative::ActionPressed("Jump"))
	AddImpulse(LAB_Vec3{ 0.0f, 0.0f, 6.0f });
```

Three things are true of these, and all three come from the same design:

- **The name is resolved live, on every call.** There is no compiled binding cache, so editing
  the project's binding table takes effect immediately, with no reload and no rebuild.
- **An unbound or unknown name is inert**, exactly like a typo'd raw key. Nothing errors.
- **`ActionPressed`'s edge is tracked by the engine**, refreshed once per tick for *every* action
  the project defines, not only the ones a behaviour happened to ask about. That matters: an
  action nobody queried this tick still has its "previous" state advanced, so a later query does
  not see a stale edge from while nobody was looking.

`ActionAxis` is the combined value of whatever the action's bindings are, in `-1..1`. A key pair
gives -1, 0 or 1; a gamepad stick gives everything between.

## Mouse

```c
LAB_Vec3 (*GetMousePosition)(void);
int (*IsMouseDown)(int button);
int (*IsMousePressed)(int button);
int (*IsMouseReleased)(int button);
```

```cpp
LAB_Vec3 LABNative::GetMousePosition();
bool LABNative::IsMouseDown(int button);
bool LABNative::IsMousePressed(int button);
bool LABNative::IsMouseReleased(int button);
```

| Button | |
|---|---|
| `0` | Left |
| `1` | Right |
| `2` | Middle |

`GetMousePosition` answers window pixels in `x` and `y`, with `z` always 0. It is the OS cursor
position, not the viewport's own coordinates, which matters in the editor where the viewport is
one panel among several.

```cpp
const LAB_Vec3 mouse = LABNative::GetMousePosition();
if (LABNative::IsMousePressed(0))
	LABNative::LogInfo("clicked");
```

**A scripted pointer replaces the mouse entirely.** When a virtual pointer is installed (see
[HUD interaction](../ui/02-hud-interaction.md#the-virtual-pointer), or the Lua
`ui.set_virtual_pointer`), these calls read that pointer's position and buttons instead of the
real device's. That is what lets a test or a replay drive a HUD, and it is why a behaviour needs
no special case to be testable.

## A complete example

Movement and jumping, using actions rather than raw keys, with the jump on the physics clock.

```cpp
class Mover : public LABNative::Behaviour
{
public:
	float Speed = 4.0f;
	float JumpSpeed = 6.0f;

	void OnUpdate(float deltaTime) override
	{
		LAB_Vec3 position = GetPosition();
		position.y += LABNative::ActionAxis("Move Forward") * Speed * deltaTime;
		position.x += LABNative::ActionAxis("Move Right") * Speed * deltaTime;
		SetPosition(position);
	}

	void OnFixedUpdate(float fixedDeltaTime) override
	{
		// The impulse belongs on the physics clock: it changes a body the solver owns.
		if (LABNative::ActionPressed("Jump"))
			AddImpulse(LAB_Vec3{ 0.0f, 0.0f, JumpSpeed });
	}
};
```

## What is not here

There is no gamepad call on this table. A module reaches the pad through the project's actions,
which is the better answer anyway: a gamepad binding and a keyboard binding for the same action
are one `ActionAxis` call, and the remapping panel covers both.

There is also no mouse capture, no wheel and no delta on the ABI. A first-person camera in a
module wants those, and the Lua facade has them; a module wanting one today drives the camera
from the viewport's own behaviour, or asks for the call in the ABI.
