---
phase: 1
tags: [csharp, memory, performance, span, ref-struct]
date: 2026-04-30
---

# Span\<T\> and Memory\<T\>

#csharp #memory #performance #span #ref-struct

## Table of Contents
- [[#The problem they solve]]
- [[#Span-T]]
- [[#Memory-T]]
- [[#Span vs Memory comparison]]
- [[#Code example]]
- [[#ref struct restriction]]
- [[#Common use cases]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## The problem they solve

Before `Span<T>`, slicing an array meant creating a new array — a heap allocation and a copy. In hot paths (parsing, networking, serialisation) this causes GC pressure. `Span<T>` lets you work with a **slice of existing memory** with zero allocation.

---

## Span-T

`Span<T>` is a **ref struct** — a stack-only, allocation-free view over a contiguous region of memory. It can point to:

- Stack-allocated arrays (`stackalloc`)
- Heap arrays
- Native/unmanaged memory

```csharp
int[] array = { 1, 2, 3, 4, 5 };
Span<int> slice = array.AsSpan(1, 3); // [2, 3, 4] — no copy
slice[0] = 99;                        // modifies original array
```

---

## Memory-T

`Memory<T>` is the **heap-friendly cousin** of `Span<T>`. Not a ref struct, so it can be stored in fields, used in async methods, and passed across await points.

```csharp
Memory<byte> buffer = new byte[1024];
await ProcessAsync(buffer); // fine — Memory<T> survives await
```

---

## Span vs Memory comparison

| Feature | `Span<T>` | `Memory<T>` |
|---|---|---|
| Lives on | Stack only | Heap or stack |
| ref struct | ✅ Yes | ❌ No |
| Use in async | ❌ No | ✅ Yes |
| Use in fields | ❌ No | ✅ Yes |
| Allocation | Zero | One (Memory object) |
| Get a Span from it | — | `.Span` property |

---

## Code example

```csharp
// Parsing without allocations
ReadOnlySpan<char> input = "hello,world".AsSpan();
int comma = input.IndexOf(',');
ReadOnlySpan<char> first = input[..comma];   // "hello" — no new string
ReadOnlySpan<char> second = input[(comma+1)..]; // "world" — no new string
```

> [!NOTE]
> `string.AsSpan()` returns `ReadOnlySpan<char>` — strings are immutable so you can only read, not write.

---

## ref struct restriction

Because `Span<T>` is a ref struct:
- Cannot be boxed
- Cannot be used as a generic type argument
- Cannot be stored in class fields or array elements
- Cannot cross an `await` boundary

> [!WARNING]
> If you need to pass a buffer across an `await`, use `Memory<T>` instead of `Span<T>`.

---

## Common use cases

- Parsing text without `string.Substring()` allocations
- Reading/writing binary protocols (`MemoryMarshal`, `BinaryPrimitives`)
- Working with `stackalloc` buffers safely
- Writing high-performance serialisers

---

## Gotchas

- `Span<T>` pointing to a `stackalloc` buffer is only valid within that stack frame — never store or return it
- `ReadOnlySpan<T>` vs `Span<T>` — prefer readonly when you don't need to write
- `MemoryPool<T>` is to `Memory<T>` what `ArrayPool<T>` is to arrays

---

## Interview one-liner

> `Span<T>` is a zero-allocation stack-only slice of memory; `Memory<T>` is the heap-safe version you use across async boundaries.

---

## See also

- [[01-weak-reference]]
- [[03-arraypool]]
- [[09-memory-gc-internals]]
