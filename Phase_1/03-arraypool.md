---
phase: 1
tags: [csharp, memory, performance, gc, arraypool]
date: 2026-04-30
---

# ArrayPool\<T\>

#csharp #memory #performance #gc #arraypool

## Table of Contents
- [[#Why it exists]]
- [[#How it works]]
- [[#The API]]
- [[#Correct usage pattern]]
- [[#ArrayPool + Span — the power combo]]
- [[#Gotchas]]
- [[#clearArray — false vs true]]
- [[#When NOT to use ArrayPool.Shared]]
- [[#Interview one-liner]]
- [[#See also]]

---

## Why it exists

Every `new byte[4096]` allocates on the heap. In hot paths (HTTP, serialisation, networking) this causes constant Gen0 pressure and — for arrays ≥ 85 000 bytes — LOH allocations collected only at Gen2.

`ArrayPool<T>` maintains a pool of reusable arrays. You **rent** one, use it, **return** it. Zero allocation on warm paths.

---

## How it works

Internally a 2D structure: buckets by power-of-2 size, each bucket holds several arrays. `Rent(n)` gives you the smallest bucket ≥ n. `Return` puts it back for the next caller.

```
Bucket 16   → [ array, array ]
Bucket 32   → [ array ]
Bucket 64   → [ array, array, array ]
...
```

---

## The API

```csharp
// Shared pool — thread-safe, good for most cases
ArrayPool<byte> pool = ArrayPool<byte>.Shared;

byte[] buffer = pool.Rent(minimumLength: 4096);
// buffer.Length may be LARGER than 4096 — always track actual length separately
pool.Return(buffer, clearArray: false);
```

---

## Correct usage pattern

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
try
{
    int bytesRead = stream.Read(buffer, 0, 4096);
    Process(buffer.AsSpan(0, bytesRead)); // use actual length, not buffer.Length
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

> [!WARNING]
> Always return in a `finally` block. A leaked buffer is permanently removed from the pool.

---

## ArrayPool + Span — the power combo

```csharp
byte[] rented = ArrayPool<byte>.Shared.Rent(size);
Span<byte> span = rented.AsSpan(0, size); // work with exact slice
// ... use span ...
ArrayPool<byte>.Shared.Return(rented);
```

Span gives you the exact slice; the rented array (which may be bigger) goes back to the pool.

---

## Gotchas

- `Rent(n)` returns an array **≥ n**, never exactly n — always use a separate length variable
- Rented arrays contain **stale data** from previous use — clear if you care about security/correctness
- Never use the array after returning it — another thread may have rented it
- Don't return arrays you didn't rent from the same pool

---

## clearArray — false vs true

| `clearArray` | When to use |
|---|---|
| `false` (default) | Performance-critical paths where you overwrite all data before reading |
| `true` | Security-sensitive data (passwords, keys) or when stale bytes could cause bugs |

---

## When NOT to use ArrayPool.Shared

- Very large allocations (> a few MB) — pool overhead may not pay off
- Long-lived buffers — pool arrays are meant for short-term use
- When `stackalloc` works — for small, stack-safe sizes this is even cheaper

> [!NOTE]
> For custom pooling behaviour (size limits, diagnostics), create `ArrayPool<T>.Create()` instead of using `Shared`.

---

## Interview one-liner

> `ArrayPool<T>` lets you rent and return arrays to avoid repeated heap allocations — critical in high-throughput paths where `new byte[n]` would hammer the GC.

---

## See also

- [[02-span-memory]]
- [[09-memory-gc-internals]]
- [[01-weak-reference]]
