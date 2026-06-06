---
phase: 2
tags: [csharp, threading, synchronization, semaphoreslim, async]
date: 2026-06-06
---

# SemaphoreSlim #threading #synchronization #async

## Table of Contents
- [[#What is SemaphoreSlim]]
- [[#Constructor]]
- [[#Key Methods]]
- [[#Throttle Pattern]]
- [[#Signaling Pattern]]
- [[#Thread Affinity]]
- [[#Comparison Table]]
- [[#Gotchas]]
- [[#See also]]

---

## What is SemaphoreSlim

A lightweight synchronization primitive that limits the number of threads (or async tasks) that can access a resource concurrently. Unlike `lock` and `Mutex`, it:

- Supports `async/await` via `WaitAsync()`
- Allows more than 1 concurrent caller
- Has **no thread affinity** — any thread can call `Release()`

---

## Constructor

```csharp
new SemaphoreSlim(initialCount)
new SemaphoreSlim(initialCount, maxCount)
```

| Parameter | Meaning |
|---|---|
| `initialCount` | Available permits at creation time |
| `maxCount` | Upper bound — `Release()` throws `SemaphoreFullException` if exceeded |

```csharp
new SemaphoreSlim(3)      // same as (3, 3) — 3 permits, max 3
new SemaphoreSlim(0, 1)   // starts locked — signaling pattern
```

> [!NOTE]
> `new SemaphoreSlim(n)` is shorthand for `new SemaphoreSlim(n, n)`

---

## Key Methods

| Method | Description |
|---|---|
| `Wait()` | Blocks the thread until a permit is available |
| `WaitAsync()` | Suspends the async method (thread stays free) |
| `Release()` | Increments the counter by 1 (returns a permit) |
| `Release(n)` | Increments the counter by n |
| `CurrentCount` | How many permits are currently available |
| `Dispose()` | Frees internal `WaitHandle` if used |

> [!WARNING]
> Always call `Release()` in a `finally` block — if the protected code throws, the permit must still be returned

---

## Throttle Pattern

Limit concurrent access to a resource (e.g. external API, DB connection pool).

```csharp
private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(5, 5);

public async Task CallApiAsync()
{
    await _semaphore.WaitAsync();
    try
    {
        // at most 5 tasks here simultaneously
        await httpClient.GetAsync("...");
    }
    finally
    {
        _semaphore.Release();
    }
}
```

Flow: `WaitAsync()` → do work → `Release()` (same logical owner)

---

## Signaling Pattern

One task waits, another signals when ready. Semaphore starts at `initialCount = 0`.

```csharp
var semaphore = new SemaphoreSlim(0, 1);

// Consumer — starts blocked
var consumer = Task.Run(async () =>
{
    await semaphore.WaitAsync();
    Console.WriteLine("got the signal");
});

// Producer — signals when ready
var producer = Task.Run(async () =>
{
    await Task.Delay(1000);
    semaphore.Release(); // no prior Wait() needed
});

await Task.WhenAll(consumer, producer);
```

> [!NOTE]
> `Release()` does NOT need to be preceded by `Wait()`. It simply increments the counter.
> In the signaling pattern the consumer does not need to call `Release()` after finishing.

---

## Thread Affinity

**Thread affinity** means the same thread that acquired a primitive must be the one to release it.

| Primitive | Thread affinity | Works with async/await |
|---|---|---|
| `lock` / `Monitor` | ✅ Yes | ❌ No |
| `Mutex` | ✅ Yes | ❌ No |
| `SemaphoreSlim` | ❌ No | ✅ Yes |

`SemaphoreSlim` has no ownership tracking — it is just a counter. Any thread can call `Release()`.

---

## Comparison Table

| | `lock` | `Monitor` | `Mutex` | `SemaphoreSlim` |
|---|---|---|---|---|
| Max concurrent | 1 | 1 | 1 | N |
| async support | ❌ | ❌ | ❌ | ✅ |
| Cross-process | ❌ | ❌ | ✅ | ❌ |
| Thread affinity | ✅ | ✅ | ✅ | ❌ |
| Release before Wait | throws | throws | throws | ✅ allowed |

---

## Gotchas

- **Missing `try/finally`** — if work throws, `Release()` is never called, permits leak
- **`SemaphoreFullException`** — calling `Release()` when `CurrentCount == maxCount`
- **Reusing loop variable in lambdas** — capture to local variable before `Task.Run`
- **Dispose** — only critical when `initialCount = 0` (creates internal `WaitHandle`); safe to skip otherwise but good practice

---

## See also

- [[01-lock-monitor]]
- [[02-mutex]]
- [[06-deadlocks-race-conditions]]
