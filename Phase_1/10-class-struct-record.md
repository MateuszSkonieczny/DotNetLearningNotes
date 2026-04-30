---
phase: 1
tags: [csharp, class, struct, record, value-types, reference-types]
date: 2026-04-30
---

# class vs struct vs record vs record struct

#csharp #class #struct #record #value-types #reference-types

## Table of Contents
- [[#The big picture]]
- [[#Full comparison table]]
- [[#Memory and allocation]]
- [[#Equality defaults]]
- [[#Mutability]]
- [[#Inheritance]]
- [[#When to use which]]
- [[#with expressions]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## The big picture

C# has two fundamental kinds of types: **reference types** (live on heap, passed by reference) and **value types** (passed by copy). `record` adds value-based equality semantics on top of either kind.

---

## Full comparison table

| Feature | `class` | `struct` | `record` | `record struct` |
|---|---|---|---|---|
| Kind | Reference | Value | Reference | Value |
| Lives on | Heap | Stack / inline | Heap | Stack / inline |
| Default equality | Reference | Value (reflection) | Value (generated) | Value (generated) |
| `==` | Reference | Value (if overridden) | Value ✅ | Value ✅ |
| Mutable by default | ✅ | ✅ | ✅ (init props) | ✅ |
| Immutable variant | Manual | `readonly struct` | `record` (init only) | `readonly record struct` |
| Inheritance | ✅ | ❌ | ✅ (from record) | ❌ |
| `with` expression | ❌ | ❌ | ✅ | ✅ |
| Null possible | ✅ | ❌ (Nullable<T>) | ✅ | ❌ (Nullable<T>) |
| Boxing | No | Yes (if cast to object) | No | Yes |

---

## Memory and allocation

```csharp
class PointClass { public int X, Y; }
struct PointStruct { public int X, Y; }

// class: always heap-allocated, variable holds a reference (pointer)
var pc = new PointClass(); // heap allocation

// struct: lives where declared — stack for locals, inline in arrays/classes
var ps = new PointStruct(); // no heap allocation for the struct itself
```

> [!NOTE]
> Structs inside arrays or class fields live on the heap *as part of that container* — they're still not separate heap allocations.

---

## Equality defaults

```csharp
var c1 = new MyClass { X = 1 };
var c2 = new MyClass { X = 1 };
c1 == c2; // FALSE — different objects

record MyRecord(int X);
var r1 = new MyRecord(1);
var r2 = new MyRecord(1);
r1 == r2; // TRUE — value equality
```

---

## Mutability

| Type | Default | Immutable variant |
|---|---|---|
| `class` | Mutable | Manual (`private set`, `init`) |
| `struct` | Mutable | `readonly struct` |
| `record` | Init-only props by default | Already effectively immutable after construction |
| `record struct` | Mutable | `readonly record struct` |

> [!WARNING]
> A plain `record struct` is mutable by default — its properties can be changed after construction. Use `readonly record struct` for true immutability.

---

## Inheritance

- `class` → full inheritance hierarchy
- `struct` → no inheritance (can implement interfaces)
- `record` → can inherit from other records; sealed if you want no further inheritance
- `record struct` → no inheritance

---

## When to use which

| Scenario | Best choice |
|---|---|
| Domain entity (EF Core, has identity) | `class` |
| DTOs, API responses (value semantics) | `record` |
| Small value with no identity (coordinates, money) | `readonly record struct` or `readonly struct` |
| High-performance buffer / no heap alloc needed | `struct` |
| Immutable value type in hot paths | `readonly record struct` |

---

## with expressions

`with` creates a copy with specified properties changed. Only available on `record` and `record struct`:

```csharp
var original = new Point(1, 2);
var moved = original with { X = 10 }; // new Point(10, 2)
```

---

## Gotchas

- Structs are **copied on assignment** — modifying a copy doesn't affect the original
- Large structs in hot paths may be slower than classes due to copying cost
- `record struct` fields are mutable by default — easy to accidentally mutate
- `record` uses `EqualityContract` to prevent cross-type equality between base and derived records

---

## Interview one-liner

> `class` is a heap-allocated reference type with reference equality; `struct` is a value type copied on assignment; `record` adds compiler-generated value equality to a class; `record struct` does the same for a value type.

---

## See also

- [[09-memory-gc-internals]]
- [[06-record-equality]]
- [[11-compile-time-vs-runtime]]
- [[12-struct-parameter-passing]]
