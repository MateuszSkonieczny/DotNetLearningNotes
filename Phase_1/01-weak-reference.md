---
phase: 1
tags: [csharp, memory, gc, weak-reference]
date: 2026-04-30
---

# WeakReference\<T\>

#csharp #memory #gc #weak-reference

## Table of Contents
- [[#What is it]]
- [[#How it works]]
- [[#WeakReference vs WeakReference-T]]
- [[#Code example]]
- [[#When to use]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## What is it

A `WeakReference<T>` holds a reference to an object **without preventing the GC from collecting it**. The GC is free to reclaim the object as if the weak reference didn't exist.

---

## How it works

Normal (strong) references keep objects alive. A weak reference is invisible to the GC's reachability analysis — once only weak references remain, the object is eligible for collection. After collection, `TryGetTarget()` returns `false`.

---

## WeakReference vs WeakReference-T

| | `WeakReference` (old) | `WeakReference<T>` (preferred) |
|---|---|---|
| Generic | ❌ | ✅ |
| Type-safe | ❌ | ✅ |
| API | `.Target` (object) | `.TryGetTarget(out T)` |
| .NET version | 1.0+ | 4.5+ |

Always prefer `WeakReference<T>`.

---

## Code example

```csharp
var obj = new HeavyObject();
var weak = new WeakReference<HeavyObject>(obj);

obj = null; // remove strong reference
GC.Collect();

if (weak.TryGetTarget(out var recovered))
{
    // object survived — use it
    recovered.DoWork();
}
else
{
    // object was collected
}
```

> [!WARNING]
> Never use the result of `TryGetTarget` without holding a local variable. The object can be collected between the check and the use.

---

## When to use

- **Caches** — hold cached objects without preventing their eviction under memory pressure
- **Event listeners** — avoid memory leaks when subscribers outlive publishers
- **Circular reference helpers** — break cycles that confuse resource tracking

> [!NOTE]
> Don't use `WeakReference<T>` as a general substitute for proper lifetime management. It makes object lifetime unpredictable.

---

## Gotchas

- Object may be collected at *any* GC pause — even between two lines
- `TryGetTarget` returning `true` doesn't mean the object will stay alive; store the result in a local variable immediately
- Short-lived objects (Gen 0) are collected very frequently — don't rely on weak refs for them

---

## Interview one-liner

> `WeakReference<T>` lets you reference an object without keeping it alive — the GC can collect it anytime, and you check via `TryGetTarget` before use.

---

## See also

- [[02-span-memory]]
- [[03-arraypool]]
- [[09-memory-gc-internals]]
