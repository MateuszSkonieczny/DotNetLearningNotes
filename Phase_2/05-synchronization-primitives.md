---
phase: 2
topic: 5
title: Synchronization Primitives
tags: [csharp, threading, synchronization, lock, monitor, mutex]
date: 2026-06-03
status: in-progress
---

# Synchronization Primitives #threading #synchronization

## Table of Contents
- [[#Overview]]
- [[#lock]]
- [[#Monitor]]
- [[#Mutex]]
- [[#SafeQueue Example]]
- [[#Common Pitfalls]]
- [[#Interview Questions]]

---

## Overview

Synchronization primitives are used to coordinate access to shared resources across multiple threads. Each primitive has different trade-offs between simplicity, flexibility, and async compatibility.

| Primitive | Use case | Async-friendly |
|---|---|---|
| `lock` / `Monitor` | Mutual exclusion, Wait/Pulse signaling | ❌ |
| `SemaphoreSlim` | Bounded concurrency, async gates | ✅ |
| `Interlocked` | Lock-free atomic operations | ✅ |
| `ManualResetEventSlim` | One-time or resettable signal | ✅ |
| `ReaderWriterLockSlim` | Multiple readers, single writer | ❌ |
| `Mutex` | Cross-process locking | ❌ |
| `SpinLock` / `SpinWait` | Very short critical sections | ❌ |

---

## lock

### What it compiles to

`lock` is syntactic sugar. This:

```csharp
lock (_lock)
{
    // work
}
```

compiles to:

```csharp
bool lockTaken = false;
try
{
    Monitor.Enter(_lock, ref lockTaken);
    // work
}
finally
{
    if (lockTaken) Monitor.Exit(_lock);
}
```

> [!NOTE]
> The `lockTaken` guard exists because a `ThreadAbortException` can fire in the tiny window between `Monitor.Enter` acquiring the lock and execution reaching the `try` block. Without `lockTaken`, `finally` wouldn't know whether to call `Exit`, potentially leaking the lock forever.

### What NOT to lock on

| Target | Why it's wrong |
|---|---|
| `this` | The object reference is visible externally — other code can lock on the same instance, causing unexpected deadlocks |
| `string` | String literals are interned — identical strings across the AppDomain share the same reference, so unrelated code can deadlock you |
| `typeof(MyClass)` | `Type` objects are shared across the AppDomain, same problem as string interning |

Always lock on a **private, dedicated `object` field**:

```csharp
private readonly object _lock = new object();
```

### Cannot use await inside lock

```csharp
lock (_lock)
{
    await SomethingAsync(); // compiler error
}
```

`await` may resume on a different thread, but `Monitor` requires the **same thread** to release the lock it acquired. These are incompatible.

Use `SemaphoreSlim(1, 1)` with `WaitAsync()` instead for async mutual exclusion.

---

## Monitor

`Monitor` is what `lock` compiles to, but exposes additional capabilities: `Wait`, `Pulse`, `PulseAll`, and `TryEnter`.

### Monitor.Enter vs lock

`Monitor.Enter` behaves identically to `lock` but is manual — you must write the `try/finally` yourself. The risk is forgetting `Exit` on exception paths. Prefer `lock` unless you need `TryEnter`.

### Monitor.TryEnter

```csharp
bool lockTaken = false;
Monitor.TryEnter(_lock, TimeSpan.FromSeconds(1), ref lockTaken);
if (lockTaken)
{
    try { /* work */ }
    finally { Monitor.Exit(_lock); }
}
else
{
    // lock not acquired — do something else
}
```

> [!NOTE]
> `ref bool lockTaken` and `TryEnter` with timeout solve different problems:
> - `lockTaken` — safety against async exceptions between `Enter` and `try`
> - `TryEnter(timeout)` — non-blocking, do other work if lock isn't free in time

### Monitor.Wait and Monitor.Pulse

Used for **thread signaling** — a thread waits for a condition to become true, another thread signals when it does.

**Two internal queues:**

| Queue | Contains |
|---|---|
| Wait queue | Threads that called `Monitor.Wait` — suspended, waiting for `Pulse` |
| Ready queue | Threads that were pulsed and are now competing for the lock |

**`Monitor.Wait(_lock)`** — must be called while holding the lock:
1. Atomically releases the lock
2. Suspends the thread (moved to wait queue)
3. When pulsed, moves to ready queue
4. Re-acquires the lock before returning

**`Monitor.Pulse(_lock)`** — wakes one thread from the wait queue → ready queue.

**`Monitor.PulseAll(_lock)`** — wakes all threads from the wait queue.

> [!WARNING]
> `Wait` and `Pulse`/`PulseAll` must be called from inside a `lock` block. Calling them without holding the monitor throws `SynchronizationLockException`.

### Always use while, never if

```csharp
// WRONG
if (_queue.Count == 0)
    Monitor.Wait(_lock);

// CORRECT
while (_queue.Count == 0)
    Monitor.Wait(_lock);
```

With multiple waiters, `PulseAll` wakes all threads but only one can acquire the lock. By the time the second thread acquires the lock, the first may have already consumed the item. The `while` re-checks the condition, the `if` does not.

This is called a **spurious wakeup guard**.

### Pulse vs PulseAll

| | `Pulse` | `PulseAll` |
|---|---|---|
| Wakes | 1 thread | All waiting threads |
| Risk | Wrong thread woken, no progress | Thundering herd, wasted wakeups |
| Safe with | Single producer + single consumer | Multiple producers and/or consumers |

> [!NOTE]
> `Monitor` has a **single wait queue** — it cannot selectively wake producers or consumers. This is a key limitation. `SemaphoreSlim` solves this with separate signals.

### What happens to the thread during Wait

| Thread type | Behavior |
|---|---|
| `new Thread(...)` | Suspended by OS — alive but parked, no CPU used |
| ThreadPool thread | Suspended — pool may spin up replacement thread. **Avoid blocking ThreadPool threads.** |
| `async/await` | Use `SemaphoreSlim.WaitAsync()` instead — releases thread back to pool |

---

## Mutex

`Mutex` is like `lock`/`Monitor` but operates at the **OS level**, enabling synchronization across multiple processes or app instances.

### lock vs Mutex

| | `lock` / `Monitor` | `Mutex` |
|---|---|---|
| Scope | Single process | Cross-process |
| Lives in | CLR (user-space) | OS kernel |
| Speed | Fast | Slow (kernel transition) |
| Named | ❌ | ✅ |
| Async-friendly | ❌ | ❌ |

### Creating vs Acquiring

These are two separate steps — easy to confuse:

```csharp
// Creating — gets a handle to the OS object (does NOT acquire it)
var mutex = new Mutex(false, "MyAppMutex");

// Acquiring — blocks until the mutex is available
mutex.WaitOne();

// Releasing
mutex.ReleaseMutex();
```

> [!NOTE]
> The first `bool` parameter of `new Mutex(initiallyOwned, name)` controls whether the **creating thread immediately owns** the mutex. Pass `false` unless you explicitly want to own it on creation.

### Named Mutex — single instance app pattern

A named `Mutex` is a system-wide OS object identified by its name. Any process that uses the same name refers to the **same OS object**.

```csharp
var mutex = new Mutex(false, "MyAppSingleInstance");
bool acquired = false;

try
{
    try
    {
        acquired = mutex.WaitOne(0); // non-blocking — returns immediately
    }
    catch (AbandonedMutexException)
    {
        acquired = true; // we still own it — previous instance crashed
        Console.WriteLine("Warning: previous instance crashed, state may be corrupted.");
    }

    if (!acquired)
    {
        Console.WriteLine("Another instance is already running. Exiting.");
        return;
    }

    Console.WriteLine("Application started. Running... (press Enter to exit)");
    Console.ReadLine();
}
finally
{
    if (acquired)
        mutex.ReleaseMutex();

    mutex.Dispose();
}
```

> [!NOTE]
> Use `WaitOne(0)` for a **non-blocking attempt** — returns `true` if acquired, `false` if already taken. Without the timeout overload, `WaitOne()` blocks indefinitely.

### AbandonedMutexException

If a thread owns a `Mutex` and **terminates without releasing it**, the Mutex becomes *abandoned*.

- The next thread calling `WaitOne` **does acquire** the Mutex
- But receives `AbandonedMutexException` as a warning
- Signals that shared state protected by this Mutex may be **corrupted**
- You can catch it, log a warning, and decide whether to proceed

> [!WARNING]
> Do NOT use `using` on `Mutex` for release — `Dispose()` and `ReleaseMutex()` are separate concerns. Always release with `ReleaseMutex()` in a `finally` block, then `Dispose()`.

### Why Mutex is slow

`lock`/`Monitor` operate entirely in **user-space** (CLR manages them). `Mutex` is a kernel object — every acquire/release requires a **user-mode → kernel-mode transition**, which involves a context switch managed by the OS. This makes it orders of magnitude slower than `lock`.

Use `Mutex` only when cross-process synchronization is actually needed. For single-process use, prefer `lock`.

---

## SafeQueue Example

Bounded producer/consumer queue using `Monitor.Wait` and `PulseAll`:

```csharp
class SafeQueue<T>
{
    private readonly Queue<T> _queue = new Queue<T>();
    private readonly object _lock = new object();
    private readonly int _maxCapacity = 5;

    public void Enqueue(T item)
    {
        lock (_lock)
        {
            while (_queue.Count >= _maxCapacity)
                Monitor.Wait(_lock);

            _queue.Enqueue(item);
            Monitor.PulseAll(_lock); // wake all — consumers and producers re-check their condition
        }
    }

    public T Dequeue()
    {
        lock (_lock)
        {
            while (_queue.Count == 0)
                Monitor.Wait(_lock);

            var item = _queue.Dequeue();
            Monitor.PulseAll(_lock);
            return item;
        }
    }
}
```

> [!NOTE]
> Use `PulseAll` when multiple producers or consumers are possible. With a single producer + single consumer, `Pulse` is sufficient and avoids the thundering herd.

> [!WARNING]
> Setting `_maxCapacity = 0` causes an instant deadlock — producer immediately waits (queue always full), consumer immediately waits (queue always empty), nobody ever calls Pulse.

---

## Common Pitfalls

| Pitfall | Description |
|---|---|
| `if` instead of `while` on `Wait` | Condition may no longer be true when thread wakes — always use `while` |
| `await` inside `lock` | Compiler error — use `SemaphoreSlim` instead |
| Locking on `this` / `string` / `Type` | Shared references cause unexpected deadlocks |
| Forgetting `Monitor.Exit` | Use `lock` keyword or always wrap in `try/finally` |
| Unbounded `Wait` with no timeout | Thread hangs forever if `Pulse` is never called |
| `_maxCapacity = 0` | Instant deadlock — both producer and consumer wait immediately |
| Single `Pulse` with multiple consumers | Wrong thread may be woken, correct thread stays sleeping |

### Deadlock example

```csharp
// Thread A: locks _lock1 then tries _lock2
public void MethodA()
{
    lock (_lock1) { lock (_lock2) { } }
}

// Thread B: locks _lock2 then tries _lock1
public void MethodB()
{
    lock (_lock2) { lock (_lock1) { } }
}
```

Thread A holds `_lock1`, waiting for `_lock2`. Thread B holds `_lock2`, waiting for `_lock1`. Neither can proceed.

**Mitigation:** always acquire multiple locks in a consistent global order across all threads.

### Lock convoy

Lock convoy occurs when multiple threads contend for the same lock in a tight loop. Waiting threads are repeatedly woken up by the OS, fail to acquire the lock (already taken by an already-running thread), and go back to sleep — wasting context switches without doing useful work.

**Mitigations:** `ConcurrentQueue<T>` / `ConcurrentDictionary<T>`, `ReaderWriterLockSlim` for read-heavy workloads, reducing lock scope.

---

## Interview Questions

1. What does `lock` compile to?
2. Why can't you use `await` inside a `lock` block?
3. Why must you never lock on `this`, a `string`, or a `Type`?
4. What happens to the lock if an exception is thrown inside a `lock` block?
5. What is the `ref bool lockTaken` pattern and why does it exist?
6. Walk through exactly what `Monitor.Wait` does to the thread and the lock.
7. `Monitor.Pulse` vs `Monitor.PulseAll` — when to use each?
8. Why must `Monitor.Wait` always be inside a `while` loop?
9. Describe a minimal two-thread deadlock using `lock`. How do you prevent it?
10. What is lock convoy and how do you mitigate it?
11. What is the difference between `Mutex` and `lock`? When would you choose `Mutex`?
12. What does `AbandonedMutexException` mean, and does the throwing thread own the Mutex?
13. Why is `Mutex` slower than `lock`?
14. What is a named `Mutex` and where does it live?

---

See also: [[02-async-await-internals]] | [[06-deadlocks-race-conditions]]
