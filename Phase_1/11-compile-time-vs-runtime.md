---
phase: 1
tags: [csharp, dotnet, roslyn, clr, jit, const, static-readonly]
date: 2026-04-30
---

# Compile-Time vs Runtime

#csharp #dotnet #roslyn #clr #jit #const #static-readonly

## Table of Contents
- [[#The pipeline]]
- [[#Roslyn — compile time]]
- [[#CLR — runtime]]
- [[#JIT — just in time]]
- [[#const vs static readonly]]
- [[#Versioning risk with const]]
- [[#Practical implications]]
- [[#Gotchas]]
- [[#Interview one-liner]]
- [[#See also]]

---

## The pipeline

```
Your C# code
    ↓  Roslyn (compiler)
IL bytecode (.dll)
    ↓  CLR loads it
JIT compiles IL → native machine code
    ↓
CPU executes
```

---

## Roslyn — compile time

Roslyn is the C# compiler. At compile time:
- Syntax and type checking
- `const` values are **inlined** into the IL
- Generic type parameters are erased/specialised
- Source generators run
- Output: **IL (Intermediate Language)** in a `.dll`

---

## CLR — runtime

The Common Language Runtime:
- Loads assemblies
- Manages memory (GC)
- Provides type safety, exception handling
- JIT-compiles IL to native code on demand
- Runs the application

---

## JIT — just in time

The JIT compiler converts IL → native code **on first call** of each method:

- First call: JIT compiles → slow (one time)
- Subsequent calls: native code runs directly → fast
- Methods can be **tiered**: Tier 0 (quick compile) → Tier 1 (optimised) as usage increases
- AOT (Ahead-of-Time) pre-compiles everything — used in NativeAOT for startup performance

---

## const vs static readonly

| | `const` | `static readonly` |
|---|---|---|
| Value determined | Compile time | Runtime (once, on first access) |
| Inlined into IL | ✅ Yes | ❌ No — reference to field |
| Can be a string/primitive | ✅ | ✅ |
| Can be a complex object | ❌ | ✅ |
| Cross-assembly versioning | ⚠️ Dangerous | ✅ Safe |

```csharp
// Library A
public const int Version = 1;        // inlined at compile time
public static readonly int SafeVer = 1; // reference resolved at runtime
```

---

## Versioning risk with const

```csharp
// LibraryA v1:
public const int MaxRetries = 3;

// Your app compiled against v1:
// IL contains literal "3" baked in — NOT a reference to MaxRetries

// LibraryA updates to v2:
public const int MaxRetries = 5;

// Your app still uses 3 — until you RECOMPILE your app!
```

> [!WARNING]
> `const` values from external assemblies are baked into **your** IL at build time. If the library changes the value, you must recompile — no `dotnet restore` will help. Use `static readonly` for constants intended to be versioned.

---

## Practical implications

- Use `const` for truly universal constants that will **never** change (`Math.PI`, enum-like sentinels, internal magic numbers)
- Use `static readonly` for values that may change between library versions or that require runtime initialisation
- Roslyn source generators run at compile time — they can generate code based on attributes without reflection overhead at runtime

---

## Gotchas

- `const` can only hold: primitive types, `string`, `enum` values, `null`
- JIT inlines small methods aggressively — `[MethodImpl(MethodImplOptions.NoInlining)]` to prevent this (useful in benchmarking)
- `typeof(T)` is resolved at compile time for non-generic code but at JIT time for generics
- AOT compilation (NativeAOT) loses some reflection capabilities — `[RequiresDynamicCode]` warns you

---

## Interview one-liner

> Roslyn compiles C# to IL at build time; the CLR's JIT converts IL to native code at runtime on first method call. `const` is baked into IL at compile time (versioning risk), while `static readonly` is evaluated once at runtime (safe to update across assemblies).

---

## See also

- [[10-class-struct-record]]
- [[12-struct-parameter-passing]]
