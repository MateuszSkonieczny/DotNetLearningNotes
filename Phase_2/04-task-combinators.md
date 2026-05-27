---
phase: 2
tags: [threading, async, task-combinators, whenall, whenany, wheneach]
date: 2026-05-27
---

# Task Combinators #threading #async #task-combinators

## Table of Contents
- [[#Overview]]
- [[#Task.WhenAll]]
- [[#Task.WhenAny]]
- [[#Task.WhenEach]]
- [[#Blocking Equivalents]]
- [[#Key Mechanics]]
- [[#Decision Guide]]
- [[#Code Examples]]
- [[#See also]]

---

## Overview

Task combinators are methods that **observe** already-running tasks — they do not start them. Tasks start the moment you call an async method. By the time you pass tasks to a combinator, they are already in flight.

```csharp
var t1 = FetchAsync("api1"); // starts HERE
var t2 = FetchAsync("api2"); // starts HERE

await Task.WhenAll(t1, t2);  // just observes — waits for both to finish
```

---

## Task.WhenAll

Waits for **all** tasks to complete. Acts as a barrier — nothing continues until the slowest task finishes.

- Returns `Task<T[]>` when all tasks share the same return type
- Returns `Task` (void) when tasks have different return types

```csharp
var t1 = FetchAsync("api1", 300);
var t2 = FetchAsync("api2", 500);
var t3 = FetchAsync("api3", 400);

string[] results = await Task.WhenAll(t1, t2, t3);
```

### Exception handling

Awaiting `WhenAll` surfaces only the **first** exception. All exceptions are accessible by inspecting individual tasks:

```csharp
try
{
    await Task.WhenAll(t1, t2, t3);
}
catch (Exception ex)
{
    // ex = first exception only
}

foreach (var task in new[] { t1, t2, t3 })
{
    if (task.IsFaulted)
        Console.WriteLine(task.Exception?.InnerException?.Message);
    else if (task.IsCompletedSuccessfully)
        Console.WriteLine(task.Result); // safe — task is already done
}
```

> [!WARNING]
> The catch block is required even if you plan to inspect tasks manually. Without it the faulted WhenAll task propagates up before you reach the loop.

> [!NOTE]
> Only one exception is surfaced by await because throwing multiple exceptions simultaneously is not a concept in C#. The rest are accessible via `task.Exception.InnerExceptions`.

---

## Task.WhenAny

Completes when the **first** task finishes (any state — success, faulted, or cancelled). Returns the winning `Task<T>`, not the result `T`.

```csharp
var winner = await Task.WhenAny(t1, t2, t3);
var result = await winner; // second await required to unwrap value or throw exception
```

### Why Task<T> and not T?

Three reasons:

1. **Identity** — returning the task lets you check which task won (`if (winner == t1)`)
2. **Mixed types** — tasks may return different types, so there is no single `T` to unwrap to
3. **Exception control** — you decide when and whether to observe the exception; `WhenAny` does not decide for you

```csharp
var winner = await Task.WhenAny(t1, t2, t3);

if (winner.IsFaulted)
{
    LogError(winner.Exception);
    // handle gracefully before rethrowing or using fallback
}
else
{
    var result = await winner;
}
```

### Timeout pattern

```csharp
using var cts = new CancellationTokenSource();

var work    = SlowWorkAsync(cts.Token);
var timeout = Task.Delay(TimeSpan.FromSeconds(5)); // no token — it's just a clock

if (await Task.WhenAny(work, timeout) == timeout)
{
    cts.Cancel();
    throw new TimeoutException();
}

var result = await work;
```

> [!WARNING]
> Use separate CancellationTokenSources for the timeout delay and the work task. Sharing one token means cancelling the timeout also cancels the work unintentionally.

### Hedged requests pattern

```csharp
var replicas = new[]
{
    QueryReplicaAsync("replica-1"),
    QueryReplicaAsync("replica-2"),
    QueryReplicaAsync("replica-3"),
};

var winner = (Task<Data>) await Task.WhenAny(replicas);
Data result = await winner;
```

### Cancelling a completed task

Cancelling a task that already completed is a no-op. A task's state is a one-way transition (`Running → RanToCompletion / Faulted / Canceled`). Once in a final state it cannot be changed. The cancellation signal is only observed by code that is still running.

---

## Task.WhenEach

**.NET 9+** — returns `IAsyncEnumerable<Task<T>>`. Yields each task as it completes, in **completion order** (not start order). Use `await foreach` to process results as they arrive.

```csharp
await foreach (var task in Task.WhenEach(tasks))
{
    var result = await task; // await inside loop to get value or surface exception
    progressBar.Update(result);
}
```

### WhenAll vs WhenEach

| | WhenAll | WhenEach |
|---|---|---|
| Completes | After all tasks | Streams as each finishes |
| Results | All at once, in start order | One at a time, in completion order |
| Use case | Barrier — need everything before proceeding | Progress, partial updates, streaming |

---

## Blocking Equivalents

Avoid in async code. Use only in legacy or synchronous-only contexts.

| Async | Blocking | Key difference |
|---|---|---|
| `await Task.WhenAll` | `Task.WaitAll` | WaitAll throws AggregateException directly |
| `await Task.WhenAny` | `Task.WaitAny` | WaitAny returns int index, not the task |

Both blocking variants hold the thread and risk deadlock in contexts with a SynchronizationContext.

---

## Key Mechanics

### Tasks start before combinators see them

```csharp
// WRONG — sequential, not concurrent
await Task.WhenAll(
    await FetchAsync("api1"), // waits for api1 before starting api2
    await FetchAsync("api2")
);

// CORRECT — concurrent
var t1 = FetchAsync("api1");
var t2 = FetchAsync("api2");
await Task.WhenAll(t1, t2);
```

### await vs .Result

| | `await` | `.Result` |
|---|---|---|
| Thread | Released to pool | Blocked |
| Exception | Original exception | Wrapped in AggregateException |
| Deadlock risk | No | Yes, in some contexts |

### Cancellation throws OperationCanceledException

Any .NET method that accepts a CancellationToken will throw `OperationCanceledException` when cancelled — it does not silently skip. This is how cancellation propagates up the call stack.

```csharp
await Task.Delay(3000, ct);        // throws OperationCanceledException when ct cancelled
await httpClient.GetAsync(url, ct); // same
await dbContext.SaveChangesAsync(ct); // same
```

Manual check is only needed in CPU-bound loops with no awaitable accepting a token:

```csharp
for (int i = 0; i < 1_000_000; i++)
{
    DoHeavyWork(i);
    if (i % 1000 == 0)
        ct.ThrowIfCancellationRequested();
}
```

---

## Decision Guide

| Scenario | Use |
|---|---|
| Need all results before continuing | `Task.WhenAll` |
| First response wins (timeout, race, hedging) | `Task.WhenAny` |
| Process results as each finishes (progress) | `Task.WhenEach` (.NET 9+) |
| Synchronous / legacy context only | `Task.WaitAll` / `Task.WaitAny` |

---

## Code Examples

### WhenAll — collect successful results after partial failure

```csharp
try { await Task.WhenAll(t1, t2, t3); }
catch { }

var successful = tasks
    .Where(t => t.IsCompletedSuccessfully)
    .Select(t => t.Result)
    .ToList();
```

### WhenEach — completion order with elapsed time

```csharp
var sw = Stopwatch.StartNew();

await foreach (var task in Task.WhenEach(tasks))
{
    var result = await task;
    Console.WriteLine($"{result} completed at {sw.ElapsedMilliseconds}ms");
}
```

---

## See also

- [[01-task-thread-threadpool]]
- [[02-async-await-internals]]
- [[03-cancellation-token]]
- [[05-synchronization-primitives]]
