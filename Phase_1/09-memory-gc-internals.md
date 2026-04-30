---
phase: 1
tags: [csharp, memory, gc, heap, stack, performance]
date: 2026-04-30
---

# Memory & GC Internals

#csharp #memory #gc #heap #stack #performance

## Table of Contents
- [[#Stack vs Heap]]
- [[#GC generations]]
- [[#Large Object Heap (LOH)]]
- [[#Collection triggers]]
- [[#GC modes]]
- [[#Stop-the-world pauses]]
- [[#Background GC]]
- [[#Finalizers and their cost]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## Stack vs Heap

| | Stack | Heap |
|---|---|---|
| What lives there | Value types (local vars), method frames | Reference types, boxed value types |
| Allocation | O(1) — move stack pointer | O(n) — GC must find free space |
| Deallocation | Automatic on method return | GC-managed |
| Size limit | ~1 MB (default) | Limited by RAM |
| Thread | Each thread has its own stack | Shared across threads |

---

## GC generations

The GC is **generational** — most objects die young (generational hypothesis).

```
Gen 0  →  Gen 1  →  Gen 2
(young)            (old / long-lived)
```

| Generation | Collected | Objects |
|---|---|---|
| Gen 0 | Most frequently | New, short-lived |
| Gen 1 | Medium | Survived one Gen 0 |
| Gen 2 | Rarely (expensive) | Long-lived, static-like |

Surviving a collection = **promoted** to next generation.

---

## Large Object Heap (LOH)

Objects ≥ **85 000 bytes** bypass Gen 0/1/2 entirely and go straight to the LOH.

- LOH is only collected during **Gen 2** collections
- LOH is **not compacted by default** (fragmentation risk)
- Arrays of `double` ≥ 1 000 elements go to LOH on 32-bit

> [!WARNING]
> Frequent large allocations (e.g., 100KB buffers per HTTP request) cause LOH fragmentation and Gen 2 pressure. Use `ArrayPool<T>` to avoid this.

---

## Collection triggers

Gen 0 is triggered when:
- Gen 0 budget is exhausted (~256KB by default)
- OS signals low memory
- `GC.Collect()` is called explicitly (avoid in production)

Gen 2 is triggered when:
- Gen 1 budget is exhausted
- LOH budget is exhausted
- System-wide memory pressure

---

## GC modes

| Mode | Use case | Threads | Latency |
|---|---|---|---|
| Workstation GC | Desktop apps, single user | 1 GC thread | Low pause |
| Server GC | ASP.NET Core, services | 1 GC thread per CPU core | Higher throughput |

ASP.NET Core uses **Server GC** by default — larger budgets, more memory, better throughput.

---

## Stop-the-world pauses

During collection, the GC must pause all managed threads (STW pause) to safely move objects. Long STW pauses cause request latency spikes in web apps.

> [!NOTE]
> Background GC (enabled by default) reduces STW pauses by doing most Gen 2 work concurrently, but Gen 0/1 collections are still stop-the-world.

---

## Background GC

Background GC runs Gen 2 concurrently with your application threads. Gen 0 and Gen 1 collections can happen even while background GC is running.

This significantly reduces long pause times in Gen 2 without sacrificing throughput.

---

## Finalizers and their cost

Objects with finalizers require **two GC cycles** to collect:
1. First cycle: moved to finalizer queue (not collected yet)
2. Finalizer thread runs
3. Second cycle: actually collected

This doubles the lifetime and GC cost. Always implement `IDisposable` + `GC.SuppressFinalize()` to avoid finalizer overhead when the object is properly disposed.

---

## Gotchas

- `GC.Collect()` in production is almost always wrong — it forces expensive Gen 2 collections
- Boxing a struct sends it to the heap — avoid boxing in hot paths
- Memory leaks in .NET are usually caused by **rooted references** (static fields, event subscriptions), not the GC failing
- `WeakReference<T>` is the right tool when you want the GC to be able to collect an object despite holding a reference

---

## Interview one-liner

> The GC is generational — short-lived objects are collected cheaply in Gen 0, while long-lived objects survive into Gen 2 where collections are expensive and rare. The LOH handles large allocations separately and isn't compacted.

---

## See also

- [[01-weak-reference]]
- [[02-span-memory]]
- [[03-arraypool]]
- [[10-class-struct-record]]
