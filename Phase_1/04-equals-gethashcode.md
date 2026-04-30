---
phase: 1
tags: [csharp, equality, gethashcode, contracts]
date: 2026-04-30
---

# Equals / GetHashCode Contract

#csharp #equality #gethashcode #contracts

## Table of Contents
- [[#The contract]]
- [[#Default behaviour]]
- [[#Overriding Equals]]
- [[#Overriding GetHashCode]]
- [[#The golden rule]]
- [[#Full example]]
- [[#operator == and !=]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## The contract

Equals and GetHashCode are a **pair** — you must override both together. The contract:

1. If `a.Equals(b)` → `a.GetHashCode() == b.GetHashCode()` (**mandatory**)
2. If `a.GetHashCode() == b.GetHashCode()` → `a.Equals(b)` is **not** guaranteed (hash collision is allowed)
3. `Equals` must be: reflexive, symmetric, transitive, consistent, and handle `null`

---

## Default behaviour

| Type | Default Equals | Default GetHashCode |
|---|---|---|
| `class` | Reference equality (same object in memory) | Based on object identity |
| `struct` | Value equality (field-by-field via reflection) | Based on fields (slow, reflection) |
| `record` | Value equality (compiler-generated) | Based on all properties |

---

## Overriding Equals

```csharp
public override bool Equals(object? obj)
{
    if (obj is not MyClass other) return false;
    if (ReferenceEquals(this, other)) return true;
    return Name == other.Name && Age == other.Age;
}
```

> [!NOTE]
> Also implement `IEquatable<T>` to avoid boxing when comparing value types or for performance with generics.

---

## Overriding GetHashCode

```csharp
public override int GetHashCode()
    => HashCode.Combine(Name, Age);
```

Rules for a good hash code:
- Fast to compute
- Same object → always same hash (within one process run)
- Well-distributed — minimise collisions
- **Never change** if the object is used as a dictionary key

---

## The golden rule

> If two objects are **equal**, they **must** have the same hash code.
> The reverse is NOT required — collisions are allowed.

Violating this breaks `Dictionary<K,V>`, `HashSet<T>`, and any hash-based collection.

---

## Full example

```csharp
public class Person : IEquatable<Person>
{
    public string Name { get; }
    public int Age { get; }

    public Person(string name, int age) => (Name, Age) = (name, age);

    public bool Equals(Person? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return Name == other.Name && Age == other.Age;
    }

    public override bool Equals(object? obj) => Equals(obj as Person);

    public override int GetHashCode() => HashCode.Combine(Name, Age);

    public static bool operator ==(Person? a, Person? b) => Equals(a, b);
    public static bool operator !=(Person? a, Person? b) => !Equals(a, b);
}
```

---

## operator == and !=

Overriding `Equals` does **not** automatically affect `==`. Override the operators explicitly if you want `==` to use value equality for a class.

> [!WARNING]
> For structs and records, `==` uses value equality by default. For classes, `==` remains reference equality unless you override the operator.

---

## Gotchas

- Using a mutable field in `GetHashCode` and then mutating the object while it's in a `Dictionary` will lose it — the key's hash bucket changes but the dictionary doesn't know
- `struct` default `GetHashCode` may include padding bytes or behave inconsistently — always override for structs used as dictionary keys
- `null.GetHashCode()` throws — `HashCode.Combine` handles nulls safely

---

## Interview one-liner

> If you override `Equals`, you must override `GetHashCode` to maintain the contract: equal objects must produce identical hash codes, or hash-based collections will silently break.

---

## See also

- [[05-iequatable]]
- [[06-record-equality]]
- [[07-hash-collections]]
- [[08-hashcode-combine]]
