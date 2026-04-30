---
phase: 1
tags: [csharp, records, equality, value-semantics]
date: 2026-04-30
---

# Record Equality

#csharp #records #equality #value-semantics

## Table of Contents
- [[#What records give you for free]]
- [[#How it works under the hood]]
- [[#record vs record struct equality]]
- [[#Customising equality]]
- [[#with expressions and equality]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## What records give you for free

When you declare a `record`, the compiler auto-generates:

- `Equals(T? other)` — compares all properties by value
- `Equals(object? obj)` — delegates to the typed version
- `GetHashCode()` — combines hash codes of all properties
- `operator ==` and `operator !=` — use value equality
- `IEquatable<T>` implementation

```csharp
record Point(int X, int Y);

var a = new Point(1, 2);
var b = new Point(1, 2);
Console.WriteLine(a == b);        // True — value equality
Console.WriteLine(a.Equals(b));   // True
Console.WriteLine(ReferenceEquals(a, b)); // False — different objects
```

---

## How it works under the hood

The compiler generates something like:

```csharp
public virtual bool Equals(Point? other)
    => other is not null
    && EqualityContract == other.EqualityContract
    && EqualityComparer<int>.Default.Equals(X, other.X)
    && EqualityComparer<int>.Default.Equals(Y, other.Y);
```

`EqualityContract` is a `Type` property used for inheritance safety — two records of different types are never equal even if their properties match.

---

## record vs record struct equality

| | `record` (class) | `record struct` |
|---|---|---|
| Equality type | Value (by properties) | Value (by properties) |
| Reference identity | Different objects possible | N/A — value type |
| `==` overloaded | ✅ | ✅ |
| Nullable | Can be null | Cannot be null (unless `Nullable<T>`) |
| Boxing on compare | No (IEquatable) | No (IEquatable) |

---

## Customising equality

You can override specific properties from equality by redefining the generated members:

```csharp
record Person(string Name, int Age)
{
    // Exclude Age from equality — only Name matters
    public virtual bool Equals(Person? other)
        => other is not null && Name == other.Name;

    public override int GetHashCode() => HashCode.Combine(Name);
}
```

> [!WARNING]
> If you customise `Equals`, always customise `GetHashCode` to match — same contract as with classes.

---

## with expressions and equality

`with` creates a **copy** with changed properties. The copy is a new object but `==` returns true if all properties match:

```csharp
var original = new Point(1, 2);
var copy = original with { X = 1 }; // same property values

Console.WriteLine(original == copy);          // True
Console.WriteLine(ReferenceEquals(original, copy)); // False
```

---

## Gotchas

- Records with mutable properties (`init` can be set after construction via `with`) — equality is a snapshot at comparison time
- Inheritance: derived record type is NOT equal to base record type, even with identical properties (EqualityContract check)
- `record struct` does not support inheritance — no EqualityContract needed

---

## Interview one-liner

> Records auto-generate value-based `Equals`, `GetHashCode`, and `==` across all declared properties — you get correct equality for free without writing any boilerplate.

---

## See also

- [[04-equals-gethashcode]]
- [[05-iequatable]]
- [[10-class-struct-record]]
- [[08-hashcode-combine]]
