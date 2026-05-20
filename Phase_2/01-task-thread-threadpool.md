---
phase: 2
topic: 1
title: Task / Thread / ThreadPool
tags: [dotnet, threading, task, threadpool, thread, async]
created: 2026-05-20
status: done
---

# Task / Thread / ThreadPool

## Table of Contents
- [[#Three abstraction layers]]
- [[#Thread]]
- [[#ThreadPool]]
- [[#Task]]
- [[#Task states]]
- [[#Exception behavior]]
- [[#Fire and forget]]
- [[#Long-running work]]
- [[#Task.Delay vs Thread.Sleep]]
- [[#.Wait() vs await]]
- [[#Quick reference]]
- [[#See also]]

---

## Three abstraction layers

```
Task / Task<T>
    └── ThreadPool  (default scheduler)
            └── OS Threads
```

`Task` sits on top of `ThreadPool`, which manages reusable OS threads. `new Thread()` bypasses the pool entirely.

---

## Thread

A raw OS-level worker. Each thread has its own stack (~1 MB), a kernel object, and a full lifecycle you manage manually.

```csharp
var t = new Thread(() =>
{
    Console.WriteLine(Thread.CurrentThread.ManagedThreadId);
    Thread.Sleep(200);
});
t.Start();
t.Join(); // block caller until thread finishes
```

**Why you almost never use it:**
- Expensive to create (~1 MB stack + kernel allocation)
- No result propagation
- No built-in exception handling
- No cancellation support
- Manual start/join lifecycle

**When you do use it:**
- Need a specific thread priority
- Interop scenarios requiring a dedicated OS thread
- COM STA thread requirements

---

## ThreadPool

A set of pre-created, reusable threads managed by the .NET runtime. Instead of creating/destroying threads per work item, work items are queued and dispatched onto idle threads.

```csharp
ThreadPool.QueueUserWorkItem(_ => DoWork()); // low-level, rarely used directly
```

**Internal structure:**
- **Global queue** (FIFO) — work queued from outside the pool
- **Per-thread local queues** (LIFO) — work queued by a pool thread itself (e.g. nested Task.Run); promotes cache locality
- **Work stealing** — an idle thread steals from another thread's local queue when its own is empty

**Auto-tuning:** the pool uses a *hill-climbing* algorithm — it experiments with thread count and measures throughput, adding or removing threads to find the optimal point. You can influence minimums with `ThreadPool.SetMinThreads()`.

> [!warning] Not designed for long-running work
> A blocked pool thread causes the pool to inject replacement threads, which can cascade into thread explosion.

---

## Task

The abstraction you use in practice. `Task` wraps a unit of work and tracks its outcome.

```csharp
// Fire and track
Task t = Task.Run(() => DoWork());
await t;

// With result
Task<int> t = Task.Run(() => ComputeAnswer());
int result = await t; // result == 42
```

What `Task` adds over raw `ThreadPool`:
- **Result** — `Task<T>` carries a return value
- **Exception storage** — exceptions caught and stored, rethrown on `await`
- **Cancellation** — via `CancellationToken`
- **Composition** — `WhenAll`, `WhenAny`
- **Continuations** — chained `await`s or `.ContinueWith()`

---

## Task states

```
Created
   ↓
WaitingToRun       ← Task.Run() starts here, skips Created
   ↓
Running
   ↓
 ┌─────────────────────────────┐
 ↓             ↓               ↓
RanToCompletion  Faulted     Canceled
```

`Task.Run()` skips `Created` and goes straight to `WaitingToRun`.

---

## Exception behavior

When a task body throws, the exception is **caught by the runtime and stored** inside the `Task` as an `AggregateException`. The task moves to `Faulted`. Nothing blows up until you observe it.

```csharp
// await — unwraps AggregateException, throws inner exception directly
try
{
    await Task.Run(() => throw new InvalidOperationException("boom"));
}
catch (InvalidOperationException ex) { } // caught directly

// .Wait() / .Result — throws AggregateException
try
{
    task.Wait();
}
catch (AggregateException ex)
{
    var inner = ex.InnerExceptions[0]; // InvalidOperationException
}

// Never observed — silently lost in .NET 4.5+
Task.Run(() => throw new Exception()); // exception disappears on GC
```

Subscribe to unobserved exceptions:
```csharp
TaskScheduler.UnobservedTaskException += (sender, e) =>
{
    logger.LogError(e.Exception, "Unobserved task exception");
    e.SetObserved();
};
```

> [!note] .NET 4.0 vs 4.5+
> In .NET 4.0, unobserved exceptions crashed the process on GC finalization. From .NET 4.5 onward they are silently swallowed unless you subscribe to `UnobservedTaskException`.

---

## Fire and forget

When you genuinely need fire-and-forget, catch internally and use the discard syntax:

```csharp
_ = Task.Run(async () =>
{
    try { await DoWorkAsync(); }
    catch (Exception ex) { logger.LogError(ex, "Background task failed"); }
});
```

`_` signals intentional discard to both the reader and static analyzers (suppresses the "unawaited task" warning).

---

## Long-running work

```csharp
// Hint to the scheduler: dedicate a thread, don't use the pool
Task.Factory.StartNew(
    () => LongRunningWork(),
    TaskCreationOptions.LongRunning);
```

For app-lifetime background work in ASP.NET Core use `BackgroundService` (Phase 4).

---

## Task.Delay vs Thread.Sleep

| | `Thread.Sleep` | `await Task.Delay` |
|---|---|---|
| Thread during wait | Blocked, held | Released back to pool |
| Suitable in async | ❌ No | ✅ Yes |
| Mechanism | OS thread suspension | Timer callback |

```csharp
// Wrong in async context — wastes a pool thread
Thread.Sleep(1000);

// Correct — thread is free during the delay
await Task.Delay(1000);
```

---

## .Wait() vs await

| | `.Wait()` / `.Result` | `await` |
|---|---|---|
| Calling thread | Blocked | Released |
| Exception type thrown | `AggregateException` | Inner exception directly |
| Deadlock risk | Yes (UI/ASP.NET classic) | No |
| Use in async code | ❌ Avoid | ✅ Always prefer |

`.Wait()` can deadlock when the continuation needs the same thread that `.Wait()` is blocking (common on UI threads and classic ASP.NET `SynchronizationContext`).

---

## Quick reference

| Need | Use |
|---|---|
| Queue short work onto pool | `Task.Run()` |
| Long-running dedicated thread | `Task.Factory.StartNew(..., LongRunning)` |
| App-lifetime background work | `BackgroundService` (Phase 4) |
| Delay in async method | `await Task.Delay()` |
| Fire and forget safely | `_ = Task.Run(async () => { try{} catch{} })` |
| Raw OS thread (rare) | `new Thread()` |

---

## See also

- [[02-async-await-internals]]
- [[03-cancellation-token]]
- [[04-task-combinators]]
