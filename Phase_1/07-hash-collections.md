---
phase: 1
tags: [csharp, collections, hashset, dictionary, internals]
date: 2026-04-30
---

# Hash-Based Collection Internals

#csharp #collections #hashset #dictionary #internals

## Table of Contents
- [[#The core idea]]
- [[#How Dictionary works step by step]]
- [[#HashSet vs Dictionary]]
- [[#Load factor and resize]]
- [[#Collision handling]]
- [[#What breaks hash collections]]
- [[#Performance characteristics]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## The core idea

Hash-based collections achieve O(1) average lookup by turning a key into a **bucket index** via `GetHashCode()`, then using `Equals()` to find the exact item within that bucket.

```
Key → GetHashCode() → bucket index → scan bucket with Equals()
```

---

## How Dictionary works step by step

1. Call `key.GetHashCode()` → get an integer
2. Map integer to a bucket: `bucketIndex = hashCode % bucketCount`
3. Look in that bucket for an entry where `Equals(existingKey, key)` is true
4. If found → return value. If not → `KeyNotFoundException`

```csharp
var dict = new Dictionary<string, int>();
dict["hello"] = 42;
// Internally:
// 1. "hello".GetHashCode() → some int, e.g. 1234567
// 2. 1234567 % 16 → bucket 7
// 3. Store ("hello", 42) in bucket 7
```

---

## HashSet vs Dictionary

| | `HashSet<T>` | `Dictionary<K,V>` |
|---|---|---|
| Stores | Keys only | Key-value pairs |
| Lookup | O(1) average | O(1) average |
| Use case | Membership testing | Key-to-value mapping |
| Duplicate keys | ❌ Ignored | ❌ Exception on Add |

---

## Load factor and resize

Buckets have a **load factor** threshold (default ~0.72 for `Dictionary`). When `count / capacity > threshold`:

1. New backing array created (roughly 2× size)
2. All entries **re-hashed** into new buckets
3. Old array discarded → GC pressure spike

> [!NOTE]
> Pre-size your collection when you know the count: `new Dictionary<K,V>(capacity)` avoids mid-population resizes.

---

## Collision handling

When two keys land in the same bucket (collision), .NET uses **chaining** — bucket holds a linked list of entries. Lookup scans the list with `Equals`.

A **bad `GetHashCode`** (e.g., always returns 0) degrades every operation to O(n) — every key lands in bucket 0.

---

## What breaks hash collections

```csharp
// DANGER: mutable key
var key = new MutablePoint { X = 1, Y = 2 };
dict[key] = "hello";
key.X = 99; // hash changes — dict can no longer find this entry!
```

> [!WARNING]
> Never mutate an object while it's being used as a dictionary key. The object's hash changes but its bucket position in the dictionary doesn't.

---

## Performance characteristics

| Operation | Average | Worst case |
|---|---|---|
| `Add` | O(1) | O(n) resize |
| `TryGetValue` | O(1) | O(n) all collisions |
| `Remove` | O(1) | O(n) |
| `ContainsKey` | O(1) | O(n) |

---

## Gotchas

- `GetHashCode` must be consistent within a process run — .NET randomises string hashes across runs (security)
- `Dictionary<K,V>` is not thread-safe — use `ConcurrentDictionary<K,V>` for concurrent access
- Enumerating a `Dictionary` while modifying it throws `InvalidOperationException`
- `HashSet<T>.UnionWith`, `IntersectWith`, `ExceptWith` are all O(n) set operations

---

## Interview one-liner

> Hash collections use `GetHashCode()` to find the bucket and `Equals()` to confirm the match — both must be correctly implemented or lookups silently fail.

---

## See also

- [[04-equals-gethashcode]]
- [[05-iequatable]]
- [[08-hashcode-combine]]
