---
phase: 1
tags: [csharp, equality, iequatable, generics, performance]
date: 2026-04-30
---

# IEquatable\<T\>

#csharp #equality #iequatable #generics #performance

## Table of Contents
- [[#What it is]]
- [[#Why not just override Equals(object)]]
- [[#Implementation]]
- [[#When the compiler uses it]]
- [[#Full pattern]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## What it is

`IEquatable<T>` is an interface with a single method:

```csharp
public interface IEquatable<T>
{
    bool Equals(T? other);
}
```

It provides a **strongly-typed, non-boxing** equality check alongside the standard `object.Equals` override.

---

## Why not just override Equals(object)

`Equals(object? obj)` accepts `object` — for value types (`struct`), this means **boxing**: the value type is wrapped in a heap object just to compare it.

```csharp
// Without IEquatable<T>:
struct Point { public int X, Y; }
object a = new Point(1, 2); // boxed
object b = new Point(1, 2); // boxed again
a.Equals(b); // reflection-based, slow
```

`IEquatable<T>` bypasses boxing entirely:

```csharp
bool Equals(Point? other) // no boxing, type-safe
```

---

## Implementation

```csharp
public struct Point : IEquatable<Point>
{
    public int X { get; }
    public int Y { get; }

    public bool Equals(Point other)        // strongly typed — no boxing
        => X == other.X && Y == other.Y;

    public override bool Equals(object? obj) // still needed for object API
        => obj is Point p && Equals(p);

    public override int GetHashCode()
        => HashCode.Combine(X, Y);
}
```

---

## When the compiler uses it

Generic collections and LINQ prefer `IEquatable<T>` when available:

| API | Uses `IEquatable<T>`? |
|---|---|
| `List<T>.Contains()` | ✅ |
| `HashSet<T>` | ✅ via `EqualityComparer<T>.Default` |
| `Dictionary<K,V>` | ✅ |
| `Enumerable.Contains()` | ✅ |
| `Array.IndexOf()` | ❌ uses `object.Equals` |

`EqualityComparer<T>.Default` checks for `IEquatable<T>` at runtime and uses the typed path if available.

---

## Full pattern

For a class (where boxing isn't the main concern, but consistency is):

```csharp
public class Person : IEquatable<Person>
{
    public string Name { get; }

    public bool Equals(Person? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return Name == other.Name;
    }

    public override bool Equals(object? obj) => Equals(obj as Person);
    public override int GetHashCode() => HashCode.Combine(Name);

    public static bool operator ==(Person? a, Person? b) => Equals(a, b);
    public static bool operator !=(Person? a, Person? b) => !Equals(a, b);
}
```

> [!NOTE]
> Records implement `IEquatable<T>` automatically — you get this for free without writing any code.

---

## Gotchas

- Implementing `IEquatable<T>` without overriding `object.Equals` is inconsistent — always do both
- For structs: also override `GetHashCode` — the default struct implementation is reflection-based and slow
- Don't forget `operator ==` and `!=` for classes if you want `==` to use value equality

---

## Interview one-liner

> `IEquatable<T>` provides a typed, non-boxing `Equals` method — critical for structs used in collections, where the default `object.Equals` would cause boxing on every comparison.

---

## See also

- [[04-equals-gethashcode]]
- [[06-record-equality]]
- [[07-hash-collections]]
- [[08-hashcode-combine]]
