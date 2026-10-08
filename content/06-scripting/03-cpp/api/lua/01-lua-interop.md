---
title: "Lua interop"
---

A behaviour and a Lua script are two languages over one scene, and a project that mixes them
needs each to reach the other. This is that boundary, and it is deliberately small: one value
type, one call into Lua, and one registration to be reachable from Lua.

## `LAB_Variant`

The value that crosses.

```c
typedef enum LAB_VariantType
{
	LAB_VARIANT_NONE = 0,   // no value: nil in Lua
	LAB_VARIANT_NUMBER = 1, // both Lua's only number type and every numeric ABI entry
	LAB_VARIANT_BOOL = 2,   // an int, because this header is plain C
	LAB_VARIANT_VEC3 = 3,
	LAB_VARIANT_STRING = 4,
	LAB_VARIANT_ENTITY = 5  // an entity id
} LAB_VariantType;

typedef struct LAB_Variant
{
	int Type;              // the only thing that says which member is meaningful

	union
	{
		double Number;
		int Bool;
		LAB_Vec3 Vec3;
		const char* String;
		LAB_EntityId Entity;
	};

	char* StringBuffer;    // where a string *return* goes
	size_t StringCapacity;
} LAB_Variant;
```

A tag plus the value, in one fixed-size struct two independently compiled C++ compilers lay out
the same way. That is the whole reason it is not a `std::variant`, and not the engine's own
`AI::Value`: either of those would tie the module to the engine's compiler, its STL version and
its exact type layout, which is the thing this boundary exists to avoid.

**Nothing zeroes a member it did not use.** A caller that reads the wrong one reads whatever was
left there rather than a reassuring zero, so check `Type` first. That is the entire contract,
and it is worth following literally.

### Strings, and who owns them

The rule every string on this boundary follows: **a string that comes back is copied into a
buffer the caller owns, and a string argument is a pointer that only has to be valid for the
length of the call it was passed to.**

| | |
|---|---|
| An argument | Read from the union's `String`. The caller owns it and keeps it alive until the call returns, and no longer |
| A return | Copied into `StringBuffer` when the caller supplied one, truncated rather than overrun and always null-terminated. `String` then points at that buffer |
| No `StringBuffer` | The caller gets the type and no text, which is the caller's own choice rather than a lost value |

Size it to what you want. A returned string longer than the buffer is truncated, not refused.

## Calling Lua: `CallFunction`

```c
int (*CallFunction)(const char* name, const LAB_Variant* args, int argCount, LAB_Variant* out);
```

```cpp
bool LABNative::Lua::CallFunction(const char* name, const LAB_Variant* args, int argCount, LAB_Variant* out);
```

Calls a Lua function by name, and answers 1 on success.

The name is looked up among the Lua state's **globals**, which is the same place a node graph's
**Call Lua Function** node looks. So a graph and a behaviour reach the same functions, and a
project does not have to choose which one may call a given script function.

```cpp
char buffer[256];
LAB_Variant out{};
out.StringBuffer = buffer;
out.StringCapacity = sizeof(buffer);

LAB_Variant args[2]{};
args[0].Type = LAB_VARIANT_STRING;
args[0].String = "Score";
args[1].Type = LAB_VARIANT_NUMBER;
args[1].Number = 42.0;

if (LABNative::Lua::CallFunction("on_score_changed", args, 2, &out))
{
	if (out.Type == LAB_VARIANT_STRING)
		LABNative::LogInfo(buffer);
	else if (out.Type == LAB_VARIANT_BOOL && out.Bool)
		LABNative::LogInfo("accepted");
}
```

**The call is protected.** A Lua function that errors is a logged failure and a 0 answer, not an
exception crossing the boundary and taking the process down.

It answers 0 for: no bound scene, no running Lua state, an unknown name, or a call that threw.
Those are four different situations with one answer, so the log is where to look when a call
does not land. `HasFunction` is how a caller tells "no such function" from the others.

Arguments are converted **one at a time from their own `Type`**, so what the callee receives is
the shape the caller tagged rather than a reinterpretation of the union.

`out`, when non-null, receives the function's **first return value**. A returned string is copied
into `out->StringBuffer`. A return type this boundary cannot carry, a table or a function, arrives
as `LAB_VARIANT_NONE`.

## Being called from Lua: `RegisterFunction`

```c
int (*RegisterFunction)(void* self, const char* name, LAB_LuaFunctionFn fn);
typedef void (*LAB_LuaFunctionFn)(const LAB_Variant* args, int argCount, LAB_Variant* out);
```

On the registry handed to `LAB_ModuleInit`:

```cpp
static void SetDifficulty(const LAB_Variant* args, int argCount, LAB_Variant* out)
{
	if (argCount < 1 || args[0].Type != LAB_VARIANT_NUMBER)
	{
		out->Type = LAB_VARIANT_BOOL;
		out->Bool = 0;
		return;
	}

	g_Difficulty = static_cast<int>(args[0].Number);

	out->Type = LAB_VARIANT_BOOL;
	out->Bool = 1;
}

extern "C" LAB_NATIVE_EXPORT int LAB_ModuleInit(const LAB_EngineAPI* api, LAB_ModuleRegistry* registry)
{
	LABNative::Init(api);
	LAB_REGISTER_BEHAVIOUR(registry, Mover);
	registry->RegisterFunction(registry->Self, "set_difficulty", &SetDifficulty);
	return LAB_NATIVE_API_VERSION;
}
```

```lua
if native.call("set_difficulty", 3) then
    log.info("difficulty set")
end
```

Answers 1 on success, 0 for a null or empty name, a null function, or a name already registered.

**`name` has to outlive the module**, like a behaviour's own name. A string literal or a
namespace-scope constant is correct; anything built per call is not.

`out` is **always a real `LAB_Variant` to fill**. Leaving `out->Type` at `LAB_VARIANT_NONE` means
the function returned nothing, which Lua reads as nil. That is the honest answer rather than a
distinction Lua makes anywhere else: a function with no return value and a function that does not
exist are both "no value" to a caller looking only at the result.

### A function must not throw

The engine catches at the boundary, but the consequence is different from a behaviour's: a throw
here is *a Lua script's call* failing, which the script sees as an error, rather than a behaviour
being disabled. Do not wrap a module function in `try`/`catch` of its own either: the engine is
the single catch on this path, and swallowing the exception means nobody learns the call failed.

## `HasFunction`

```c
int (*HasFunction)(const char* name);
```

```cpp
bool LABNative::Lua::HasFunction(const char* name);
```

1 when the **loaded module** registered a function by this name.

So a caller can ask before calling rather than reading meaning out of a 0 answer, which is the
difference between "no such function" and "the function ran and failed":

```cpp
if (LABNative::Lua::HasFunction("on_score_changed"))
	LABNative::Lua::CallFunction("on_score_changed", args, 2, &out);
```

From Lua, a test can list what is registered with `native_function_names()`, which is how "why
did my `native.call` do nothing" is answered without a debugger.

## Which direction to use

The two directions are not symmetric, and the asymmetry is worth designing around.

| | Module calling Lua | Lua calling a module |
|---|---|---|
| Cost | A lookup and a call, per call | The same |
| Coupling | The module needs a function's name | The module needs to have registered a name |
| Best for | Notifying: an event happened, react to it | Commands: set this, do that |

The usual shape is Lua for the one-off and the module for the system: a module announces
`on_wave_complete` and the level's Lua script decides what that means for this level, while the
script calls back into the module's own `spawn_wave` when it wants one. Neither side has to know
the other's internals, only a name.

## What does not cross

A value that is a **table, a function or a userdata** cannot. A Lua function returning one hands
the module `LAB_VARIANT_NONE`, and there is no way to pass one in either.

A module therefore cannot hand Lua an object, and Lua cannot hand it one. What crosses is
numbers, bools, strings, vectors and entity ids, which is enough for the shapes above and not
enough for anything object-shaped.

There is also no way for a module to read or write a Lua global, and no way to run a string as
code: `CallFunction` looks up a *function* by name among the globals and calls it. The scripting
sandbox is what makes a shipped game's Lua safe to run, and none of that is bypassed here.
