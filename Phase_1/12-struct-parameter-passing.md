---
phase: 1
tags: [csharp, struct, parameters, ref, out, in, performance]
date: 2026-04-30
---

# Struct Parameter Passing

#csharp #struct #parameters #ref #out #in #performance

## Table of Contents
- [[#Default — pass by copy]]
- [[#ref — pass by reference]]
- [[#out — initialise and return]]
- [[#in — read-only reference]]
- [[#ref readonly — return by reference, no copy]]
- [[#Defensive copy trap]]
- [[#Comparison table]]
- [[#When to use each]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## Default — pass by copy

By default, structs are **copied** when passed to a method. The method works on its own copy — the original is unchanged.

```csharp
void Move(Point p) { p.X += 10; } // mutates the COPY

var point = new Point(1, 2);
Move(point);
Console.WriteLine(point.X); // still 1
```

---

## ref — pass by reference

`ref` passes the actual memory location. Modifications affect the original. Must be initialised before passing.

```csharp
void Move(ref Point p) { p.X += 10; } // mutates the ORIGINAL

var point = new Point(1, 2);
Move(ref point);
Console.WriteLine(point.X); // 11
```

---

## out — initialise and return

`out` is like `ref` but the variable doesn't need to be initialised before the call. The method **must** assign it.

```csharp
bool TryParse(string s, out int result)
{
    result = 0; // must assign
    return int.TryParse(s, out result);
}

TryParse("42", out int value);
```

---

## in — read-only reference

`in` passes a reference but **prevents mutation**. The method gets a pointer to the original without being able to modify it. Useful for large structs — avoids copying without allowing changes.

```csharp
void Print(in LargeStruct s)
{
    Console.WriteLine(s.Value); // OK — reading
    // s.Value = 99;            // compile error — readonly
}
```

> [!NOTE]
> For small structs (≤ 16 bytes), `in` may actually be slower than a copy because of pointer indirection overhead. Profile before optimising.

---

## ref readonly — return by reference, no copy

Lets you return a reference to a struct from a method without copying, while preventing the caller from mutating it:

```csharp
private Point _cached;

public ref readonly Point GetCached() => ref _cached; // no copy, readonly

// Caller:
ref readonly Point p = ref obj.GetCached(); // no copy
```

---

## Defensive copy trap

When you call a method on an `in` parameter (or a `readonly` field of struct type), the compiler may insert a **defensive copy** to ensure the method can't mutate through the reference:

```csharp
readonly struct Timer { public void Reset() { /* mutates */ } }

void Process(in Timer t)
{
    t.Reset(); // compiler makes a COPY of t and calls Reset() on the copy!
               // your original t is unchanged — and you didn't ask for a copy
}
```

> [!WARNING]
> This silent defensive copy is a common performance bug. Mark your structs `readonly` at the struct level if they don't mutate — the compiler then knows it's safe to skip the defensive copy.

---

## Comparison table

| Modifier | Direction | Caller must init? | Method can mutate? | Copy made? |
|---|---|---|---|---|
| (none) | In | ✅ | Mutates copy only | ✅ Always |
| `ref` | In/Out | ✅ | ✅ Original | ❌ |
| `out` | Out | ❌ | Must assign | ❌ |
| `in` | In | ✅ | ❌ Readonly | ❌ (unless defensive copy) |

---

## When to use each

- **`ref`** — when the method needs to mutate the caller's value
- **`out`** — multiple return values or Try-pattern methods
- **`in`** — large structs (> ~16 bytes) passed read-only for performance
- **`ref readonly` return** — expose large struct fields without copying

---

## Gotchas

- `ref` and `out` cannot be used with async methods — use return values instead
- `in` with non-readonly structs triggers defensive copies on method calls — can eliminate the performance gain
- Forgetting `ref` at the call site is a compile error — it's explicit and required
- `ref struct` (like `Span<T>`) can be passed by `ref`, but the struct type itself can't escape the stack

---

## Interview one-liner

> Structs are copied by default on method calls; `ref` passes the original, `out` requires the method to assign a value, and `in` passes a read-only reference — avoiding copies for large structs without allowing mutation, though watch out for defensive copy traps.

---

## See also

- [[10-class-struct-record]]
- [[09-memory-gc-internals]]
- [[02-span-memory]]
- [[11-compile-time-vs-runtime]]
