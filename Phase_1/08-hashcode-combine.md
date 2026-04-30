---
phase: 1
tags: [csharp, hashcode, equality, performance]
date: 2026-04-30
---

# HashCode.Combine()

#csharp #hashcode #equality #performance

## Table of Contents
- [[#What it is]]
- [[#Basic usage]]
- [[#HashCode struct — incremental building]]
- [[#Why not XOR fields manually]]
- [[#Handling collections]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## What it is

`HashCode.Combine()` is a static helper (introduced in .NET Core 2.1) that produces a well-distributed hash from multiple values. It replaces manual hash combining patterns.

---

## Basic usage

```csharp
public override int GetHashCode()
    => HashCode.Combine(FirstName, LastName, DateOfBirth);
```

Supports 1–8 arguments. Handles `null` safely — `null` contributes 0 to the combination.

---

## HashCode struct — incremental building

For more than 8 fields, or when building the hash conditionally:

```csharp
public override int GetHashCode()
{
    var hash = new HashCode();
    hash.Add(FirstName);
    hash.Add(LastName);
    hash.Add(DateOfBirth);
    foreach (var tag in Tags)
        hash.Add(tag);
    return hash.ToHashCode();
}
```

> [!NOTE]
> Call `ToHashCode()` exactly once — subsequent calls throw `InvalidOperationException`.

---

## Why not XOR fields manually

XOR (`^`) is a common mistake:

```csharp
// BAD — XOR is commutative: (1,2) and (2,1) produce the same hash
return X.GetHashCode() ^ Y.GetHashCode();
```

`HashCode.Combine` uses a non-commutative mixing algorithm (based on xxHash32) — order matters, so `(1,2) ≠ (2,1)`.

| Approach | Collision-resistant | Null-safe | Order-sensitive |
|---|---|---|---|
| Manual XOR | ❌ | ❌ | ❌ |
| Old multiply pattern (`17 * 31 + ...`) | ⚠️ | Manual | ✅ |
| `HashCode.Combine` | ✅ | ✅ | ✅ |

---

## Handling collections

`HashCode.Combine` doesn't handle collections directly. Use the incremental API:

```csharp
var hash = new HashCode();
hash.Add(Id);
foreach (var item in Items)
    hash.Add(item);
return hash.ToHashCode();
```

Or use `SequenceEqualityComparer` if you want collection-as-value semantics.

---

## Gotchas

- Hash codes are **not stable across process restarts** — .NET randomises them (security). Never persist or serialise hash codes.
- `HashCode.Combine` is available from .NET Core 2.1+ and .NET Standard 2.1. For older targets, use the manual `(hash * 31) + field.GetHashCode()` pattern.
- Don't include mutable fields in `GetHashCode` if the object is used as a dictionary key.

---

## Interview one-liner

> `HashCode.Combine()` is the modern, null-safe, order-sensitive way to produce a well-distributed hash from multiple fields — replacing error-prone manual XOR patterns.

---

## See also

- [[04-equals-gethashcode]]
- [[05-iequatable]]
- [[06-record-equality]]
- [[07-hash-collections]]
