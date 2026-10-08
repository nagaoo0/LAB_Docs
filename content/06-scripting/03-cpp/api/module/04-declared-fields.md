---
title: "Declared fields"
---

Values an author edits in the inspector, and the module reads as its own members.

A declared field is a plain data member of the behaviour, plus one table entry saying what it is
called and where it lives. The member is the module's; the engine only knows the offset.

## The types that can cross

```c
typedef enum LAB_FieldType
{
	LAB_FIELD_FLOAT = 0,
	LAB_FIELD_INT = 1,
	LAB_FIELD_BOOL = 2,
	LAB_FIELD_VEC3 = 3,
	LAB_FIELD_COUNT
} LAB_FieldType;
```

Only types the engine can write into a module's own member without knowing its layout. Each is a
fixed number of bytes the engine `memcpy`s to the declared offset.

A string or an asset path is deliberately not among them. Either would have to cross as
something whose representation two independently compiled sides agree on, which makes it a
setter the module owns rather than a write the engine performs, and there is no such thing yet.
The practical consequence today: a behaviour that loads assets by path has no way to declare
that path, so a packaged build cannot see the dependency and the asset has to be named in
`Build.AdditionalIncludes` instead.

## `LAB_FieldDesc`

```c
typedef struct LAB_FieldDesc
{
	uint32_t StructSize;
	const char* Name;
	int Type;
	uint32_t Offset;
} LAB_FieldDesc;
```

| Field | |
|---|---|
| `StructSize` | `sizeof` of this struct as the module built it, so a later append is a minor change |
| `Name` | What a stored value is keyed by. Must outlive the module |
| `Type` | A `LAB_FieldType` |
| `Offset` | Where to write it, in the module's own instance struct |

## Declaring them

```cpp
class Tuner : public LABNative::Behaviour
{
public:
	// Deliberately not zero. A field nothing authored has to be distinguishable from one
	// the engine wrote a default over, and zeroes make those two the same observation.
	float Speed = 1.5f;
	int Steps = 3;
	bool Enabled = true;
	LAB_Vec3 Offset{ 9.0f, 9.0f, 9.0f };
};

// Namespace scope: the descriptor the engine keeps points at this array for as long as
// the module is loaded.
const LAB_FieldDesc kTunerFields[] = {
	LABNative::Field("Speed", LAB_FIELD_FLOAT, LABNative::FieldOffset(&Tuner::Speed)),
	LABNative::Field("Steps", LAB_FIELD_INT, LABNative::FieldOffset(&Tuner::Steps)),
	LABNative::Field("Enabled", LAB_FIELD_BOOL, LABNative::FieldOffset(&Tuner::Enabled)),
	LABNative::Field("Offset", LAB_FIELD_VEC3, LABNative::FieldOffset(&Tuner::Offset)),
};
```

```cpp
LABNative::RegisterBehaviour<Tuner>(registry, "Tuner", kTunerFields, 4);
```

`LABNative::Field(name, type, offset)` fills a descriptor. `LABNative::FieldOffset(&T::member)`
computes the offset.

## Why `FieldOffset` and not `offsetof`

A behaviour inherits its callbacks from `LABNative::Behaviour`, which makes it polymorphic, and
the library's `offsetof` is only required to work on standard-layout types. `FieldOffset` takes
a pointer to member, which is the same arithmetic with the type checked, and every compiler
here supports that form.

```cpp
template<typename T, typename M>
inline uint32_t FieldOffset(M T::* member);
```

It is the reason a field has to be a plain data member. A reference, a bitfield or anything
requiring a custom accessor cannot be one.

## Two rules

**The field table must outlive the module.** The engine keeps the pointer for as long as the
library is loaded, so a namespace-scope array is correct and a local one is not.

**Only fixed-size members.** `float`, `int`, `bool`, `LAB_Vec3`. Nothing else crosses.

## When a value is applied

1. The engine constructs the instance (`new T()`), so your constructor's values are in place.
2. The entity id is assigned.
3. Stored field values are written to their offsets.
4. `OnCreate` is called.

So the first callback already sees authored values, and a field nothing authored keeps whatever
the constructor gave it. Two consequences worth designing around:

- **A default the engine never wrote is the constructor's default.** If you want to tell "the
  author set this" from "nobody set this", pick a constructor default that is not your
  operational default. The engine's own fixture does this on purpose: `Tuner`'s fields default
  to values no sane author would pick, so a test can distinguish an applied value from a stored
  one.
- **A value that will not parse is reported and skipped**, rather than written as a zero. A
  typo in a scene file leaves the constructor's default rather than silently producing a zero.

**A value for a field the module no longer declares is kept, not dropped.** Changing which
fields a behaviour declares does not erase what an author had set, which matters the first time
you rename a field in a shipping project.

## Reading the values back

| From | Call | Answers |
|---|---|---|
| Lua, in play | `entity:native_fields()` | What the module's members hold right now, keyed by behaviour then field |
| Editor Lua | `editor.native_field_defs(behaviour)` | What that behaviour declares |
| Editor Lua | `editor.set_native_field(entity, behaviour, field, value)` | Writes one |

`entity:native_fields()` reads the module's own memory, which is what tells an applied value
from a stored one: a field the scene has a value for and the module never declared does not
appear, and a declared field nobody authored appears with the constructor's default.

## In the inspector

The **Native Script** block draws one row per declared field under the class it came from. Only
a row that actually changed writes, so opening the panel is not an edit, and an untouched
default is not quietly turned into a zero.

Values are stored per entity, keyed by behaviour and field, so two instances of the same
behaviour on different entities keep their own. An entity running the same class twice keeps two
independent sets.

## Fields and hot reload

Declared fields are the only thing carried across a module reload. Everything else in the
instance is the module's private business and is recreated from its constructor.

At reload, values are written back **after** the scene's own authored values, so a value the
author set wins over a value the previous instance happened to hold. See
[Building and loading](03-building-and-loading.md#hot-reload).
