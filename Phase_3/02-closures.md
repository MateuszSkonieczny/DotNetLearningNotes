---
phase: 3
topic: Closures
status: complete
tags: [csharp, closures, lambdas, compiler, memory]
---

# Closures

## Table of Contents
- [[#Core Idea]]
- [[#What The Compiler Actually Does]]
- [[#The for vs foreach Capture Trap]]
- [[#Shared State On Purpose - The Accumulator Pattern]]
- [[#Self-Check]]
- [[#See Also]]

## Core Idea

A **closure** is a function (usually a lambda) that captures a variable from its enclosing scope and keeps it alive — even after the method that declared that variable has already returned.

```csharp
Func<int> MakeCounter()
{
    int count = 0;
    return () => ++count;
}

var counter = MakeCounter();
Console.WriteLine(counter()); // 1
Console.WriteLine(counter()); // 2
Console.WriteLine(counter()); // 3
```

`count` would normally die with the stack frame of `MakeCounter` once it returns. It doesn't, because the lambda needs it later.

> [!NOTE]
> This is the same mechanism underpinning a delegate's `_target` reference (see [[01-delegates-events]]) — a closure is just an object on the heap, and the delegate holds a reference to it like any other target.

## What The Compiler Actually Does

The move to the heap is not something that happens at runtime — it's decided **statically, at compile time**. The compiler analyzes the method body, sees that `count` is captured by a lambda that outlives the method, and generates a class (a "display class") to hold it:

```csharp
// conceptually, what the compiler generates for MakeCounter:
private sealed class DisplayClass
{
    public int count;
    public int Lambda() => ++count;
}

Func<int> MakeCounter()
{
    var d = new DisplayClass { count = 0 };
    return d.Lambda;
}
```

`count` never sits on the stack at all in the compiled output — the compiler decided at compile time, from static analysis of the source, that it belongs on the heap from the start. There's no "promotion" happening while the program runs.

> [!WARNING]
> Every local variable captured by a lambda in the same method that's *not* captured gets folded into the same display class instance. Capturing one variable can quietly extend the lifetime of others declared nearby, if the compiler groups them into one generated class.

## The for vs foreach Capture Trap

```csharp
var actions = new List<Action>();

for (int i = 0; i < 3; i++)
    actions.Add(() => Console.WriteLine(i));

foreach (var a in actions) a();
// 3
// 3
// 3
```

All three lambdas capture the **same** `i` — not three separate copies. `for` declares its loop variable **once**, outside the loop body; each iteration only mutates it. By the time the lambdas run, `i` already holds its final value, `3`.

```csharp
foreach (var x in someList) { /* body */ }

// desugars to (C# 5.0+):
using var e = someList.GetEnumerator();
while (e.MoveNext())
{
    var x = e.Current;   // fresh declaration, inside the loop body
    /* body */
}
```

Because `x` is declared **inside** the loop body, each iteration gets its own display-class instance. A lambda capturing `x` in a `foreach` gets a distinct value per iteration — `0, 1, 2`, not `2, 2, 2`.

| | `for` | `foreach` (C# 5.0+) |
|---|---|---|
| Where the variable is declared | Before the loop body | Inside the loop body, fresh each time |
| Closures created per iteration | One, shared | One per iteration |
| Captured-in-lambda result | last value repeated | each iteration's own value |

> [!NOTE]
> Before C# 5.0 (2012), `foreach` behaved exactly like `for` — one shared variable, same trap. Fixing it was a deliberate, rare breaking change: the C# team judged the old behavior was a footgun nobody intentionally relied on. `for` never got the same fix, because its control variable is explicitly declared by the programmer outside the body — changing that scope would change the loop's own semantics, not just closures over it.

**The manual fix for `for`** (the idiom used before `foreach` was fixed, and still needed for `for` today):

```csharp
for (int i = 0; i < 3; i++)
{
    int local = i;          // fresh variable, fresh closure, every iteration
    actions.Add(() => Console.WriteLine(local));
}
// 0
// 1
// 2
```

## Shared State On Purpose - The Accumulator Pattern

The `for`/`foreach` trap is closures behaving exactly as designed, applied somewhere unintended. The same "one shared variable, many lambdas" mechanism is also the *entire point* of a pattern like this:

```csharp
public (Action<int> Add, Func<int> Total) MakeAccumulator()
{
    var total = 0;
    Action<int> add = amount => { total += amount; };
    Func<int> getTotal = () => total;
    return (add, getTotal);
}

var (add, getTotal) = MakeAccumulator();
add(5);
add(10);
Console.WriteLine(getTotal()); // 15
```

`add` and `getTotal` **must** see the same `total` for the pattern to make sense — that's the intent, not a bug. The mechanism (shared capture) is identical to the loop trap; only the intent differs.

### Why the lambda types differ here

| Lambda body | What the compiler sees | Inferred delegate |
|---|---|---|
| `x => x + 1` | an expression, produces a value | `Func<T, TResult>` |
| `x => { total += x; }` | a block with a statement, no `return` | `Action<T>` |
| `x => { return x + 1; }` | a block with an explicit `return` | `Func<T, TResult>` |

Braces alone don't decide the delegate type — whether there's a value to return does. `total += x;` as a statement inside `{ }` produces nothing to return, so it fits `Action<T>`.

## Self-Check

- [x] Why a captured local outlives the method that declared it
- [x] That the heap allocation is a compile-time decision, not a runtime "promotion"
- [x] Why `for` captures one shared variable but `foreach` (C# 5.0+) captures a fresh one per iteration
- [x] How to manually replicate `foreach`'s per-iteration capture inside a `for` loop
- [x] How to tell, from a lambda's body alone, whether the compiler infers `Action<T>` or `Func<T,TResult>`
- [x] When shared capture is the intended pattern rather than a bug (accumulator-style closures)

## See Also

- [[Phase_3]]
- [[01-delegates-events]] (a closure is the heap object a delegate's `_target` points to)
- [[Generics]] (next in Phase 3)
